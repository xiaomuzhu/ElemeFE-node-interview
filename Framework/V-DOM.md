# 如何实现一个虚拟DOM

---

### 1. 虚拟DOM简述

DOM是什么,我们不需要多说,上古时代的前端工程师们就是靠DOM活着的,当初读着<DOM编程艺术>入门的人们眼里js就等于DOM,以至于当Node.js横空出世的时候,某些前端工程师在Node中调用DOM(出自justjavac在知乎上讲得著名案例).

现在大家都知道DOM是浏览器环境中才有的东西,是我们的前端开发最终还是离不开DOM的,可是为什么会有虚拟DOM出现,这就不得不提DOM的几个缺点:

>1.慢: 在现代浏览器运行的DOM本身并不慢,慢的是DOM被操作后产生的副作用(回流、重绘)

[DOM真的很慢吗？如果真的是，为什么不去改进它？](https://www.zhihu.com/question/30498056)

> 2.缺乏通用性: DOM是浏览器环境的产物,脱离了浏览器他就不复存在,所以在Node中用DOM是不可以的.

**虚拟DOM的出现正是用来解决上述问题.**

1. 虚拟DOM本质是JavaScript对象,他具有良好的通用性,正是因为有了虚拟DOM,vue和React才可以实现服务端渲染和Rn/Weex等移动技术.

2. 也是因为虚拟DOM是JavaScript对象,因此操作起来速度极快,通过diff算法找出差异,最后只需要将差异的部分批量操作DOM,避免我们频繁直接操作DOM而产生性能问题.

---

### 2. 实现虚拟DOM

虚拟DOM是一个思想,不同的库或者框架采用的不同的实现方法,但是本质上都是一样的,就是用js模拟DOM,我们先看一下真实的DOM有哪些属性.

![](http://omrbgpqyl.bkt.clouddn.com/18-3-12/85127059.jpg)

我把DOM的属性打印了出来,很明显DOM是一个十分**重**的对象，虚拟DOM相对而言就轻量了太多。

举个例子:

我们在React的JSX中写一段代码

```html
<ul className='todo'>
  <li>1</li>
  <li>2</li>
</ul>
```

实际上在执行过程中是这样的

```javascript
React.createElement(
  'ul',
  {className: 'todo'},
)
```

我们希望只用`ul` `{className: 'todo'}`等少数标识就能模拟一个DOM。

### 2.1 社区的主流实践

目前社区主流的`Virtual DOM`实现方法有三种：
1. 一种是先驱者React实现的虚拟dom，这种方法在React 16.x版本中已经被彻底重写为Fiber架构，因此我们就不过多探究
> Fiber架构是个大话题，借鉴了操作系统的调度方法，React团队花了一年打造，目前依然在改进Fiber。
2. 一种是[snabbdom](https://github.com/snabbdom/snabbdom)的实现方法，目前许多虚拟DOM的实现都是基于snabbdom，或许是最主流的实现方法，比较著名的就是Vue，它大量借鉴了snabbdom。
3. 另外一种是高性能的`Virtual DOM`，以Inferno.js为代表，它优化了diff算法和`Virtual DOM`的数据结构，使得性能大幅提高。

> 社区还有各种各样或复杂或简单的实现方法，其实大同小异，我们就不过多列举了。

我们先探究一下社区内最主流的虚拟DOM实现方法。

### 2.2 定义虚拟DOM的数据结构

> 我们的实现是基于snabbdom的简化版，省去了大量的类型判断和兼容性代码，只能作为学习使用。

真实DOM是一个对象，自然使用对象作为虚拟DOM的数据结构最为合理。

```JavaScript
/**
 * 生成 vnode
 * @param  {String} type     类型，如 'p' 'span'等
 * @param  {Object} data     data，包括属性，事件等等
 * @param  {Array} children  子 vnode
 * @param  {String} text     文本
 * @param  {Element} elm     对应的dom
 * @param  {String} key      唯一标识
 * @return {Object}          vnode
 */
function vnode(type, data, children, text, elm， key) {
  const element = {
    type, data, children, text, elm，key,
  }

  return element
}
```

虚拟DOM的模拟其实很简单，以上几行代码就能模拟一个虚拟DOM了，但是这才是刚刚开始，我们都知道真实DOM是以树的形式呈现的，我们需要一个函数，帮助我们构造虚拟DOM树。

我们需要类似于`createElement`的函数,利用`ul`等参数形成虚拟DOM.

```javascript
React.createElement(
  'ul',
  {className: 'todo'},
)
```

我们实现一个h函数，接收一个`type`代表标签名称（例如‘div’等），接收一个`config`代表属性（例如样式属性等），接收一个`children`代表子节点。

```JavaScript

// 判断是否是number 或者 string
isPrimitive = val => {
  return typeof val === 'number' || typeof val === 'string'
}

function h(type, config, ...children) {
  // 建立空对象,作为虚拟dom属性
  const props = {}

  // 默认key为null
  let key = null

  // 获取 key，对props赋值
  if (config) {
    if (config.key) {
      key = config.key
    }

    for (let propName in config) {
      if (!config.hasOwnProperty(propName)) {
        props[propName] = config[propName]
      }
    }
  }
  // 成功转化为虚拟DOM
  return vnode(
    type,
    props,
    flattenArray(children).map(c => {
      return isPrimitive(c) ? vnode(undefined, undefined, undefined, c, undefined) : c
    }),
    key,
  )
}
```

### 2.3 diff算法的可行性

现在我们梳理一下逻辑，以React为例，我们平时在书写的是jsx，它类似于HTML和JavaScript的混写，但是真正在执行过程中是事先需要babel将其转换成`React.createElement`的形式,当然我们目前已经实现了一个类似于此方法的`h`函数,但是真正要我们书写的jsx中的代码转换成真实DOM的过程我们还没有涉及，我们仅仅实现了jsx到虚拟DOM的过程，那么接下来我们需要完成剩余的工作。

我们都知道虚拟DOM可以以接近最小的代价来局部刷新页面，也就是所谓的**只更改有变化的地方**，这就不得不引入diff算法。

> diff算法是计算机世界最常用的算法之一，例如unix系统中的diff命令，git中的git diff都是对diff的典型应用，用于比较两个文本、文件、数据的不同。

可是普通的diff算法并不适用于虚拟DOM树的对比，因为两棵树的对比时间复杂度为 O(n^3)，看到这里大家心里已经凉了，这个时间复杂度其实就意味着diff是极度低效的，如果虚拟DOM采用这个算法，那么这个架构属于画蛇添足之举。

所幸的是，我们在实际项目中基本不存在夸层级之间的DOM操作，因此我们只需要对同层级的虚拟DOM节点进行diff即可，我们的时间复杂度仅仅为O(n)。
![](http://omrbgpqyl.bkt.clouddn.com/18-3-31/55658635.jpg)

### 2.4 diff算法的实现

```JavaScript
function updateChildren(parentElm, oldCh, newCh, insertedVnodeQueue) {
  let oldStartIdx = 0, newStartIdx = 0
  let oldEndIdx = oldCh.length - 1
  let oldStartVnode = oldCh[0]
  let oldEndVnode = oldCh[oldEndIdx]
  let newEndIdx = newCh.length - 1
  let newStartVnode = newCh[0]
  let newEndVnode = newCh[newEndIdx]
  let oldKeyToIdx
  let idxInOld
  let elmToMove
  let before

  // 遍历 oldCh 和 newCh 来比较和更新
  while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
    // 1⃣️ 首先检查 4 种情况，保证 oldStart/oldEnd/newStart/newEnd
    // 这 4 个 vnode 非空，左侧的 vnode 为空就右移下标，右侧的 vnode 为空就左移 下标。
    if (oldStartVnode == null) {
      oldStartVnode = oldCh[++oldStartIdx]
    } else if (oldEndVnode == null) {
      oldEndVnode = oldCh[--oldEndIdx]
    } else if (newStartVnode == null) {
      newStartVnode = newCh[++newStartIdx]
    } else if (newEndVnode == null) {
      newEndVnode = newCh[--newEndIdx]
    }
    /**
     * 2⃣️ 然后 oldStartVnode/oldEndVnode/newStartVnode/newEndVnode 两两比较，
     * 对有相同 vnode 的 4 种情况执行对应的 patch 逻辑。
     * - 如果同 start 或同 end 的两个 vnode 是相同的（情况 1 和 2），
     *   说明不用移动实际 dom，直接更新 dom 属性／children 即可；
     * - 如果 start 和 end 两个 vnode 相同（情况 3 和 4），
     *   那说明发生了 vnode 的移动，同理我们也要移动 dom。
     */
    // 1. 如果 oldStartVnode 和 newStartVnode 相同（key相同），执行 patch
    else if (isSameVnode(oldStartVnode, newStartVnode)) {
      // 不需要移动 dom
      patchVnode(oldStartVnode, newStartVnode, insertedVnodeQueue)
      oldStartVnode = oldCh[++oldStartIdx]
      newStartVnode = newCh[++newStartIdx]
    }
    // 2. 如果 oldEndVnode 和 newEndVnode 相同，执行 patch
    else if (isSameVnode(oldEndVnode, newEndVnode)) {
      // 不需要移动 dom
      patchVnode(oldEndVnode, newEndVnode, insertedVnodeQueue)
      oldEndVnode = oldCh[--oldEndIdx]
      newEndVnode = newCh[--newEndIdx]
    }
    // 3. 如果 oldStartVnode 和 newEndVnode 相同，执行 patch
    else if (isSameVnode(oldStartVnode, newEndVnode)) {
      patchVnode(oldStartVnode, newEndVnode, insertedVnodeQueue)
      // 把获得更新后的 (oldStartVnode/newEndVnode) 的 dom 右移，移动到
      // oldEndVnode 对应的 dom 的右边。为什么这么右移？
      // （1）oldStartVnode 和 newEndVnode 相同，显然是 vnode 右移了。
      // （2）若 while 循环刚开始，那移到 oldEndVnode.elm 右边就是最右边，是合理的；
      // （3）若循环不是刚开始，因为比较过程是两头向中间，那么两头的 dom 的位置已经是
      //     合理的了，移动到 oldEndVnode.elm 右边是正确的位置；
      // （4）记住，oldVnode 和 vnode 是相同的才 patch，且 oldVnode 自己对应的 dom
      //     总是已经存在的，vnode 的 dom 是不存在的，直接复用 oldVnode 对应的 dom。
      api.insertBefore(parentElm, oldStartVnode.elm, api.nextSibling(oldEndVnode.elm))
      oldStartVnode = oldCh[++oldStartIdx]
      newEndVnode = newCh[--newEndIdx]
    }
    // 4. 如果 oldEndVnode 和 newStartVnode 相同，执行 patch
    else if (isSameVnode(oldEndVnode, newStartVnode)) {
      patchVnode(oldEndVnode, newStartVnode, insertedVnodeQueue)
      // 这里是左移更新后的 dom，原因参考上面的右移。
      api.insertBefore(parentElm, oldEndVnode.elm, oldStartVnode.elm)
      oldEndVnode = oldCh[--oldEndIdx]
      newStartVnode = newCh[++newStartIdx]
    }

    // 3⃣️ 最后一种情况：4 个 vnode 都不相同，那么我们就要
    // 1. 从 oldCh 数组建立 key --> index 的 map。
    // 2. 只处理 newStartVnode （简化逻辑，有循环我们最终还是会处理到所有 vnode），
    //    以它的 key 从上面的 map 里拿到 index；
    // 3. 如果 index 存在，那么说明有对应的 old vnode，patch 就好了；
    // 4. 如果 index 不存在，那么说明 newStartVnode 是全新的 vnode，直接
    //    创建对应的 dom 并插入。
    else {
      // 如果 oldKeyToIdx 不存在，创建 old children 中 vnode 的 key 到 index 的
      // 映射，方便我们之后通过 key 去拿下标。
      if (oldKeyToIdx === undefined) {
        oldKeyToIdx = createKeyToOldIdx(oldCh, oldStartIdx, oldEndIdx)
      }
      // 尝试通过 newStartVnode 的 key 去拿下标
      idxInOld = oldKeyToIdx[newStartVnode.key]
      // 下标不存在，说明 newStartVnode 是全新的 vnode。
      if (idxInOld == null) {
        // 那么为 newStartVnode 创建 dom 并插入到 oldStartVnode.elm 的前面。
        api.insertBefore(parentElm, createElm(newStartVnode, insertedVnodeQueue), oldStartVnode.elm)
        newStartVnode = newCh[++newStartIdx]
      }
      // 下标存在，说明 old children 中有相同 key 的 vnode，
      else {
        elmToMove = oldCh[idxInOld]
        // 如果 type 不同，没办法，只能创建新 dom；
        if (elmToMove.type !== newStartVnode.type) {
          api.insertBefore(parentElm, createElm(newStartVnode, insertedVnodeQueue), oldStartVnode.elm)
        }
        // type 相同（且key相同），那么说明是相同的 vnode，执行 patch。
        else {
          patchVnode(elmToMove, newStartVnode, insertedVnodeQueue)
          oldCh[idxInOld] = undefined
          api.insertBefore(parentElm, elmToMove.elm, oldStartVnode.elm)
        }
        newStartVnode = newCh[++newStartIdx]
      }
    }
  }

  // 上面的循环结束后（循环条件有两个），处理可能的未处理到的 vnode。
  // 如果是 new vnodes 里有未处理的（oldStartIdx > oldEndIdx
  // 说明 old vnodes 先处理完毕）
  if (oldStartIdx > oldEndIdx) {
    before = newCh[newEndIdx+1] == null ? null : newCh[newEndIdx+1].elm
    addVnodes(parentElm, before, newCh, newStartIdx, newEndIdx, insertedVnodeQueue)
  }
  // 相反，如果 old vnodes 有未处理的，删除 （为处理 vnodes 对应的） 多余的 dom。
  else if (newStartIdx > newEndIdx) {
    removeVnodes(parentElm, oldCh, oldStartIdx, oldEndIdx)
  }
}
```