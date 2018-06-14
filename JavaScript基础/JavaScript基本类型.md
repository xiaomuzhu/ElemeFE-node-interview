# 你可能真的不懂JavaScript中最基础类型

---

#### 前言
　　众所周知,JavaScript是动态弱类型的多范式编程语言,由于设计时的粗糙(当时设计js的初衷就是在浏览器中处理表单这种简单事件)导致JavaScript在许多方面表现出了这样或者那样的问题,其中'类型'便是语法层面最常见的'埋坑'重灾区.
　　
> JavaScript原始类型:Undefined、Null、Boolean、Number、String、Symbol 

> JavaScript引用类型:Object 

---

#### 1.原始类型与引用类型

###### 1.1
　　**原始类型**又被称为**基本类型**，原始类型保存的变量和值直接保存在**栈内存**(Stack)中,且空间相互独立,通过值来访问,说到这里肯定一同懵逼,不过我们可以通过一个例子来解释.

```javascript
var person = 'Messi';
var person1 = person;
```
上述代码在栈内存的示意图是这样的,可以看到,虽然`person`赋值给了`person1`.但是两个变量并没有指向同一个值,而是`person1`自己单独建立一个内存空间,虽然两个变量的值相等,但却是相互独立的.

![](http://omrbgpqyl.bkt.clouddn.com/18-2-20/84377115.jpg)

```javascript
var person = 'Messi';
var person1 = person;

var person = 1;

console.log(person); //1
console.log(person1); //'Messi'

```
上述代码示意图是这样的,`person`的值虽然改变,但是由于`person1`的值是独立储存的,因此不受影响.

值得一提的是,虽然原始类型的值是储存在相对独立空间,但是它们之间的比较是**按值**比较的.

```javascript
var person = 'Messi';
var person1 = 'Messi';
console.log(person === person1); //true
```


###### 1.2引用类型

剩下的就是引用类型了,即Object 类型,再往下细分，还可以分为：Object 类型、Array 类型、Date 类型、Function 类型 等。

与原始类型不同的是,引用类型的内容是保存在**堆内存**(Heap)中,而**栈内存**(Stack)中会有一个**堆内存地址**,通过这个地址变量被指向堆内存中`Object`真正的值,因此引用类型是按照引用访问的.

由于示意图太难画,我从网上找了一个例子,能很清楚的说明引用类型的特质.

```javascript
var a = {name:"percy"};
var b;
b = a;
a.name = "zyj";
console.log(b.name);    // zyj
b.age = 22;
console.log(a.age);     // 22
var c = {
  name: "zyj",
  age: 22
};
console.log(a === c); //false

```

我们可以逐行分析:
    1. `b = a`,如果是原始类型的话,`b`会在栈内自己独自创建一个内存空间保存值,但是引用类型只是`b`的产生一个对内存地址,指向堆内存中的`Object`.
    2.`a.name = "zyj"`,这个操作属于改变了变量的值,在原始类型中会重新建立新的内存空间(可以看上一节的示意图),而引用类型只需要自己在堆内存中更新自己的属性即可.
    3.最后创建了一个新的对象`c`,看似跟`b` `a`一样,但是在堆内存中确实两个相互独立的`Object`,引用类型是按照**引用比较**,由于`a` `c`引用的是不同的`Object`所以得到的结果是`fasle`.  

![](http://omrbgpqyl.bkt.clouddn.com/18-2-20/34304948.jpg)
---
#### 2. 类型中的坑

2.1 数组中的坑
数组是JavaScript中最常见的类型之一了,但是在我们实践过程中同样会遇到各种各样的麻烦.

**稀疏数组**:指的是含有空白或空缺单元的数组
```javascript
var a = [];

console.log(a.length); //0

a[4] = a[5];

console.log(a.length); //5

a.forEach(elem => {
  console.log(elem); //undefined
});

console.log(a); //[,,,,undefined]
```
这里有几个坑需要注意:
1. 一开始建立的空数组`a`的长度为0,这可以理解,但是在`a[4] = a[5]`之后出现了问题,`a`的长度居然变成了5,此时`a`数组是`[,,,,undefined]`这种形态.
2. 我们通过遍历,只得到了`undefined`这一个值,这个`undefind`是由于`a[4] = a[5]`赋值,由于`a[5]`没有定义值为`undefined`被赋给了`a[4]`,可以等价为`a[4] = undefined`.

**字符串索引**

```javascript
var a = [];
a[0] = 'Bale';
a['age'] = 28;
console.log(a.length); //1
console.log(a['age']); //28
console.log(a); //[ 'Bale', age: 28 ]
```
数组不仅可以通过数字索引,也可以通过字符串索引,但值得注意的是,字符串索引的键值对并不算在数组的长度里.


2.2 数字中的坑
**二进制浮点数**

JavaScript 中的数字类型是基于“二进制浮点数”实现的,使用的是“双精度”格式,这就带来了一些反常的问题,我们那一道经典面试提来讲解下.
```javascript
var a = 0.1 + 0.2;
var b = 0.3;
console.log(a === b); //false
```
这是个出人意料的结果,实际上a的值约为`0.30000000000000004`这并不是一个整数值,这就是`二进制浮点数`带来的副作用.

```javascript
var a = 0.1 + 0.2;
var b = 0.3;
console.log(a === b); //false
console.log(Number.isInteger(a*10)); //false
console.log(Number.isInteger(b*10)); //true
console.log(a); //0.30000000000000004
```
我们可以用`Number.isInteger()`来判断一个数字是否为整数.

**NaN**

```javascript
var a = 1/new Object();
console.log(typeof a); //Number
console.log(a); //NaN
console.log(isNaN(a)); //true
```
`NaN`属于特殊的`Number`类型,我们可以把它理解为`坏数值`,因为它属于数值计算中的错误,更加特殊的是它自己都不等价于自己`NaN === NaN //false`,我们只能用`isNaN()`来检测一个数字是否为`NaN`.



---
#### 3.类型转换原理

**类型转换**指的是将一种类型转换为另一种类型,例如:
```javascript
var b = 2;
var a = String(b);
console.log(typeof a); //string
```

当然,**类型转换**分为显式和隐式,但是不管是隐式转换还是显式转换,都会遵循一定的原理,由于JavaScript是一门动态类型的语言,可以随时赋予任意值,但是各种运算符或条件判断中是需要特定类型的,因此JavaScript引擎会在运算时为变量设定类型.

这看起来很美好,JavaScript引擎帮我们搞定了`类型`的问题,但是引擎毕竟不是ASI(超级人工智能),它的很多动作会跟我们预期相去甚远,我们可以从一到面试题开始.

```javascript
{}+[] //0
```


答案是0

是什么原因造成了上述结果呢?那么我们得从ECMA-262中提到的转换规则和抽象操作说起,有兴趣的童鞋可以仔细阅读下这浩如烟海的[语言规范](http://ecma-international.org/ecma-262/5.1/),如果没这个耐心还是往下看.

这是JavaScript种类型转换可以从**原始类型**转为**引用类型**,同样可以将**引用类型**转为**原始类型**,转为原始类型的抽象操作为`ToPrimitive`,而后续更加细分的操作为:`ToNumber ToString ToBoolean`,这三种抽象操作的转换表如下所示
![](http://omrbgpqyl.bkt.clouddn.com/17-9-13/15517231.jpg)


如果想应付面试,我觉得这张表就差不多了,但是为了更深入的探究JavaScript引擎是如何处理代码中类型转换问题的,就需要看 ECMA-262详细的规范,从而探究其内部原理,我们从这段内部原理示意代码开始.
```javascript
// ECMA-262, section 9.1, page 30. Use null/undefined for no hint,
// (1) for number hint, and (2) for string hint.
function ToPrimitive(x, hint) {  
  // Fast case check.
  if (IS_STRING(x)) return x;
  // Normal behavior.
  if (!IS_SPEC_OBJECT(x)) return x;
  if (IS_SYMBOL_WRAPPER(x)) throw MakeTypeError(kSymbolToPrimitive);
  if (hint == NO_HINT) hint = (IS_DATE(x)) ? STRING_HINT : NUMBER_HINT;
  return (hint == NUMBER_HINT) ? DefaultNumber(x) : DefaultString(x);
}

// ECMA-262, section 8.6.2.6, page 28.
function DefaultNumber(x) {  
  if (!IS_SYMBOL_WRAPPER(x)) {
    var valueOf = x.valueOf;
    if (IS_SPEC_FUNCTION(valueOf)) {
      var v = %_CallFunction(x, valueOf);
      if (IsPrimitive(v)) return v;
    }

    var toString = x.toString;
    if (IS_SPEC_FUNCTION(toString)) {
      var s = %_CallFunction(x, toString);
      if (IsPrimitive(s)) return s;
    }
  }
  throw MakeTypeError(kCannotConvertToPrimitive);
}

// ECMA-262, section 8.6.2.6, page 28.
function DefaultString(x) {  
  if (!IS_SYMBOL_WRAPPER(x)) {
    var toString = x.toString;
    if (IS_SPEC_FUNCTION(toString)) {
      var s = %_CallFunction(x, toString);
      if (IsPrimitive(s)) return s;
    }

    var valueOf = x.valueOf;
    if (IS_SPEC_FUNCTION(valueOf)) {
      var v = %_CallFunction(x, valueOf);
      if (IsPrimitive(v)) return v;
    }
  }
  throw MakeTypeError(kCannotConvertToPrimitive);
}
```
上面代码的逻辑是这样的：

1. 如果变量为字符串，直接返回.
2. 如果`!IS_SPEC_OBJECT(x)`，直接返回.
3. 如果`IS_SYMBOL_WRAPPER(x)`，则抛出异常.
4. 否则会根据传入的`hint`来调用`DefaultNumber`和`DefaultString`，比如如果为`Date`对象，会调用`DefaultString`.
5. `DefaultNumber`：首`先x.valueOf`，如果为`primitive`，则返回`valueOf`后的值，否则继续调用`x.toString`，如果为`primitive`，则返回`toString`后的值，否则抛出异常
6. `DefaultString`：和`DefaultNumber`正好相反，先调用`toString`，如果不是`primitive`再调用`valueOf`.

那讲了实现原理，这个`ToPrimitive`有什么用呢？实际很多操作会调用`ToPrimitive`，比如加、相等或比较操。在进行加操作时会将左右操作数转换为`primitive`，然后进行相加。

下面来个实例，({}) + 1（将{}放在括号中是为了内核将其认为一个代码块）会输出啥？可能日常写代码并不会这样写，不过网上出过类似的面试题。

加操作只有左右运算符同时为`String或Number`时会执行对应的`%_StringAdd或%NumberAdd`，下面看下`({}) + 1`内部会经过哪些步骤：

`{}`和`1`首先会调用ToPrimitive
`{}`会走到`DefaultNumber`，首先会调用`valueOf`，返回的是`Object` `{}`，不是primitive类型，从而继续走到`toString`，返回`[object Object]`，是`String`类型
最后加操作，结果为`[object Object]1`
再比如有人问你`[] + 1`输出啥时，你可能知道应该怎么去计算了，先对`[]`调用`ToPrimitive`，返回空字符串，最后结果为"1"。


---
本文主要参考：
1. [JavaScript 类型的那些事](https://segmentfault.com/a/1190000010352325)



