# Node中Buffers详解


---

#### **前言**

Buffer 是 Node.js 中用于处理二进制数据的类, 其中与 IO 相关的操作 (网络/文件等) 均基于 Buffer,Buffer类是Node中的一个全局变量,这就意味着你不需要用额外的`require`将模块引入就可以使用它.

值得一提的是,在`Node 6.x`之前,我们都是用`new buffer`进行buffer相关的操作,但是随着ES2015的普及和`new buffer`自身安全性的问题,这个api已经被废弃.

因此,我们需要重新探究在新的标准下buffer的使用

在阅读本文前,你可以先了解一下ES2015中[二进制数组](http://es6.ruanyifeng.com/?search=buffer&x=0&y=0#docs/arraybuffer)的相关知识.

---
#### 1.Buffer类的使用

#### 1.1 创建buffer

由于`new buffer`的废弃,用来替换它的Buffer.from()、Buffer.alloc()、和 Buffer.allocUnsafe() 方法应运而生。

| 方法 | 用途 |
|---|---|
|Buffer.from()|根据已有数据生成一个 Buffer 对象|
|Buffer.alloc()|创建一个初始化后的 Buffer 对象|
|Buffer.allocUnsafe()|创建一个未初始化的 Buffer 对象|

```javascript
//创建一个根据数组[1,2,3]生成的buffer
const buffer1 = Buffer.from([1,2,3]);
console.log(buffer1); //<Buffer 01 02 03>

// 创建一个长度为 10、且用 0x1 填充的 Buffer。
const buffer2 = Buffer.alloc(10, 1);
console.log(buffer2); //<Buffer 01 01 01 01 01 01 01 01 01 01>

// 创建一个长度为 10、且未初始化的 Buffer。
const buffer3 = Buffer.allocUnsafe(10);
console.log(buffer3); //<Buffer 00 00 00 00 00 00 00 00 00 00>

```

#### 1.2buffer转换

我们先设想一个场景,目前有一个`1.txt`文件,内容如下:
```
aaa
bbb
ccc
ddd

```
我们需要读取`1.txt`内容
```javascript
const fs = require('fs');

fs.readFile('1.txt', (err, data) => {
  console.log(data);
}); //<Buffer 61 61 61 0d 0a 62 62 62 0d 0a 63 63 63 0d 0a 64 64 64 0d 0a>

```
此时会发现读取出来的是二进制的类数组,此时我们就需要进行buffer编码转换.

```javascript
fs.readFile('1.txt', (err, data) => {
  console.log(data.toString()); //转换成字符串
 });
// aaa
// bbb
// ccc
// ddd
```

以上就是buffer编码转换的应用场景之一,Buffer是可以与字符串进行转换,但是仅限于以下编码格式:

    
    'ascii' - 仅支持 7 位 ASCII 数据。如果设置去掉高位的话，这种编码是非常快的。
    
    'utf8' - 多字节编码的 Unicode 字符。许多网页和其他文档格式都使用 UTF-8 。
    
    'utf16le' - 2 或 4 个字节，小字节序编码的 Unicode 字符。支持代理对（U+10000 至 U+10FFFF）。
    
    'ucs2' - 'utf16le' 的别名。
    
    'base64' - Base64 编码。当从字符串创建 Buffer 时，按照 RFC4648 第 5 章的规定，这种编码也将正确地接受“URL 与文件名安全字母表”。
    
    'latin1' - 一种把 Buffer 编码成一字节编码的字符串的方式（由 IANA 定义在 RFC1345 第 63 页，用作 Latin-1 补充块与 C0/C1 控制码）。
    
    'binary' - 'latin1' 的别名。
    
    'hex' - 将每个字节编码为两个十六进制字符。


同样的,我们可以将字符串转换成`base64`编码
```javascript
let user = 'Messi';
let pass = '1987';
let authstring = user + ':' +pass;
let encoded = Buffer.from(authstring).toString('base64');
console.log(encoded); //TWVzc2k6MTk4Nw==
```



#### 1.3 Buffer拼接

说起Buffer拼接,就不得不提其中的一个坑,那就是中文.

```javascript
let str = '世界,你好';
let bf = Buffer.from(str);
console.log(bf.slice(0,8).toString()); //世界,�
```
由于一个汉字所占的字节为3,因此在处理汉字的时候,上例只取了8个字节,所以相当于`世界`占了6个字节,`,`占了一个字节,`你`我们只取了其中一个字节,因此`你`就显示乱码了.

中文乱码的问题是node.js中十分常见的坑,在用流读取文件时最容易产生中文乱码的情况.

我们读取中文文本文件`2.txt`内容如下:
```
王者农药是最农药的游戏

```
进行流读取操作:
```javascript
const fs = require('fs');

let rs = fs.createReadStream('./2.txt', {
    highWaterMark: 5 //最大限制5个字节读取
});
let txt = '';
rs.on('data',(chunk) => {
    txt += chunk;
});
rs.on('end',() => {
    console.log(txt);
});
//王���农���是最���药���游戏
```
这里出现乱码了，是因为`txt += chunk`隐含了一个操作，即 `chunk.toString()`，因为txt是String类型的。因此相当于是`txt += chunk.toString()`.

由于每次限定只读取5个字节，所以不少汉字被截断,造成了大量乱码的情况.


一个可行的办法是,先将读取到的多个小`Buffer`拼接成一个大`Buffer`,再对大`Buffer`进行转换,此时就避免了单个转换造成的部分中文字节不完整造成的乱码现象.

```javascript
const fs = require('fs');

let rs = fs.createReadStream('./2.txt', {
    highWaterMark: 5
});
let buf = [];
let size = 0;
rs.on('data',chunk => {
    buf.push(chunk);
    size += chunk.length;
});
rs.on('end',() => {
    let buffer1 = Buffer.concat(buf, size);
    console.log(buffer1.toString()); //王者农药是最农药的游戏
});
```
但是如果文件过大,使得大`Buffer`超过了我们的内存,此时需要借助第三方模块`string_decoder`来进行处理,有兴趣的话可以自行探究.

---

#### 3.Buffer底层探究

---

#### 3.1 Buffer的组成

`Buffer`是Node中很常见的由JavaScript与c++组合而成的模块,c++负责底层和性能的部分,其余则是由js来负责,因此由于有了c++的支持,`Buffer`的性能还是很有保证的.
![](http://i2.muimg.com/567571/d52d38adb442582c.png)


#### 3.2 Buffer与内存

我从网络上寻找到了一张图形象地解释了Buffer的内存分配管理体系.  
![](http://i4.buimg.com/567571/226ba23149c08c5e.png)

3.2.1 `Buffer.from`  

`Buffer.from`的作用是**根据已有数据生成一个 Buffer 对象**,那么它的内部是如何运作的呢?

我们可以看看源码,一探究竟.
```javascript
Buffer.from = function(value, encodingOrOffset, length) {
  if (typeof value === 'number')
    throw new TypeError('"value" argument must not be a number');

  if (value instanceof ArrayBuffer)
    return fromArrayBuffer(value, encodingOrOffset, length);

  if (typeof value === 'string')
    return fromString(value, encodingOrOffset);

  return fromObject(value);
};
```
**1.`ArrayBuffer`**

我们可以清楚地看到,当`value`为ArrayBuffer的实例,将会返回`return fromArrayBuffer(value, encodingOrOffset, length)`.

`return fromArrayBuffer(value, encodingOrOffset, length)`会通过返回`binding.createFromArrayBuffer(obj, byteOffset, length)`来操作c++模块进行底层处理,源码如下:

```javascript
// fromArrayBuffer:
function fromArrayBuffer(obj, byteOffset, length) {
  byteOffset >>>= 0;

  if (typeof length === 'undefined')
    return binding.createFromArrayBuffer(obj, byteOffset);

  length >>>= 0;
  return binding.createFromArrayBuffer(obj, byteOffset, length);
}
// c++ 模块:
void CreateFromArrayBuffer(const FunctionCallbackInfo<Value>& args) {
  ...
  Local<ArrayBuffer> ab = args[0].As<ArrayBuffer>();
  ...
  Local<Uint8Array> ui = Uint8Array::New(ab, offset, max_length);
  ...
  args.GetReturnValue().Set(ui);
}
```
    如果不懂c++也没关系,你只需要知道这一系列操作是这样的:
            1. `Buffer.from`通过已有的`ArrayBuffer`数据生成了`Buffer` 对象.
            2. 此时的`Buffer`对象并没有重新申请内存而是与`ArrayBuffer`共享了内存.

我们可以通过实例来证明这一点:
```javascript
let a = new ArrayBuffer(8);
let b = new Uint8Array(a);
let c = Buffer.from(a);

console.log(b); //Uint8Array [ 0, 0, 0, 0, 0, 0, 0, 0 ]
console.log(c); //<Buffer 00 00 00 00 00 00 00 00>

b[0] = 11;
console.log(b); //Uint8Array [ 11, 0, 0, 0, 0, 0, 0, 0 ]
console.log(c);  //<Buffer 0b 00 00 00 00 00 00 00>
```
我们看到,`b`被修改后,`c`也跟着发生了变动,可见内存是共享的.

**2. `string`**

在`value === 'string'`的情况下返回`fromString(value, encodingOrOffset)`,fromString的源码如下:

```javascript
function fromString(string, encoding) {
  ...
  var length = byteLength(string, encoding);
  if (length === 0)
    return Buffer.alloc(0);
  // 进行判断,当字节大于4KB会直接分配内存
  if (length >= (Buffer.poolSize >>> 1))
    return binding.createFromString(string, encoding);
  // 当小于4KB,会通过pool进行分配
  if (length > (poolSize - poolOffset))
    createPool();
  var actual = allocPool.write(string, poolOffset, encoding);
  var b = allocPool.slice(poolOffset, poolOffset + actual);
  poolOffset += actual;
  alignPool();
  return b;
}
```
    以上源码相当于对根据字符串创建`Buffer`所需字节进行一个判断,所需大于4KB会直接创建内存,当所需小于4KB会借助pool进行分配,原因是为了合理分配内存,避免内存的浪费.
    
**3.`Buffer/TypeArray/Array`**
在`value`既不是`string`也不是`ArryBuffer`的情况下,会返回`fromObject(value)`,其相关源码如下:
```javascript
function fromObject(obj) {
  // 当为Buffer时
  if (obj instanceof Buffer) {
    ...
    const b = allocate(obj.length);
    obj.copy(b, 0, 0, obj.length);
    return b;
  }
  // 当为TypeArray或Array时
  if (obj) {
    if (obj.buffer instanceof ArrayBuffer || 'length' in obj) {
      ...
      return fromArrayLike(obj);
    }
    if (obj.type === 'Buffer' && Array.isArray(obj.data)) {
      return fromArrayLike(obj.data);
    }
  }

  throw new TypeError(kFromErrorMsg);
}
// 进行copy
function fromArrayLike(obj) {
  const length = obj.length;
  const b = allocate(length);
  for (var i = 0; i < length; i++)
    b[i] = obj[i] & 255;
  return b;
}

```
    与`ArryBuffer`的情况不同,`fromObject(obj)`进行了复制,另创建了新的内存空间,并没有进行内存共享.
我们看一个实例:

```javascript
let a = Buffer.from([1,2,3]);
let b = Buffer.from(a);
console.log(a); //<Buffer 01 02 03>
console.log(b); //<Buffer 01 02 03>

a[1] = 12;
console.log(a); //<Buffer 01 0c 03>
console.log(b); //<Buffer 01 02 03>
```

明显看出来,`b`并没有受到`a`值改变的影响,因为它另创建了自身的内存空间,所以`a` `b`内存空间相互独立,互不影响.

3.2.2 `Buffer.alloc` `Buffer.allocUnSafe` `Buffer.allocUnsafeSlow`

`Buffer.alloc`:先`createBuffer`申请内存，然后再进行填充,会覆盖旧数据,安全性较好,不借助`Buffer pool`而是直接分配内存.
`Buffer.allocUnSafe`:借助`Buffer pool`分配内存,不会覆盖旧数据,安全性较差
`Buffer.allocUnsafeSlow`:直接分配内存,但是不会对旧数据覆盖,因此安全性也较差.





