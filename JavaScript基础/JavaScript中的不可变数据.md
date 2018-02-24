# JavaScript中的不可变数据结构

---

#### 前言
　　我们之前已经提到过,在JavaScript中分为原始类型和引用类型.
　　
> JavaScript原始类型:Undefined、Null、Boolean、Number、String、Symbol 

> JavaScript引用类型:Object

同时引用类型在使用过程中经常会产生副作用.

```javascript
const person = {player: {name: 'Messi'}};

const person1 = person;

console.log(person, person1);

//[ { name: 'Messi' } ] [ { name: 'Messi' } ]

person.player.name = 'Kane';

console.log(person, person1);
//[ { name: 'Kane' } ] [ { name: 'Kane' } ]
```

我们看到,当修改了`person`中属性后,`person1`的属性值也随之改变,这就是引用类型的副作用.

可是绝大多数情况下我们并不希望`person1`的属性值也发生改变,我们应该如何解决这个问题?

---

#### 1.浅复制

　　在ES6中我们可以用`Object.assign` 或者 `...`对引用类型进行浅复制.

```javascript
const person = [{name: 'Messi'}];
const person1 = person.map(item =>
  ({...item, name: 'Kane'})
)

console.log(person, person1);
// [{name: 'Messi'}] [{name: 'Kane'}]
```

`person`的确被成功复制了,但是之所以我们称它为浅复制,是因为这种复制只能复制一层,在多层嵌套的情况下依然会出现副作用.

```javascript
const person = [{name: 'Messi', info: {age: 30}}];
const person1 = person.map(item =>
  ({...item, name: 'Kane'})
)

console.log(person[0].info === person1[0].info); // true

```
上述代码表明当利用浅复制产生新的`person1`后其中嵌套的`info`属性依然与原始的`person`的`info`属性指向同一个堆内存对象,这种情况依然会产生副作用.

我们可以发现浅复制虽然可以解决浅层嵌套的问题,但是依然对多层嵌套的引用类型无能为力.




#### 2.深克隆

既然浅复制(克隆)无法解决这个问题,我们自然会想到利用深克隆的方法来实现多层嵌套复制的问题.

我们之前已经讨论过如何实现一个深克隆,在此我们不做深究,深克隆毫无疑问可以解决引用类型产生的副作用.

