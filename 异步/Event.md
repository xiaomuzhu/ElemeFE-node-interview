# Node中Events详解

 
---
## **前言**
`Events` 是 `Node.js` 中一个非常重要的 `core` 模块, 在 `node` 中有许多重要的 `core API` 都是依赖其建立的. 比如 `Stream` 是基于 `Events` 实现的, 而 `fs, net, http` 等模块都依赖 `Stream`, 所以 `Events` 模块的重要性可见一斑.

---
#### 1.基础用法

**1.1继承**
虽然你可以通过传统的原型链的方式进行`Events`的继承,但是Node中提供了更标准的方法`util.inherits`用于事件的继承.

```javascript
var util = require('util');
var EventEmitter = require('events').EventEmitter;

function Music() {
    EventEmitter.call(this);
}

util.inherits(Radio, EventEmitter);

```

**1.2 监听与触发**

通常我们用`on`方法来监听事件,并用`emit`方法触发事件.

```javascript
const EventEmitter = require('events').EventEmitter;
const myEmitter = new EventEmitter();

const connection = (id) => {
  console.log('client id: ' + id); //client id: 6
};

myEmitter.on('connection', connection); //监听名为`connection`的事件,并执行connection方法
myEmitter.emit('connection', 6); //触发名为`connection`的事件

```

类似的监听方法还有`once`,其作用与`on`类似,只是只触发一次回调.

```javascript
const EventEmitter = require('events').EventEmitter;
const myEmitter = new EventEmitter();

const connection = (id) => {
  console.log('client id: ' + id); //client id: 6
};

myEmitter.once('connection', connection); //监听名为`connection`的事件,并执行connection方法
myEmitter.emit('connection', 6); //触发名为`connection`的事件
myEmitter.emit('connection', 7); //触发事件,但不执行回调函数

```

**1.3 移除监听器**

监听器既然可以被创建自然可以被移除`removeListener`方法便是用来移除监听器的回调函数,它接受两个参数，第一个是事件名称，第二个是回调函数名称.
```javascript
const EventEmitter = require('events').EventEmitter;
const myEmitter = new EventEmitter();

const connection = (id) => {
  console.log('client id: ' + id);
};

myEmitter.once('connection', connection); //监听名为`connection`的事件,并执行connection方法
myEmitter.removeListener('connection', connection); //移除监听器,次后触发的事件全部无效
myEmitter.emit('connection', 7); //无效
myEmitter.emit('connection', 6); //无效
```
如果某事件有多个回调函数,你想一次移除,可以调用`removeAllListeners`方法.

**1.4 监听新监听器**
`EventEmitter` 实例会在一个监听器被添加到其内部监听器数组之前触发自身的 `'newListener'` 事件。

我们可以通过监听这个`'newListener'`事件来追踪新的监听器.

```javascript
const myEmitter = new MyEmitter();
// 只处理一次，所以不会无限循环
myEmitter.once('newListener', (event, listener) => {
  if (event === 'event') {
    // 在开头插入一个新的监听器
    myEmitter.on('event', () => {
      console.log('B');
    });
  }
});
myEmitter.on('event', () => {
  console.log('A');
});
myEmitter.emit('event');
//   B
//   A
```

---
#### 2.异常管理

事件中的异常处理是我们不得不讨论的一个话题,通常我们会采用两种方式来管理我们的异常,一种是通过监听`error`事件的方式,另一种是通过`domain`集中管理,由于`domain`模块即将被废弃我们只讨论第一种方法.


```javascript
const myEmitter = new MyEmitter();
myEmitter.on('error', (err) => {
  console.log('有错误');
});
myEmitter.emit('error', new Error('whoops!'));
//  有错误
```

---
#### 3.Events源码分析

**3.1 默认的监听器数量限制**
每个事件默认可以注册最多 `10` 个监听器。 单个 `EventEmitter` 实例的限制可以使用 `emitter.setMaxListeners(n)` 方法改变。 所有 `EventEmitter` 实例的默认值可以使用 `EventEmitter.defaultMaxListeners` 属性改变。

那这个方法是如何实现的呢?

```javascript
...
var defaultMaxListeners = 10;

Object.defineProperty(EventEmitter, 'defaultMaxListeners', { //ES5的defineProperty方法
  enumerable: true, //属性可枚举
  get: function() {
    return defaultMaxListeners;
  },
  set: function(arg) { //传入设置的最大监听器数量
    // force global console to be compiled.
    // see https://github.com/nodejs/node/issues/4467
    console;
    // check whether the input is a positive number (whose value is zero or
    // greater and not a NaN).
    if (typeof arg !== 'number' || arg < 0 || arg !== arg) //判断参数是否符合要求
      throw new TypeError('"defaultMaxListeners" must be a positive number');
    defaultMaxListeners = arg;
  } 
});
...
```
这是一个很简单的`Object.defineProperty`方法应用案例,对于大多数人理解起来没有太大难度.

值得注意的是,一旦修改这个数量,便会影响全部的事件,对于特定的事件我们建议用`emitter.setMaxListeners(n)`修改.

