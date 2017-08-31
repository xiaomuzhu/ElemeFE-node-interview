# JavaScript面向对象之原型

---

#### 1.原型对象
绝大部分的函数(少数内建函数除外)都有一个`prototype`属性,这个属性是原型对象用来创建新对象实例,而所有被创建的对象都会共享原型对象,因此这些对象便可以访问原型对象的属性,例如`hasOwnProperty()`方法存在于Obejct原型对象中,它便可以被任何对象当做自己的方法使用.
> `object.hasOwnProperty( propertyName )`
> `hasOwnProperty()`函数的返回值为`Boolean`类型。如果对象`object`具有名称为`propertyName`的属性，则返回`true`，否则返回`false`。
```javascript
 var person = {
    name: "Messi",
    age: 29,
    profession: "football player"
  };
console.log(person.hasOwnProperty("name")); //true
console.log(person.hasOwnProperty("hasOwnProperty")); //false
console.log(Object.prototype.hasOwnProperty("hasOwnProperty")); //true
```
由以上代码可知,`hasOwnProperty()`并不存在于`person`对象中,但是`person`依然可以拥有此方法.

很多人此时会好奇,`person`对象是如何找到`Object`对象中的方法的呢?这其中的内部机制是什么?
这便是我们接下来要说的原型链.


---

#### 2.`__proto__`与`[[Prototype]]`

上一篇我们的示意图中曾经出现过`__proto__`,在ES6之前这个`__proto__`是大部分主流浏览器(IE除外)引擎提供的,还尚属非ECMA标准,在解析一个对象实例的时候为对象实例添加一个`__proto__`属性,此属性指向原型对象,我们便可以通过此属性找到原型对象.
```javascript
function person(pname, page) {
  this.name = pname;
  this.age = page;
}
person.prototype.profession = "football player";
var person1 = new person("Messi", 29); //person1 = {name:"Messi", age: 29, profession: "football player"};
var person2 = new person("Bale", 28); //person2 = {name:"Bale", age: 28, profession: "football player"};
console.log(person1.__proto__ === person.prototype); //true
```
> `__proto__`除了被主流浏览器支持外,还被Node.js支持,在ES2015进入到规范附录部分,算是被正式纳入了标准.

而在标准的语法里,实例对象是通过内置的内部属性`[[Prototype]]`来追踪原型对象的,这个`[[Prototype]]`的指针始终指向原型对象,此属性通常情况下是不可见的,我们需要用`getPrototypeOf()`来读取`[[Prototype]]`属性值(这个值就是原型对象).

```javascript
var obj = {};
console.log(Object.getPrototypeOf(obj) === Object.prototype); //true
```
同时我们也可以用`isPrototypeOf`来检验某个对象是否是另一个对象的原型对象.

```javascript
var obj = {};
console.log(Object.prototype.isPrototypeOf(obj)); //true
```

---
### ３．原型链

在我们了解了`__proto__`与`[[Prototype]]`之后,就可以相对容易理解原型链了,由于`__proto__`与`[[Prototype]]`功能相似,但是`__proto__`更容易测试方便学习,我们选择`__proto__`来进行原型链的讲解.

```javascript
function person(pname, page) {
  this.name = pname;
  this.age = page;
}
person.prototype.profession = "football player";
var person1 = new person("Messi", 29); //person1 = {name:"Messi", age: 29, profession: "football player"};
var person2 = new person("Bale", 28); //person2 = {name:"Bale", age: 28, profession: "football player"};
person1.hasOwnProperty("name");
console.log(person1.hasOwnProperty("hasOwnProperty")); //fasle
console.log(person1.__proto__ === person.prototype); //true
console.log(person.prototype.hasOwnProperty("hasOwnProperty")); //false
console.log(person1.__proto__.__proto__ === person.prototype.__proto__); // true
console.log(person.prototype.__proto__.hasOwnProperty("hasOwnProperty")); //true
```
我们可以分析这个例子,看看`person1`对象实例是如何调用`hasOwnProperty()`这个方法的.

1. 首先`person1`对象实例中寻找`hasOwnProperty()`方法,`person1.hasOwnProperty("hasOwnProperty")`返回`false`,发现不存在此方法,这时通过`__proto__`找到`person1`的原型对象.
2. 在`person1`的原型对象`person1.__proto__`即`person.prototype`中寻找`hasOwnProperty()`方法,`person.prototype.hasOwnProperty("hasOwnProperty")`返回`false`,依然没有找到,此时顺着`person.prototype`的`__proto__`找到其原型对象.
3. 在`person.prototype`原型对象`person.prototype.__proto__`即`Object.prototype`中寻找`hasOwnProperty()`方法,`Object.prototype.hasOwnProperty("hasOwnProperty")`返回`true`,由于`hasOwnProperty()`为`Object.prototype`内置方法,因此`person1`顺利找到此方法并调用.

总而言之,实例对象方法调用,是现在实力对象内部找,如果找到则立即返回调用,如果没有找到就顺着`__proto__`向上寻找,如果找到该方法则调用,没有找到会直接报错,这便是**原型链**.

如图所示,会更加直观.

---
#### 4.ES6中的`__proto__`

虽然`__proto__`在最新的ECMA标准中被纳入了规范,但是由于`__proto__`前后的双下划线，说明它本质上是一个内部属性，而不是一个正式的对外的 API.
标准明确规定，只有浏览器必须部署这个属性，其他运行环境不一定需要部署，而且新的代码最好认为这个属性是不存在的。因此，无论从语义的角度，还是从兼容性的角度，都不要使用这个属性，而是使用下面的`Object.setPrototypeOf()`（写操作）、`Object.getPrototypeOf()`（读操作）代替。

```javascript
function person(pname, page) {
  this.name = pname;
  this.age = page;
}
person.prototype.profession = "football player";
var person1 = new person("Messi", 29);

console.log(Object.getPrototypeOf(person1) === person.prototype); //true
Object.setPrototypeOf(person1, {League: "La Liga"}); 
console.log(person1.League); //La Liga

```
以上为具体用法,但是值得注意的是`Object.setPrototypeOf()`在使用中有一个坑,如代码所示:
```javascript
function person(pname, page) {
  this.name = pname;
  this.age = page;
}
person.prototype.profession = "football player";
var person1 = new person("Messi", 29); 
var person2 = new person("Bale", 28);

 Object.setPrototypeOf(person1, { League: "La Liga"});

console.log(person1.League); //La Liga
console.log(person2.League); //undefind
```
也就是说不同于直接用`person1.__proto__.League = "La Liga";`会使得两个实例同时生效,`Object.setPrototypeOf()`只能生效一个实例对象.

