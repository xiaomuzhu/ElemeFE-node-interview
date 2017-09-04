# Node中Stream(流)详解
 
---

## 前言

流（stream）在 Node.js 中是处理流数据的抽象接口（abstract interface）,其中Node.js 中的 `HTTP `请求 和 `process.stdout` 就都是流的实例.

可以这么说,`Stream`是Node开发过程中无论如何都无法绕开的知识点,因为基于它的场景很多,我们应该尽可能理解`Stream`并掌握它的一些高级用法.

    Stream 接口分成三类:
        可读数据流接口，用于对外提供数据。
        可写数据流接口，用于写入数据。
        双向数据流接口，用于读取和写入数据。

由于`Stream`基本方法较多,我们在本文中就不做过多介绍,相关的基本用法可以直接阅读[官方文档](http://nodejs.cn/api/stream.html#stream_two_modes).

---

#### 1. Stream 中Readable的运作流程

Readable有两种模式,分别是`flowing mode`和`paused mode`,这两种模式的不同之处在于:是否需要手动`Readable.prototype.read(n)`，读取缓冲区数据.

    如何触发这两种模式?
    flowing mode: 注册data事件、resume方法、pipe方法
    paused mode: pause方法、移除data、unpipe方法


```javascript
// resume触发flowing mode
Readable.prototype.resume = function() {
    var state = this._readableState;
    if (!state.flowing) {
           debug('resume');
           state.flowing = true;
    resume(this, state);
  }
  return this;
}
```
在源码内部全部都是用`resume`方法来住触发flowing mode的(包括resume自身),而调用`resume`触发flowing mode的关键标志是`state.flowing`,源码通过`state.flowing`来判断是否调用`resume`.

那么,这个`state.flowing`来自哪里呢?  

其实`state.flowing`是来自于`_readableState`,`_readableState`同时是`ReadableState`的实例,由于源码量有千行我不便一一贴出来,[相关源码](https://github.com/nodejs/node/blob/master/lib/_stream_readable.js)可以自行查看,我们可以简单介绍下`ReadableState`.  

`ReadableState`其实是一个构造函数,用于记录`Readable`的各种状态和信息,例如读取模式、highWaterMask、编码(默认utf-8)、各种事件、缓冲区、flowing模式等.  

可以说,Readable内部各种状态转换或者缓存读取等操作,都需要依赖`ReadableState`提供的信息支持.  

我们都知道可读流有三种状态:

        readable._readableState.flowing = null
        readable._readableState.flowing = false
        readable._readableState.flowing = true
       
我们调用上述`resume` `pipe`等方法可以改变状态,但是最初始的  ` readable._readableState.flowing = null` 状态就是在`ReadableState`中定义的.

```javascript
function ReadableState(options, stream) {
...
this.flowing = null;
...
}
```
   
此时我们需要看一下Stream整个运行机制的示意图:

![](http://i1.piimg.com/567571/82c592bb2211a405.png)

我们只对Readable部分进行解读
>* _read:可读数据流的`_read`方法，可以将数据放入可读数据流,真正从外部读取数据的是此方法
>* push:`push` `unshift`等方法将数据压入**读缓存区**.
>* 读缓存区:以数组的形式存在,主要存储着`Buffer`等数据缓存
>* read:`read`方法将缓存中的数据读取,不直接从外部读取数据,而是读取读缓存区内的数据

通过`push`等方法将数据压入读缓存区的过程中,数据会以`Buffer`的形式被存在数组里,因此在`read`后会出现`Buffer`的形式.  
```javascript
var Read = require('stream').Readable;
var r = new Read();

r.push('hello');
r.push('world');
r.push(null);

console.log(r.read()) //<Buffer 68 65 6c 6c 6f 77 6f 72 6c 64>

```
为了避免以上情况,我们通常会选择如下操作:
```javascript
console.log(r.read().toString()); //helloworld
```
将数据`chunk`转化为`Buffer`的操作正是在`push` `unshift`等方法内部实现的,源码如下:
```javascript
Readable.prototype.push = function(chunk, encoding) {
  var state = this._readableState;

  if (!state.objectMode && typeof chunk === 'string') {
    encoding = encoding || state.defaultEncoding;
    if (encoding !== state.encoding) {
      chunk = Buffer.from(chunk, encoding); //转化为Buffer
      encoding = '';
    }
  }

  return readableAddChunk(this, state, chunk, encoding, false); 
};
```

之后我们就需要讨论`read`方法读取缓存数据的问题,首先我们熟悉一下`read`的用法.
>* `read`方法从系统缓存读取并返回数据。如果读不到数据，则返回null。

>* 该方法可以接受一个整数作为参数，表示所要读取数据的数量，然后会返回该数量的数据。如果读不到足够数量的数据，返回null。如果不提供这个参数，默认返回系统缓存之中的所有数据。

>* 只在“暂停态”时，该方法才有必要手动调用。“流动态”时，该方法是自动调用的，直到系统缓存之中的数据被读光。
>* 如果该方法返回一个数据块，那么它就触发了data事件。

```javascript
var Read = require('stream').Readable;
var r = new Read();

r.push('hello');
r.push('world');
r.push(null);
console.log(r.read(1).toString()); //h
```
上述代码表示只读取数量(n)为1的数据,可以初步窥探到`read`的基本用法.

现在我们通过阅读部分`read`源码来了解它的运作原理.
```javascript
Readable.prototype.read = function(n) {
    ...
  if (n === 0 && //当要读取的数据量为0,或者缓存已经满的时候,只触发Readable事件,不进行read读取
      state.needReadable &&
      (state.length >= state.highWaterMark || state.ended)) {
    debug('read: emitReadable', state.length, state.ended);
    if (state.length === 0 && state.ended)
      endReadable(this);
    else
      emitReadable(this);
    return null; //读不到数据返回null
  }

...
// 设置读取的数量
  n = howMuchToRead(n, state);

 //利用doRead判断是否可以开启可读流,
  var doRead = state.needReadable;

  // 如果缓存区为空,亦或者未超过设置的警戒线,则可以开启可读流
  if (state.length === 0 || state.length - n < state.highWaterMark) {
    doRead = true;
    debug('length less than watermark', doRead);
  }

  if (state.ended || state.reading) {//但是如果已经结束则停止可读流
    doRead = false;
    debug('reading or ended', doRead);
  } else if (doRead) {
    debug('do read');
    state.reading = true;
    state.sync = true;
    if (state.length === 0)
      state.needReadable = true;
    this._read(state.highWaterMark); //在可读流开启的情况下用_read方法读取一段警戒线大小的数据
    state.sync = false;
    if (!state.reading)
      n = howMuchToRead(nOrig, state);
  }

  var ret;
  if (n > 0)
    ret = fromList(n, state); //在缓存区的数组内提取数量为n的数据,用于从读缓存区的数据读取
  else
    ret = null;

  if (ret === null) {
    state.needReadable = true;
    n = 0;
  } else {
    state.length -= n;
  }

  if (state.length === 0) {
    if (!state.ended)
      state.needReadable = true;
    if (nOrig !== n && state.ended)
      endReadable(this);
  }

  if (ret !== null)
    this.emit('data', ret); //通过事件data将从缓存中读取到的数据交出去

  return ret;
};
...

}
```
我们可以梳理一下`Readable`的工作流程:
1.paused 模式下

```javascript
var Read = require('stream').Readable;
var r = new Read();

  r.push('hello');
  r.push('world');
  r.push(null);

console.log(r.read().toString());
```
以上述代码为例,我们梳理一下`Readbale`的内部流程.

>* 1.首先通过 `new Read()`来创建一个可读流,此时`readable._readableState.flowing = null`,而且这个可读流其他内部模式也被内部函数` ReadableState(options, stream)`初始化为初始状态并保存.
>* 2. `push`方法将数据转化为`Buffer`类型写入读缓存区内,缓存区以数组的形式存在.
>* 3. 再**手动**调用`read`方法进行对读缓存区的读取,首先判断此可读流是否结束,如果没有结束需要调用`_read`方法读取大小等于警戒线（highWaterMark）的数据,同时利用`fromList`去读取缓存中的数据(读取后就将缓存内的数据清除),随后通过`data`事件将数据交出.
>* 4. `_read`会调用`push`方法,`push`方法如果判定没有读取到结束符的情况下继续向缓存中写入数据,同时调用内部方法` maybeReadMore_`触发`read(0)`.
>* 5. 步骤3 与 步骤4 反复循环直到读取到结束符(null)或者出现错误而终止.


2 . flowing 模式下

我们可以分别用`pipe`方法与监听`data`事件的方式实现flowing mode,因为下文中会涉及到`pipe`方法,我们姑且以监听`data`事件为例,讲解内部流程.
```javascript
var Read = require('stream').Readable;
var r = new Read();

r.push('hello ');
r.push('world ');
r.push(null)

r.on('data', function (chunk) {
    console.log(chunk.toString())
})

```
>* 1.同paused 模式
>* 2. 监听`data`事件后将会自动调用该可读流的`resume`方法，使流切换至流动模式,`readable._readableState.flowing = flowing`,`resume`内部调用私有函数`_resume`,此函数产生`read`的自动调用.
>* 3. 同paused 模式
>* 4. 同paused 模式
>* 5. 同paused 模式



---

#### 2. Stream 中Writeable的运作流程

有了前面对Readable的分析,我们理解起Writeable就相对容易了很多,因为很多逻辑是相通的,无非是读入与输出的区别.

我们再回到上面我们列举的Stream运行示意图中关于Writeable的部分.  
![](http://i1.piimg.com/567571/b2eb4036eec3a0db.png)


我们再回到上面我们列举的Stream运行示意图中关于Writeable的部分:
> write:利用`write`接收到通过`pipe`传过来的数据`chunk`.
> writeOrBuffer:`writeOrBuffer`方法将数据写入**写缓存区**.
> 写缓存区:以链表的形式存在,主要存储着`Buffer`等数据缓存
> _write:`dowrite`调用 `_write`方法将缓存中的数据输出

Writeable的相关源码可以访问[这里](https://github.com/nodejs/node/blob/master/lib/_stream_writable.js),我们只贴出少部分关键源码以供参考.

跟Readable类似,Writeable也有一个记录、管理Writebale相关状态的`WritableState`对象.

我们知道,`write`方法负责将数据写入可写流中,而真正负责管理写入的是`writeOrBuffer`方法,其源码如下.
```javascript
function writeOrBuffer(stream, state, isBuf, chunk, encoding, cb) {
  if (!isBuf) { //对数据进行编码处理
    var newChunk = decodeChunk(state, chunk, encoding);
    if (chunk !== newChunk) {
      encoding = 'buffer';
      chunk = newChunk;
    }
  }
  var len = state.objectMode ? 1 : chunk.length;

  state.length += len;
//判断写入的数据时都超过了设定好的预警线,如果超过通过改变`needDrain`表示缓存区已满,停止写入
  var ret = state.length < state.highWaterMark;
  if (!ret)
    state.needDrain = true;

  if (state.writing || state.corked) { //如果可写流正在写数据,那么将数据写入缓存区
    var last = state.lastBufferedRequest;
    state.lastBufferedRequest = { chunk, encoding, callback: cb, next: null };
    if (last) {
      last.next = state.lastBufferedRequest;
    } else {
      state.bufferedRequest = state.lastBufferedRequest;
    }
    state.bufferedRequestCount += 1;
  } else {
    doWrite(stream, state, false, len, chunk, encoding, cb);//如果不在写数据,那么调用`doWrite`方法
  }

  return ret;
}
```
实际上,`doWrite`方法也不是将缓存区数据输出的具体方法,它会调用`_write`,而`_write`会触发回调函数`state.onwrite`,

```javascript
function onwrite(stream, er) {
  var state = stream._writableState;
  var sync = state.sync;
  var cb = state.writecb;

  onwriteStateUpdate(state);

  if (er)
    onwriteError(stream, state, sync, er, cb);
  else {
    var finished = needFinish(state);

    if (!finished && //将缓存数据输出
        !state.corked &&
        !state.bufferProcessing &&
        state.bufferedRequest) {
      clearBuffer(stream, state);
    }

    if (sync) { 
      process.nextTick(afterWrite, stream, state, finished, cb);
    } else {
      afterWrite(stream, state, finished, cb);
    }
  }
}
```

`doWrite`大致工作机制如下:首先调用`clearBuffer`方法将缓存区的数据依次输出并清空,随后触发`afterWrite`方法,进行判断是否结束可写流或者触发`drain`事件通知继续将数据写入可写流.

我们梳理的可写流的大概运作机制如下:
>* 1.首先创建一个可读流,并且通过数` WriteableState`进行初始化.
>* 2. `write`方法通过调用 `writeOrBuffer`管理写入写缓存区的操作,如果符合条件就写入缓存区.
>* 3. 同时`writeOrBuffer`中`doWrite`方法负责将缓存区的数据输出,`doWrite`会调用`_write`方法,而`_write`会触发回调函数`state.onwrite`,`doWrite`首先调用`clearBuffer`方法将缓存区的数据依次输出并清空,随后触发`afterWrite`方法,进行判断是否结束可写流或者触发`drain`事件通知继续将数据写入可写流,再次进入步骤2的流程.
>* 4. 直到触发`end`方法结束该可写流.