```javascript
EventEmitter.prototype.setMaxListeners = function setMaxListeners(n) {
  if (typeof n !== 'number' || n < 0 || isNaN(n))
    throw new TypeError('"n" argument must be a positive number');
  this._maxListeners = n;
  return this;
};
```

**3.2 事件监听**
事实上事件监听除了常用`on`方法以外,还有一个同样效果的方法`emitter.addListener(eventName, listener)`,我们可以通过源码窥探一下这两个方法是如何实现的.

```javascript
function _addListener(target, type, listener, prepend) { //Events的私有方法,供`addListener` `on`使用
  var m;
  var events;
  var existing;

  if (typeof listener !== 'function') //如果listener不是函数那抛出错误,确保listener可以被执行
    throw new TypeError('"listener" argument must be a function');

  events = target._events;
  if (!events) { //保证_events对象已经被创建
    events = target._events = Object.create(null);
    target._eventsCount = 0;
  } else {
    // To avoid recursion in the case that type === "newListener"! Before
    // adding it to the listeners, first emit "newListener".
    if (events.newListener) { //防止递归调用的，这里只有定义了newListener后，才会发送事件newListener
      target.emit('newListener', type,
                  listener.listener ? listener.listener : listener);

      // Re-assign `events` because a newListener handler could have caused the
      // this._events to be assigned to a new object
      events = target._events;
    }
    existing = events[type];
  }


  if (!existing) { //判断事件是否存在
    // Optimize the case of one listener. Don't need the extra array object.
    existing = events[type] = listener; //如果不存在，直接将listener写入
    ++target._eventsCount;
  } else { 
    if (typeof existing === 'function') { //如果是函数那么要转化成为一个数组写入
      // Adding the second element, need to change to array.
      existing = events[type] = prepend ? [listener, existing] :
                                          [existing, listener];
    } else {
      // If we've already got an array, just append.
      //如果是数组,直接添加到数组
      if (prepend) {
        existing.unshift(listener);
      } else {
        existing.push(listener);
      }
    }

    // Check for listener leak
    if (!existing.warned) { //判断监听数量是否超过最大限制,如果超过报错.
      m = $getMaxListeners(target);
      if (m && m > 0 && existing.length > m) {
        existing.warned = true;
        const w = new Error('Possible EventEmitter memory leak detected. ' +
                            `${existing.length} ${String(type)} listeners ` +
                            'added. Use emitter.setMaxListeners() to ' +
                            'increase limit');
        w.name = 'MaxListenersExceededWarning';
        w.emitter = target;
        w.type = type;
        w.count = existing.length;
        process.emitWarning(w);
      }
    }
  }

  return target;
}

EventEmitter.prototype.addListener = function addListener(type, listener) {
  return _addListener(this, type, listener, false); //_addListener私有方法供addListener使用
};

EventEmitter.prototype.on = EventEmitter.prototype.addListener; //这里可以看出addListener方法与on方法等价
```

**3.3 回调移除方法**


```javascript
EventEmitter.prototype.removeListener = //移除具体时间的回调函数
    function removeListener(type, listener) {
      var list, events, position, i, originalListener;

      if (typeof listener !== 'function')  //检查是否为函数否则报错
        throw new TypeError('"listener" argument must be a function');
    // 确保_events存在
      events = this._events; 
      if (!events)
        return this;

      list = events[type];
      if (!list)
        return this;

      if (list === listener || list.listener === listener) {//如果指定的函数与当前回调相同那么发送removeListener事件删除当前回调函数
        if (--this._eventsCount === 0)
          this._events = Object.create(null);
        else {
          delete events[type];
          if (events.removeListener)
            this.emit('removeListener', type, list.listener || listener);
        }
      } else if (typeof list !== 'function') { //如果不是函数的情况下,此时应该是个回调函数组成的数组
        position = -1; //设置函数所在的位置为-1.表示先设置不存在于数组中

        for (i = list.length; i-- > 0;) { //在数组列表中查询此回调
          if (list[i] === listener || list[i].listener === listener) {
            originalListener = list[i].listener;
            position = i;
            break;
          }
        }

        if (position < 0) //小于0代表查不到,直接返回
          return this;

        if (list.length === 1) { //=1说明已经查到,可以操作删除
          list[0] = undefined;
          if (--this._eventsCount === 0) {
            this._events = Object.create(null);
            return this;
          } else { 
            delete events[type]; 
          }
        } else if (position === 0) {
          list.shift();
        } else {
          spliceOne(list, position);
        }

        if (events.removeListener)
          this.emit('removeListener', type, originalListener || listener);
      }

      return this;
    };

EventEmitter.prototype.removeAllListeners =
    function removeAllListeners(type) { //删除与单一事件相关的所有回调函数
      var listeners, events;

      events = this._events; //判断events是否为空
      if (!events)
        return this;

      // not listening for removeListener, no need to emit
      if (!events.removeListener) { // 查看当前是否有removeListener的监听者
        if (arguments.length === 0) {
          this._events = Object.create(null);
          this._eventsCount = 0;
        } else if (events[type]) {
          if (--this._eventsCount === 0)
            this._events = Object.create(null);
          else
            delete events[type];
        }
        return this;
      }

      // emit removeListener for all listeners on all events
      //如果不指定事件,直接清空所有事件的所有回调函数
      if (arguments.length === 0) {
        var keys = Object.keys(events);
        for (var i = 0, key; i < keys.length; ++i) {
          key = keys[i];
          if (key === 'removeListener') continue; //但是唯独不删除removeListener的回调,属于特殊情况
          this.removeAllListeners(key);
        }
        this.removeAllListeners('removeListener');
        this._events = Object.create(null);
        this._eventsCount = 0;
        return this;
      }

      listeners = events[type];
    //如果指定事件,那么删除当前事件下的所有回调
      if (typeof listeners === 'function') {
        this.removeListener(type, listeners);
      } else if (listeners) {
        // LIFO order
        do {
          this.removeListener(type, listeners[listeners.length - 1]);
        } while (listeners[0]);
      }

      return this;
    };
```

