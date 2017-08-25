# JavaScript的"预解释"

---

### 前言:  

 JavaScript的作用域一直是JavaScript比较让人头痛的一部分，也是面试中几乎必考的内容，因此，我们将从更深层次来讲述js作用域。

---

#### 1.从一个实例开始  

仔细阅读以下JavaScript代码,你觉得运行结果会是什么呢?是 `1` 还是`2`?
``` javascript
var a= 1;
function f() {
  console.log(a);
  var a = 2;
}
f();
```

答案是undefined,我们在Atom编辑器内运行的结果如下:

![Markdown](http://p1.bpimg.com/586294/b14e2d8434d1433b.png)

那么到底是什么原因导致了这个让人意外的结果呢?这就要从JavaScript解释阶段说起。

---

#### 2.JavaScript预解释  

我们可以大致把JavaScript在浏览器中运行的过程分为两个阶段`预解释阶段`（有人说准确的说法是应该是Parser，我们以预解释方便理解） `执行阶段`,在JavaScript引擎对JavaScript代码进行执行之前,需要进行预先处理,然后再对处理后的代码进行执行。

>  我们平时书写的JavaScript代码并不是JavaScript执行的代码(V8引擎读取一行执行一行这种理解是错误的),它需要预解释后,再由引擎进行执行.

具体的解释过程涉及到浏览器内核的技术不属于前端领域,不过我们可以浅显的理解一下V8在处理JavaScript的一般过程:

以上例中的`var a = 2; `为例,我们一般人的理解为**声明了一个值为2的变量a**,但是在JavaScript引擎处理时却分为了两个步骤:
>1. 读取`var a`后,在当前作用域中查找是否有相同声明,如果没有就在当前作用域集合中创建一个名为`a`的变量,否则忽略此声明继续进行解析.

>2. 接下来,V8引擎会处理`a = 2`的赋值操作,首先会询问当前作用域中是否有名为`a`的变量,如果有进行赋值,否则继续向上级作用域询问.

---

#### 3.JavaScript执行环境  

我们上面提到的所谓javascript预解释正是创建函数的**执行环境**（又称“执行上下文”），只有搞定了javascript的执行环境我们才能搞清楚一段代码在执行过后为什么产生这样的结果。

我们用一段伪代码表示创立的**执行环境**
```javascript
executionContextObj = {
    'scopeChain': { /* 变量对象 + 所有父级执行上下文中的变量对象 */ },
    'variableObject': { /*  函数参数 / 参数, 内部变量以及函数声明 */ },
    'this': {}
}
```
**作用域链(scopeChain)**包括下面提到的变量对象(variableObject)和所有父级执行上下文中的变量对象.

**变量对象(variableObject)**是与执行上下文相关的数据作用域,一个与上下文相关的特殊对象，其中存储了在上下文中定义的变量和函数声明:  
     变量;  
     函数声明;  
     函数的形参

在有了这些基板概念之后我们可以梳理一下js引擎创建执行的过程:  
* 创建阶段
    * 创建Scope chain  
    * 创建variableObject
    * 设置this
* 执行阶段
    * 变量的值、函数的引用  
    * 执行代码

而变量对象的创建细节如下:

* 根据函数的参数，创建并初始化arguments object
* 扫描函数内部代码，查找函数声明（Function declaration）
    * 对于所有找到的函数声明，将函数名和函数引用存入变量对象中
    * 如果变量对象中已经有同名的函数，那么就进行覆盖
* 扫描函数内部代码，查找变量声明（Variable declaration）
    * 对于所有找到的变量声明，将变量名存入变量对象中，并初始化为"undefined"
    * 如果变量名称跟已经声明的形式参数或函数相同，则变量声明不会干扰已经存在的这类属性


---

#### 4.变量提升  

正是由于以上的处理,产生了大家熟知的JavaScript中的**变量提升**,具体以上代码的执行过程如以下伪代码所示:
```javascript
// global context
executionContextObj = {
    'scopeChain': { ... },
    'variableObject': { a: undefined, f: pointer to function f() },
    'this': {...}
}
...
}//首先在全局执行环境中声明了变量a以及函数f,此时a虽然被声明,但是尚未赋值
x = 1;
function f() {
    executionContextObj {
    'scopeChain': { ... },
    'variableObject': {        
    arguments: {}, 
    a: undefined 
        },
    'this': {...}
    }
    //内部词法环境中声明了变量a,此时a虽然被声明,但是尚未赋值
    console.log(a);//此时a需要被被打印出来,在作用域内寻找a变量赋值,于是被赋值undefined
    a = 2;
}
```

我们可以明显看到,`a`变量在预解释阶段已经被赋值`undefined`,在执行阶段js是自上而下单线执行，当`console.log(a)`执行之时,`a=2`还没有被执行,`a`变量的值便是预处理阶段被赋予的`undefined`,

---
#### 5.函数声明与函数表达式
我们看到,在编译器处理阶段,除了被`var`声明的变量会有变量提升这一特性之外,函数也会产生这一特性,但是函数声明与函数表达式两种范式创建的函数却表现出不同的结果.  


我们先看一个实例,运行以下代码
```javascript
f();
g();
//函数声明
function f() {
    console.log('f');
}
//函数表达式
var g = function() {
    console.log('g');
};

```

`f`成功被打印出来,而`g函数`出现了类型错误,这是什么原因呢?  
![Markdown](http://p1.bpimg.com/586294/658277838f700ab6.png)

```javascript
executionContextObj = {
    'scopeChain': { ... },
    'variableObject': { f: pointer to function f(), g: undefined},
    'this': {...}
}

f();
g();
//函数声明
function f() {
    console.log('f');
}
//函数表达式
var g = function() {
    console.log('g');
};
```
我们看到,在预解释阶段函数声明的`f`是被指向了正确的函数得以执行,而函数表达式`g`被赋予`undefined`,`undefined`无法被当作函数执行因此报错`g is not a function`.


---
#### 6.冲突处理

通常情况下我们不会将同一变量变量重复声明,但是出现了类似情况后,编译器会如何处理这些冲突呢?
1. 变量之间冲突
执行以下函数:
```javascript
var a = 3;
var a = 4;
console.log(a);
```
结果显而易见,后声明变量值覆盖前者的值
2. 函数之间冲突
```javascript
f();
function f() {
    console.log('f');
}

function f () {
    console.log('g');
};
```
结果同变量冲突,后者覆盖前者.

3. 函数与变量之间冲突

```javascript
console.log(f);

function f() {
    console.log('f');
}
var f ='g';
```
结果如下,函数声明将覆盖变量声明.
`[Function: f]`
---
#### 7.ES6中的let

在ES6中出现了两个最新的声明语法`let`与`const`,我们以`let`为例,进行测试看看与`var`的区别.
```javascript
function f() {
  console.log(a);
  let a = 2;
}
f(); // ReferenceError: a is not defined
```
这段代码直接报错显示未定义,`let`与`const`拥有类似的特性,阻止了变量提升,当代码执行到`console.log(a)`时,执行换将中`a`还从未被定义,因此产生了错误.