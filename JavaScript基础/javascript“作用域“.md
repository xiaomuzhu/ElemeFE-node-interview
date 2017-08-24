# JavaScript面向对象之作用域

标签（空格分隔）： JavaScript 作用域

---
#### 1. 为什么要理解作用域  

原因很简单,JavaScript中最重要的一个概念**闭包**的理解就建立在对**作用域**的理解之上,而一个对象的的构成往往离不开**闭包**以及**作用域**.

---
#### 2. 动态作用域or静态作用域?  
首先我们要搞清楚JavaScript的作用域类型,这有助于我们在分析作用域时的判断.
> 静态作用域:静态作用域是指声明的作用域是根据程序正文在编译时就确定的，有时也称为词法作用域。

> 动态作用域:程序中某个变量所引用的对象是在程序运行时刻根据程序的控制流信息来确定的。

大多数现代编程语言都采用的静态作用域,即代码在写出来的时候就已经确定的,并非在执行时再确定,我们可以根据以下代码一探究竟.
```javascript
function f() {
  console.log(a);
}
function g() {
  var a = 7;
  f();
}
g(); // a is not defined
```
这段代码在执行时候会报错,很明显,如果JavaScript采用了动态作用域,`a`在执行时确定的话,那么以上代码相当于这样:
```javascript
function g() {
    var a = 7;
    function f() {
    console.log(a);
    }
}
g(); //undefind
```
因此,我们可以判断出JavaScript属于静态作用域.


---
#### 3.函数作用域
大家都知道,函数是JavaScript的一等公民,那么函数是存在自身作用域的,在创建函数之初,函数体内就产生了作用域,为了方便理解,我们引用了《你不知道的JavaScript》书中的代码及图例,他会很清晰地帮助我们理解函数作用域.
```javascript
function foo(a) {
    var b = a * 2;
    function bar(c) {
        console.log(a, b, c);
    }
    bar(b * 3);
}
foo(2); // 2, 4, 12
```
以上实例中的作用域是什么样子的呢?  
![Markdown](http://p1.bqimg.com/586294/d9c1fe993e874903.png)

我们可以通过这张图清楚地看到一个函数的作用域包含着什么,而且由于JavaScript是采用静态作用域,作用域是在函数创建的时候就确定下来的.

---

#### 4.作用域链

那么,我们可以仔细分析一下这个作用域的生成过程.

首先我们分析一下函数foo:

函数`foo`在这个嵌套函数体的最外层,在V8预编译这个`foo`后,创建`foo`函数时给此函数添加了一个成员`scopeChain`,这个`scopeChain`指向创建此`foo`的执行环境(由于是在全局环境下创建,因此指向Window或者Global).

在`foo`被创建后,它会形成自身的执行环境.
```javascript
我们可以把这个创建foo的过程简单地表示为: foo.executionContext.scopeChain -> window.executionContext{...},即foo的scopeChain指向创建它的window的执行环境.
同时:
foo.executionContext.variableObject{
        arguments: {
            0: 2,
            length: 1
        },
        a: 2,
        bar: pointer to function bar()
}

同理,`bar`创建时:
bar.executionContext = {
    scopeChain: { pointer to foo.executionContext },
    variableObject: {
        arguments: {
            0: 12,
            length: 1
        },
        b: 12,
    },
    this: { ... }
}

```
在函数执行过程中,`bar`在自身执行环境中为变量abc赋值,但是发现`ab`都为`undefind`,但是它可以通过`bar.executionContext.scopeChain -> foo.executionContext `这个链条向上寻找父级执行环境中的变量,直到找到赋值对象或者到达顶层也就是window,为其赋值,这就是作用域链.

---

#### 5. 块级作用域

在ES2015之前,JavaScript中实际上是没有语法层面的块级作用域,这就造成了很多意外的产生.
```javascript
for (var i = 0; i<3; i++) {

}
console.log(i); //3
```
如果是在有块级作用域的语言中,`i`是不会被打印出来的,但是在JavaScript中却被打印出来,这就是变量泄露的情况,也就是说看似在块级作用域的变量泄漏到全局作用域中,这也就造成了全局污染.

在ES5中,人们为了解决这个问题,一般采用立即执行函数IIFE来模拟块级作用域,但是这种写法不易读也不优雅,因此,在ES2015中引入了`let`,通过`let`可以创建块级作用域.

> `let`与`var`在使用上基本是类似的,但是`let`有三个主要的特点

> * 可创建块级作用域
> * 不存在变量提升
> * 存在暂时性死区

例如上面的代码如果改用`let`声明,就不存在变量污染全局的情况
```
for (let i = 0; i<3; i++) {

}
console.log(i); //i is not defind
```
至于其它let的具体用法,可以直接参考[《ES6入门教程》](http://es6.ruanyifeng.com/#docs/let).


























