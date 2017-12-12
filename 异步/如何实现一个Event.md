# 如何实现一个Event


---
## **前言**
我们已经通过阅读`Node.js`中`Event`的实现方式对其有一定的了解,那么如何实现一个可以Node与Browser环境通用的`Event`呢?

> **提前声明:** 我们没有对传入的参数进行及时判断而规避错误,仅仅对核心方法进行了实现,如果要实现一个健壮的程序,你是需要这样做的.

---
#### 1.基本构造

**1.1初始化class**
我们利用ES6的`class`关键字对`Event`进行初始化,包括`Event`的事件清单和监听者上限.

我们选择了`Map`作为储存事件的结构,因为作为键值对的粗存方式`Map`比一般对象更加适合,我们操作起来也更加简洁,可以先看一下Map的[基本用法与特点](http://es6.ruanyifeng.com/#docs/set-map#Map).

```javascript
class EventEmeitter {
  constructor() {
    this._events = this._events || new Map();
    this._maxListeners = this._maxListeners || 10;
  }
}
```

**1.2 监听与触发**

触发监听函数我们可以用`apply`与`call`两种方法,在少数参数时`call`的性能更好,多个参数时`apply`性能更好,Node的Event就在三个参数一下用`call`否则用`apply`,但是我们不想写的那么复杂,就做了一个简化版.

```javascript
EventEmeitter.prototype.emit = function(type, ...args) {
  let handler;
  handler = this._events.get(type);
  if (args.length > 0) {
    handler.apply(this, args);
  } else {
    handler.call(this);
  }
  return true;
};

EventEmeitter.prototype.addListener = function(type, fn) {
  if (!this._events.get(type)) {
    this._events.set(type, fn);
  }
};

```

我们实现了触发事件的`emit`方法和监听事件的`addListener`方法,至此我们就可以进行简单的实践了.


```javascript
const emitter = new EventEmeitter();

emitter.addListener('arson', man => {
  console.log(`expel ${man}`);
});

emitter.emit('arson', 'low-end'); // expel low-end
```

似乎不错,我们实现了基本的触发/监听,但是如果有多个监听者呢?
```javascript
emitter.addListener('arson', man => {
  console.log(`expel ${man}`);
});
emitter.addListener('arson', man => {
  console.log(`save ${man}`);
});

emitter.emit('arson', 'low-end'); // expel low-end
```
是的,只会触发第一个,因此我们需要进行改造.


---
#### 2.升级改造

**2.1 监听/触发器升级**

我们的`addListener`实现方法还不够健全,在绑定第一个监听者之后,我们就无法对后续监听者进行绑定了,因此我们需要将后续监听者与第一个监听者函数放到一个数组里.

```javascript
EventEmeitter.prototype.emit = function(type, ...args) {
  let handler;
  handler = this._events.get(type);
  if (Array.isArray(handler)) {
    // 如果是一个数组说明有多个监听者,需要依次此触发里面的函数
    for (let i = 0; i < handler.length; i++) {
      if (args.length > 0) {
        handler[i].apply(this, args);
      } else {
        handler[i].call(this);
      }
    }
  } else { // 单个函数的情况我们直接触发即可
    if (args.length > 0) {
      handler.apply(this, args);
    } else {
      handler.call(this);
    }
  }

  return true;
};

EventEmeitter.prototype.addListener = function(type, fn) {
  const handler = this._events.get(type); // 获取对应事件名称的函数清单
  if (!handler) {
    this._events.set(type, fn);
  } else if (handler && typeof handler === 'function') {
    // 如果handler是函数说明只有一个监听者
    this._events.set(type, [handler, fn]); // 多个监听者我们需要用数组储存
  } else {
    handler.push(fn); // 已经有多个监听者,那么直接往数组里push函数即可
  }
};
```
是的,从此以后可以愉快的触发多个监听者的函数了.
```javascript
emitter.addListener('arson', man => {
  console.log(`expel ${man}`);
});
emitter.addListener('arson', man => {
  console.log(`save ${man}`);
});

emitter.addListener('arson', man => {
  console.log(`kill ${man}`);
});

emitter.emit('arson', 'low-end');
//expel low-end
//save low-end
//kill low-end
```

**2.2 移除监听**

我们会用`removeListener`函数移除监听函数,但是匿名函数是无法移除的.


```javascript
EventEmeitter.prototype.removeListener = function(type, fn) {
  const handler = this._events.get(type); // 获取对应事件名称的函数清单

  if (handler && typeof handler === 'function') {
    this._events.delete(type, fn);
  } else {
    let postion;
    // 如果handler是数组
    for (let i = 0; i < handler.length; i++) {
      if (handler[i] === fn) {
        postion = i;
      } else {
        postion = -1;
      }
    }
    // 如果找到匹配的函数,从数组中清除
    if (postion !== -1) {
      handler.splice(postion, 1);
      // 如果清除后只有一个函数,那么取消数组
      if (handler.length === 1) {
        this._events.set(type, handler[0]);
      }
    } else {
      return this;
    }
  }
};
```

---
#### 3.发现问题


我们已经基本完成了`Event`最重要的几个方法,也完成了升级改造,可以说一个`Event`的骨架是被我们开发出来了,但是它仍然有不足和需要补充的地方.

> 1. 鲁棒性不足: 我们没有对参数进行充分的判断,没有完善的报错机制.
> 2. 模拟不够充分: 除了`removeAllListeners`这些方法没有实现以外,例如监听时间后会触发`newListener`事件,我们也没有实现,另外最开始的监听者上限我们也没有利用到.

索性[Event](https://github.com/Gozala/events/blob/master/events.js)库帮我们实现了完整的特性,整个代码量有300多行,很适合阅读,你可以花十分钟的时间通读一下,见识一下完整的Event实现
