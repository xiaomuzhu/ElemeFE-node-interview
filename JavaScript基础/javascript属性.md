# JavaScript面向对象之属性


---
#### 1.JavaScript中的对象


JavaScript中的对象一般分为三类:**内置对象**(Array, Error, Date等), **宿主对象**(对于前端来说指的是浏览器对象,例如window), **自定义对象**(指我们自己创建的对象).

因此,我们主要讨论的内容是围绕自定义对象展开的,今天我们就对象的属性进行深入地探究.

---

#### 2.属性的创建 
我们先定义一个对象,然后对其赋值:
```javascript
var person = {};
person.name = "Messi";

```
以上操作相当于给`person`对象建立了一个`name`属性,且值为`'Messi'`.

那么这个赋值的过程具体的原理是什么呢?

首先,我们创建了一个'空'对象,之所以我们打上引号,是因为这并不是一个严格意义上的空对象,因为在建立这个对象的过程中,JavaScript已经为这个对象内置了方法和属性,当然是不可见的,在属性的建立过程中就调用了一个隐式的方法`[[put]]`.

大概的创建过程是,当属性第一次被创建时,对象调用内部方法`[[put]]`为对象创建一个节点保存属性.
```flow
st=>start: person
e=>end: persen.name = "Messi"
io1=>inputoutput: [[put]]

st->io1->e
```


---
#### 3.属性的修改
我们对上例中的代码做一下修改:
```javascript
var person = {};
person.name = "Messi";
person.name = "Bale";
```
很显然,`name`被创建后,该属就被进行了修改,原属性值`Messi`被修改为`Bale`,那么这个过程又是如何发生的呢?

其实对象内部除了隐式的`[[put]]`方法,还有一个`[[set]]`方法,这个方法不同于`[[put]]`在创建属性时调用,而是在同一个属性被再次赋值的时候用于更新属性进行的调用.

```flow
st=>start: person.name = "Messi"
e=>end: persen.name = "Bale"
io1=>inputoutput: [[put]]

st->io1->e
```

---
#### 4.属性的查询
判断一个属性或者方法是否在一个对象中,通常有两种方式.
`in`操作符方式:
```javascript
var person = {
    name: "Messi"
};
console.log("name" in person); //true 
```
`hasOwnProperty`方法:

```javascript
var person = {
    name: "Messi"
};
console.log(person.hasOwnProperty("name")); //true
```

---

#### 5.属性的删除
删除一个属性,最正确的方式是用`delete`方法,一个错误的方式是将该属性赋值为`null`,该方式的错误之处在于赋值`null`相当于调用了[[set]]方法把原属性值更改为了`null`,这个保存属性的节点依然存在,而用`delete`方法便能彻底删除这个节点.
```javascript
var person = {
    name: "Messi"
};
delete person.name;
console.log("name" in person); //false
```

#### 6.属性的枚举

我们通常用`for...in`枚举对象中的属性,它会将属性一一返回.
在ES5中引入了一个新的方法`Object.key()`,不同之处在于,它可以将结果以数组的形式返回
```javascript
var person = {
    name: "Messi",
    age: 29
};

for(var pros in person ) {
    console.log(pros); // name age 
}

var pros = Object.keys(person);
console.log(pros); //[ 'name', 'age' ]

```

> 值得注意的是,并非所有的属性都是可枚举的,例如对象自带的属性`length`等等,因此我们可以用`propertyIsEnumerable()`方法来判断一个属性是否可枚举.














