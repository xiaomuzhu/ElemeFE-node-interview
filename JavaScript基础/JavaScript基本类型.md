# 你可能真的不懂JavaScript中最基础类型

---

#### 前言
　　众所周知,JavaScript是动态弱类型的多范式编程语言,由于设计时的粗糙(当时设计js的初衷就是在浏览器中处理表单这种简单事件)导致JavaScript在许多方面表现出了这样或者那样的问题,其中'类型'便是语法层面最常见的'埋坑'重灾区.
　　
> JavaScript原始类型:Undefined、Null、Boolean、Number、String、Symbol 
JavaScript引用类型:Object 

---

#### 1.原始类型与引用类型

###### 1.1
　　**原始类型**又被称为**基本类型**，原始类型保存的变量和值直接保存在**栈内存**(Stack)中,且空间相互独立,通过值来访问,说到这里肯定一同懵逼,不过我们可以通过一个例子来解释.

```
var person = 'Messi';
var person1 = person;
```
上述代码在栈内存的示意图是这样的,可以看到,虽然`person`赋值给了`person1`.但是两个变量并没有指向同一个值,而是`person1`自己单独建立一个内存空间,虽然两个变量的值相等,但却是相互独立的.
![](http://p1.bqimg.com/567571/11993c9e172c8684.png)

```
var person = 'Messi';
var person1 = person;

var person = 1;

console.log(person); //1
console.log(person1); //'Messi'

```
上述代码示意图是这样的,`person`的值虽然改变,但是由于`person1`的值是独立储存的,因此不受影响.
![](http://p1.bqimg.com/567571/72d29c309b1d75ce.png)

值得一提的是,虽然原始类型的值是储存在相对独立空间,但是它们之间的比较是**按值**比较的.

```
var person = 'Messi';
var person1 = 'Messi';
console.log(person === person1); //true
```


###### 1.2引用类型

剩下的就是引用类型了,即Object 类型,再往下细分，还可以分为：Object 类型、Array 类型、Date 类型、Function 类型 等。

与原始类型不同的是,引用类型的内容是保存在**堆内存**中,而**栈内存**(Heap)中会有一个**堆内存地址**,通过这个地址变量被指向堆内存中`Object`真正的值,因此引用类型是按照引用访问的.
![](http://p1.bpimg.com/567571/ade18e93a9f9e9cb.png)

由于示意图太难画,我从网上找了一个例子,能很清楚的说明引用类型的特质.

```
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
    3.最后创建了一个新的对象`c`,看似跟`b` `a`一样,但是在堆内存中确实两个相互独立的`Object`,引用类型是按照引用比较,由于`a` `c`引用的是不同的`Object`所以得到的结果是`fasle`.
![](http://i1.piimg.com/567571/86af0de8deea6301.png)



---
#### 2. 类型中的坑

2.1 数组中的坑
数组是JavaScript中最常见的类型之一了,但是在我们实践过程中同样会遇到各种各样的麻烦.

**稀疏数组**:指的是含有空白或空缺单元的数组
```
var a = [];

console.log(a.length); //0

a[4] = a[5];

console.log(a.length); //5

a.forEach(elem => {
  console.log(elem); //undefind
});

console.log(a); //[,,,,undefind]
```
这里有几个坑需要注意:
1. 一开始建立的空数组`a`的长度为0,这可以理解,但是在`a[4] = a[5]`之后出现了问题,`a`的长度居然变成了5,此时`a`数组是`[,,,,undefind]`这种形态.
2. 我们通过遍历,只得到了`undefind`这一个值,这个`undefind`是由于`a[4] = a[5]`赋值,由于`a[5]`没有定义值为`undefind`被赋给了`a[4]`,可以等价为`a[4] = undefind`.

**字符串索引**

```
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
```
var a = 0.1 + 0.2;
var b = 0.3;
console.log(a === b); //false
```
这是个出人意料的结果,实际上a的值约为`0.30000000000000004`这并不是一个整数值,这就是`二进制浮点数`带来的副作用.

```
var a = 0.1 + 0.2;
var b = 0.3;
console.log(a === b); //false
console.log(Number.isInteger(a*10)); //false
console.log(Number.isInteger(b*10)); //true
console.log(a); //0.30000000000000004
```
我们可以用`Number.isInteger()`来判断一个数字是否为整数.

**NaN**

```
var a = 1/new Object();
console.log(typeof a); //Number
console.log(a); //NaN
console.log(isNaN(a)); //true
```
`NaN`属于特殊的`Number`类型,我们可以把它理解为`坏数值`,因为它属于数值计算中的错误,更加特殊的是它自己都不等价于自己`NaN === NaN //false`,我们只能用`isNaN()`来检测一个数字是否为`NaN`.



---
#### 3.类型转换原理

**类型转换**指的是将一种类型转换为另一种类型,例如:
```
var b = 2;
var a = String(b);
console.log(typeof a); //string
```

当然,**类型转换**分为显式和隐式,但是不管是隐式转换还是显式转换,都会遵循一定的原理,由于JavaScript是一门动态类型的语言,可以随时赋予任意值,但是各种运算符或条件判断中是需要特定类型的,因此JavaScript引擎会在运算时为变量设定类型.

这看起来很美好,JavaScript引擎帮我们搞定了`类型`的问题,但是引擎毕竟不是ASI(超级人工智能),它的很多动作会跟我们预期相去甚远,我们可以从一到面试题开始.

```
{}+[] //0
```


答案是0,下图是在`node 7.7.2`版本测试下的结果,浏览器测试结果同上.
![](http://p1.bqimg.com/567571/8bd81ae2ea2f3921.png)

是什么原因造成了上述结果呢?那么我们得从ECMA-262中提到的转换规则和抽象操作说起,有兴趣的童鞋可以仔细阅读下这浩如烟海的[语言规范](http://ecma-international.org/ecma-262/5.1/),如果没这个耐心还是往下看.

这是JavaScript种类型转换可以从**原始类型**转为**引用类型**,同样可以将**引用类型**转为**原始类型**,转为原始类型的抽象操作为`ToPrimitive`,而后续更加细分的操作为:`ToNumber ToString ToBoolean`,这三种抽象操作的转换表如下所示
|值 |ToNumber|ToString |ToBoolean|
|:---:|:---:|:---:|:---:|
|undefined|	NaN|“undefined”|false|
|null|	0|	“null”|	false|
|true|	1|	“true”|	 
|false|	0|	“false”	| 
|0	 	|“0”	|false|
|-0	 |	“0”|	false|
|NaN	 |	“NaN”|	false|
|Infinity|	 	|“Infinity”|	true|
|-Infinity|	 	|”-Infinity”|	true|
|1(非零)|	 |	“1”	|true|
|`[ ](空数组)`|	0|	””|	true|
|`[9](包含一个数字元素)`|	9|	“9”|	true|


如果想应付面试,我觉得这张表就差不多了,但是为了更深入的探究JavaScript引擎是如何处理代码中类型转换问题的,就需要看 ECMA-262详细的规范,从而探究其内部原理,我们从这段内部原理示意代码开始.
```
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









---



---



---




