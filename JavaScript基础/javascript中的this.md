# JavaScript面向对象之this


---
#### 1.什么决定了`this`的指向

　　`this`一直是JavaScript中十分玄乎的存在,很多人为了避开这个琢磨不透的东西,选择了尽量少得运用`this`,但是不可否认的是,正是因为`this`的存在才使得JavaScript拥有了更加灵活的特性,因此,搞清楚`this`是每一个JavaScript学习者的必修课.

　　`this`之所以让人又爱又恨,正是因为它的指向让人琢磨不透,在进行详细讲解之前,我们要搞清楚一个大前提,`this`的指向不是在编写时确定的,而是在执行时确定的.
```javascript
obj = {
  name: "Messi",
  sayName: function () {
    console.log(this.name);
  }
};

obj.sayName(); //"Messi"

var f = obj.sayName;
f(); //undefind
console.log(f === obj.sayName); //true
```
　　很明显,虽然`f`与`obj.sayName`是等价的,但是他们所产生的结果却截然不同,归根到底是因为它们调用位置的不同造成的.

　　`f`的调用位置在全局作用域,因此`this`指向`window`对象,而`window`对象并不存在`name`因此会显示出`undefind`,而`obj.sayName`的`this`指向的是`obj`对象,因此会打印出`"Messi"`.

　　我们可以在以下代码中加入`name = "Bale";`来证明以上说法.

```javascript
name = "Bale";
obj = {
  name: "Messi",
  sayName: function () {
    console.log(this.name);
  }
};

obj.sayName(); //"Messi"

var f = obj.sayName;
f(); //"Bale"
console.log(f === obj.sayName);  //true
```
　　大家一定会好奇,调用位置是如何决定`obj.sayName`的`this`指向`obj`对象,`f`却指向`window`对象呢,其中遵循什么规则吗?

---
#### 2.默认绑定
　　`this`一共存在4种绑定规则,默认绑定是其中最常见的,我们可以认为当其他三个绑定规则都没有体现时,就用的是默认的绑定规则.

```javascript
name = "Bale";

function sayName () {
    console.log(this.name);
};

sayName(); //"Bale"
```
　　以上代码可以看成我们第一节例子中的`f`函数,它之所以指向`window`对象,就是运用了`this`**默认绑定**的规则,因为此实例代码中既没有运用`apply` 　`bind`等显示绑定,也没有用`new`绑定,不适用于其他绑定规则,因此便是**默认绑定**,此时的`this`指向全局变量,即浏览器端的`window`Node.js中的`global`.

---
#### 3.隐式绑定
　　当函数被调用的位置存在上下文对象,或者说被某个对象拥有或包含,这时候函数的`f`的`this`被**隐式绑定**到`obj`对象上.
```javascript
function f() {
console.log( this.name );
}
var obj = {
name: "Messi",
f: f
};
obj.f(); // Messi

```

---
####  4.显式绑定

　　除了极少数的宿主函数之外,所有的函数都拥有`call` `apply`方法,而这两个大家既熟悉又陌生的方法可以强制改变`this`的指向,从而实现显式绑定.

`call` `apply`可以产生对`this`相同的绑定效果,唯一的区别便是他们参数传入的方式不同.
>**call方法**:   
**语法**：call([thisObj[,arg1[, arg2[,   [,.argN]]]]])   
**定义**：调用一个对象的一个方法，以另一个对象替换当前对象。   
**说明**：   
　　call 方法可以用来代替另一个对象调用一个方法。call 方法可将一个函数的对象上下文从初始的上下文改变为由 thisObj 指定的新对象。 
　　如果没有提供 thisObj 参数，那么 Global 对象被用作 thisObj。 


>**apply方法**：   
**语法**：apply([thisObj[,argArray]])   
**定义**：应用某一对象的一个方法，用另一个对象替换当前对象。   
**说明**： 
　　如果 argArray 不是一个有效的数组或者不是 arguments 对象，那么将导致一个 TypeError。   
如果没有提供 argArray 和 thisObj 任何一个参数，那么 Global 对象将被用作 thisObj， 并且无法被传递任何参数。


