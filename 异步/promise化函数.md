# 如何将node中默认的回调方式改为Promise

---

### 1. node默认的异步方式


`nodejs`默认的异步方式是回调函数,由于没有提供`Promise`接口导致我们常用的例如`fs`方法无法使用最新的es6+语法.

例如我们读取一个文件:

```javascript
fs.readFile('package.json', 'utf-8', (err, data) => {
  if (err) {
    throw err;
  } else {
    console.log(data);
  }
})
```

### 2. 解决方案

目前通常的解决方案是引入npm包来对当前的函数进行包装,从而使得相关函数拥有更高级的功能.

众所周知的 `Bluebird` 拥有 `promisify` 这方法可以将回调函数风格转换为`Promise`方法.

比如`mz`模块就是一个使得低版本node拥有es6+标准的工具,当然它不仅仅可以使得回调函数拥有`Promise`能力,其它es6+标准中出现的新的api都可以通过它实现.

当然`pify`就是一个专门使得回调函数拥有`Promise`接口的模块了.

```javascript
pify(fs.readFile)('package.json', 'utf8').then(data => {
	console.log(JSON.parse(data).name);
}).catch(err => {
  throw err;
});
```

当然如果在`Node v8.0.0`后可以使用`util.promisify(original)`.

```javascript
const util = require('util');
const fs = require('fs');

const stat = util.promisify(fs.stat);
stat('.').then((stats) => {
  // Do something with `stats`
}).catch((error) => {
  // Handle the error.
});
```

### 3. 源码实现

如果只是简单的实现其实难度并不是很大.

```javascript
let promisify = (fn, receiver) => {
  return (...args) => {
    return new Promise((resolve, reject) => {
      fn.apply(receiver, [...args, (err, res) => {
        return err ? reject(err) : resolve(res);
      }]);
    });
  };
};
```
例如读取一个项目名称:

```javascript
const fs = require('fs');

const promisify = (fn, receiver) => {
  return (...args) => {
    return new Promise((resolve, reject) => {
      fn.apply(receiver, [...args, (err, res) => {
        return err ? reject(err) : resolve(res);
      }]);
    });
  };
};

const readFilePromise = promisify(fs.readFile, fs);

readFilePromise('./package.json', 'utf-8').then(data => {
  console.log(JSON.parse(data).name);
}).catch(err => {
  throw err;
})

```