[如何实现深克隆](https://github.com/xiaomuzhu/ElemeFE-node-interview/blob/master/JavaScript%E5%9F%BA%E7%A1%80/javascript%E5%AE%9E%E7%8E%B0%E6%B7%B1%E5%85%8B%E9%9A%86.md)

实现一个在生产环境中可以用的深克隆是非常繁琐的事情,我们不仅要考虑到*正则*、*Symbol*、*Date*等特殊类型,还要考虑到*原型链*和*循环引用*的处理,当然我们可以选择使用成熟的[开源库](https://github.com/lodash/lodash/blob/master/cloneDeep.js)进行深克隆处理.

可是问题就在于我们实现一次深克隆的开销太昂贵了,[如何实现深克隆](https://github.com/xiaomuzhu/ElemeFE-node-interview/blob/master/JavaScript%E5%9F%BA%E7%A1%80/javascript%E5%AE%9E%E7%8E%B0%E6%B7%B1%E5%85%8B%E9%9A%86.md)中我们展示了一个勉强可以使用的深克隆函数已经处理了相当多的逻辑,如果我们每使用一次深克隆就需要一次如此昂贵的开销,程序的性能是会大打折扣.



```javascript
const person = [{name: 'Messi', info: {age: 30}}];

for (let i=0; i< 100000;i++) {
  person.push({name: 'Messi', info: {age: 30}});
}
console.time('clone');
const person1 = person.map(item =>
  ({...item, name: 'Kane'})
)
console.timeEnd('clone');
console.time('cloneDeep');
const person2 = lodash.cloneDeep(person)
console.timeEnd('cloneDeep');

// clone : 105.520ms
// cloneDeep : 372.839ms
```

我们可以看到深克隆的的性能相比于浅克隆大打折扣,然而浅克隆不能从根本上杜绝引用类型的副作用,我们需要找到一个兼具性能和效果的方案.

---
#### 3. immutable.js

immutable.js是正是兼顾了使用效果和性能的解决方案,它的解决方法是这样的.

因为对**Immutable**对象的任何修改或添加删除操作都会返回一个新的**Immutable**对象**Immutable**实现的原理是**Persistent Data Structur**（持久化数据结构），也就是使用旧数据创建新数据时，要保证旧数据同时可用且不变。同时为了避免 `deepCopy` 把所有节点都复制一遍带来的性能损耗，**Immutable** 使用了 **Structural Sharing**（结构共享），即如果对象树中一个节点发生变化，只修改这个节点和受它影响的父节点，其它节点则进行共享。请看下面动画

![](https://segmentfault.com/image?src=http://img.alicdn.com/tps/i2/TB1zzi_KXXXXXctXFXXbrb8OVXX-613-575.gif&objectId=1190000003910357&token=4f994e3bf65c373b010a157dfbab240f)

这个方法大大降低了cpu和内存的浪费

详细的immutable.js介绍请参考[Immutable 详解及 React 中实践](https://segmentfault.com/a/1190000003910357)

但是在使用过程中,immutable.js也存在很多问题.

我目前碰到的坑有:
1. 由于实现了完整的不可变数据,immutable.js的体积过于庞大,尤其在移动端这个情况被凸显出来.
2. 全新的api+不友好的文档,immutable.js使用的是自己的一套api,因此我们对js原生数组、对象的操作统统需要抛弃重新学习，但是官方文档不友好，很多情况下需要自己去试api.
3. 调试错误困难，immutable.js自成一体的数据结构,我们无法像读原生js一样读它的数据结构,很多情况下需要`toJS()`转化为原生数据结构再进行调试,这让人很崩溃.
![](http://omrbgpqyl.bkt.clouddn.com/18-2-21/25345459.jpg)


immutable.js在某种程度上来说,更适合于对数据可靠度要求颇高的大型前端应用(需要引入庞大的包、额外的学习成本甚至类型检测工具对付immutable.js与原生js类似的api)，中小型的项目引入immutable.js的代价有点高昂了,可是我们有时候不得不利用immutable的特性,那么如何保证性能和效果的情况下减少immutable相关库的体积和提高api友好度呢?


---
#### 4.如何实现更简单的immutable

我们的原则已经提到了,要尽可能得减小体积,这就注定了我们不能像immutable.js那样自己定义各种数据结构,而且要减小使用成本,所以要用原生js的方式,而不是自定义数据结构中的api.

这个时候需要我们思考如何实现上述要求呢?

我们要通过原生js的api来实现immutable,很显然我们需要对引用对象的set、get、delete等一系列操作的特性进行修改，这就需要`defineProperty`或者`Proxy`进行元编程.

我们就以`Proxy`为例来进行编码,当然,我们需要事先了解一下`Proxy`的[使用方法](http://es6.ruanyifeng.com/#docs/proxy#Proxy-revocable).

我们先定义一个目标对象
```javascript
const target = {name: 'Messi', age: 29};
```

我们如果想每访问一次这个对象的`age`属性,`age`属性的值就增加`1`.

```JavaScript
const target = {name: 'Messi', age: 29};
const handler = {
  get: function(target, key, receiver) {
    console.log(`getting ${key}!`);
    if (key === 'age') {
      const age = Reflect.get(target, key, receiver)
      Reflect.set(target, key, age+1, receiver);
      return age+1
    }
    return Reflect.get(target, key, receiver);
  }
};

const a = new Proxy(target, handler);

console.log(a.age, a.age);
//getting age!
//getting age!
//30 31
```

是的`Proxy`就像一个代理器,当有人对目标对象进行处理(set、has、get等等操作)的时候它会拦截操作，并用我们提供的代码进行处理，此时`Proxy`相当于一个中介或者叫代理人,当然`Proxy`的名字也说明了这一点,它经常被用于代理模式中,例如字段验证、缓存代理、访问控制等等。

我们的目的很简单，就是利用`Proxy`的特性，在外部对目标对象进行修改的时候来进行额外操作保证数据的不可变。

在外部对目标对象进行修改的时候,我们可以将被修改的引用的那部分进行拷贝,这样既能保证效率又能保证可靠性.

1. 那么如何判断目标对象是否被修改过,最好的方法是维护一个状态

```javascript
  function createState(target) {
    this.modified = false; // 是否被修改
    this.target = target; // 目标对象
    this.copy = undefined; // 拷贝的对象
  }
```

2. 此时我们就可以通过状态判断来进行不同的操作了

```JavaScript
createState.prototype = {
    // 对于get操作,如果目标对象没有被修改直接返回原对象,否则返回拷贝对象
    get: function(key) {
      if (!this.modified) return this.target[key];
      return this.copy[key];
    },
    // 对于set操作,如果目标对象没被修改那么进行修改操作,否则修改拷贝对象
    set: function(key, value) {
      if (!this.modified) this.markChanged();
      return (this.copy[key] = value);
    },

    // 标记状态为已修改,并拷贝
    markChanged: function() {
      if (!this.modified) {
        this.modified = true;
        this.copy = shallowCopy(this.target);
      }
    },
  };

  // 拷贝函数
  function shallowCopy(value) {
    if (Array.isArray(value)) return value.slice();
    if (value.__proto__ === undefined)
      return Object.assign(Object.create(null), value);
    return Object.assign({}, value);
  }
```
3. 最后我们就可以利用构造函数`createState`接受目标对象`state`生成对象`store`,然后我们就可以用`Proxy`代理`store`,`producer`是外部传进来的操作函数,当`producer`对代理对象进行操作的时候我们就可以通过事先设定好的`handler`进行代理操作了.

```JavaScript
  const PROXY_STATE = Symbol('proxy-state');
  const handler = {
    get(target, key) {
      if (key === PROXY_STATE) return target;
      return target.get(key);
    },
    set(target, key, value) {
      return target.set(key, value);
    },
  };

  // 接受一个目标对象和一个操作目标对象的函数
  function produce(state, producer) {
    const store = new createState(state);
    const proxy = new Proxy(store, handler);

    producer(proxy);

    const newState = proxy[PROXY_STATE];
    if (newState.modified) return newState.copy;
    return newState.target;
  }

```

4. 我们可以验证一下,我们看到`producer`并没有干扰到之前的目标函数.
```JavaScript
const baseState = [
  {
    todo: 'Learn typescript',
    done: true,
  },
  {
    todo: 'Try immer',
    done: false,
  },
];

const nextState = produce(baseState, draftState => {
  draftState.push({todo: 'Tweet about it', done: false});
  draftState[1].done = true;
});

console.log(baseState, nextState);
/*
[ { todo: 'Learn typescript', done: true },
  { todo: 'Try immer', done: true } ] 

  [ { todo: 'Learn typescript', done: true ,
  { todo: 'Try immer', done: true },
  { todo: 'Tweet about it', done: false } ]
*/
```

实际上这个实现就是不可变数据库[immer](https://github.com/mweststrate/immer)
的迷你版,我们阉割了大量的代码才缩小到了60行左右来实现这个基本功能,实际上除了`get/set`操作,这个库本身有`has/getOwnPropertyDescriptor/deleteProperty`等一系列的实现,我们由于篇幅的原因很多代码也十分粗糙,深入了解可以移步完整源码.

---
本文主要参考：
1. [immer](https://github.com/mweststrate/immer/blob/master/src/proxy.js)