　　第一个参数意义都一样。第二个参数：apply传入的是一个参数数组，也就是将多个参数组合成为一个数组传入，而`call`则作为`call`的参数传入（从第二个参数开始）。  
　　如 `func.call(func1,var1,var2,var3)`  对应的`apply`写法为：`func.apply(func1,[var1,var2,var3])`，同时使用`apply`的好处是可以直接将当前函数的`arguments`对象作为`apply`的第二个参数传入。 　

```javascript
function f() {
console.log( this.name );
}
var obj = {
name: "Messi",

};
f.call(obj); // Messi
f.apply(obj); //Messi
```
我们可以看到,效果是相同的,`call`  `apply`的作用都是强制将`f`函数的`this`绑定到`obj`对象上.
在ES5中有一个与`call` `apply`效果类似的`bind`方法,同样可以达成这种效果,

>**`Function.prototype.bind()`** 的作用是将当前函数与指定的对象绑定，并返回一个新函数，这个新函数无论以什么样的方式调用，其 this 始终指向绑定的对象。

```javascript
function f() {
    console.log( this.name );
}
var obj = {
    name: "Messi",
};

var obj1 = {
     name: "Bale"
};

f.bind(obj)(); //Messi ,由于bind将obj绑定到f函数上后返回一个新函数,因此需要再在后面加上括号进行执行,这是bind与apply和call的区别

```

---
#### 5.new绑定
用 new 调用一个构造函数，会创建一个新对象, 在创造这个新对象的过程中,新对象会自动绑定到`Person`对象的`this`上，那么 `this` 自然就指向这个新对象。
这没有什么悬念，因为 new 本身就是设计来创建新对象的。
```javascript
function Person(name) {
  this.name = name;
  console.log(name);
}

var person1 = new Person('Messi'); //Messi
```

---


#### 6.绑定优先级


通过以上的介绍,我们知道了四种绑定的规则,但是当这些规则同时出现,那么谁的优先级更高呢,这才有助于我们判断`this`的指向.
通常情况下,按照优先级排序是:
**new绑定 > 显式绑定 >隐式绑定 >默认绑定**

我们完全可以通过这个优先级顺序判断`this`的指向问题.


---

#### 7.ES6箭头函数中的this

箭头函数不同于传统JavaScript中的函数,箭头函数并没有属于自己的`this`,它的`this`是捕获其所在上下文的  this 值，作为自己的 this 值,并且由于没有属于自己的`this`,箭头函数是不会被`new`调用的.

MDN文档中关于箭头函数的实例很清楚的说明了这一点.

在 ECMAScript 3/5 中，这个问题可以通过新增一个变量来指向期望的 this 对象，然后将该变量放到闭包中来解决。
```javascript
function Person() {
  var self = this; // 也有人选择使用 `that` 而非 `self`. 
                   // 只要保证一致就好.
  self.age = 0;

  setInterval(function growUp() {
    // 回调里面的 `self` 变量就指向了期望的那个对象了
    self.age++;
  }, 1000);
}
```
除此之外，还可以使用 bind 函数，把期望的 this 值传递给 growUp() 函数。

箭头函数则会捕获其所在上下文的  this 值，作为自己的 this 值，因此下面的代码将如期运行。
```javascript
function Person(){
  this.age = 0;

  setInterval(() => {
    this.age++; // |this| 正确地指向了 person 对象
  }, 1000);
}

var p = new Person();
```

当然,我们用babel转码器,也可以让我们更清楚理解箭头函数的`this`

```javascript
// ES6
const obj = {
    getArrow() {
        return () => {
            console.log(this === obj);
        };
    }
} 
```
```javascript
// ES5，由 Babel 转译
var obj = {
    getArrow: function getArrow() {
        var _this = this;
        return function () {
            console.log(_this === obj);
        };
    }
};
```