**3.4 事件触发**


```javascript
EventEmitter.prototype.emit = function emit(type) { //触发事件的方法
  var er, handler, len, args, i, events, domain;
  var needDomainExit = false;
  var doError = (type === 'error');

  events = this._events;
  if (events)
    doError = (doError && events.error == null);
  else if (!doError)
    return false;

  domain = this.domain;

  // If there is no 'error' event listener then throw.
  // 对error事件做特殊处理
  if (doError) {
    if (arguments.length > 1)
      er = arguments[1];
    if (domain) {
      if (!er)
        er = new Error('Unhandled "error" event');
      if (typeof er === 'object' && er !== null) {
        er.domainEmitter = this;
        er.domain = domain;
        er.domainThrown = false;
      }
      domain.emit('error', er);
    } else if (er instanceof Error) {
      throw er; // Unhandled 'error' event
    } else {
      // At least give some kind of context to the user
      const err = new Error('Unhandled "error" event. (' + er + ')');
      err.context = er;
      throw err;
    }
    return false;
  }
    //指定触发的事件
  handler = events[type];
    //如果没有制定触发事件直接返回
  if (!handler)
    return false;

  if (domain && this !== process) { //如果domain在使用,那么进入domain处理
    domain.enter();
    needDomainExit = true;
  }

  var isFn = typeof handler === 'function';
  len = arguments.length;
  //Events对于触发事件方法的不定参数进行了性能优化,对三个以下的参数这种情况作了分别作了单个处理,而三个以上则没有进行单个特殊处理,导致参数在3个或者以下的情况下会比3个以上的速度快出了一倍以上
  switch (len) {
    // fast cases
    
    case 1:
      emitNone(handler, isFn, this);
      break;
    case 2:
      emitOne(handler, isFn, this, arguments[1]);
      break;
    case 3:
      emitTwo(handler, isFn, this, arguments[1], arguments[2]);
      break;
    case 4:
      emitThree(handler, isFn, this, arguments[1], arguments[2], arguments[3]);
      break;
    // slower
    default:
      args = new Array(len - 1);
      for (i = 1; i < len; i++)
        args[i - 1] = arguments[i];
      emitMany(handler, isFn, this, args);
  }

  if (needDomainExit)
    domain.exit();

  return true;
};
```

**3.5 防止死循环**

如果我们在同一个时间中继续监听此事件是否会触发死循环?

```
const EventEmitter = require('events')

let myEventEmitter = new EventEmitter()

myEventEmitter.on('wtf', function wtf () {
  myEventEmitter.on('wtf', wtf)
})

myEventEmitter.emit('wtf')
```
我们不妨看一下一下源码:

```javascript
...
function emitMany(handler, isFn, self, args) {
  if (isFn)
    handler.apply(self, args);
  else {
    var len = handler.length;
    var listeners = arrayClone(handler, len);
    for (var i = 0; i < len; ++i)
      listeners[i].apply(self, args);
  }
}

...
function arrayClone(arr, n) {
  var copy = new Array(n);
  for (var i = 0; i < n; ++i)
    copy[i] = arr[i];
  return copy;
}

...
```

 `handler` 是具体事件监听数组，此时源码使用 `arrayClone` 方法，拷贝出一个副本，我们在执行过程中其实是在执行这个副本,因此再给原监听数组添加新函数的时候,副本是不会增加的,因此避免了死循环.


---
#### 4.一道面试题

```javascript
const EventEmitter = require('events');

let emitter = new EventEmitter();

emitter.on('myEvent', () => {
  console.log('hi');
  emitter.emit('myEvent');
});

emitter.emit('myEvent');
```
以上代码是否会造成死循环?

答案是会的,因为在监听器中再次**触发**同一个事件会造成死循环,而且`Events`内部只是用克隆副本的方法避免了**监听**同一事件的死循环,无法避免**触发**事件的循环,因此在使用中要避免这种情况.

---




