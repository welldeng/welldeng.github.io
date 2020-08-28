---
layout: /archives
title: 学习vue源码—vue-diff
date: 2019-06-23 22:40:48
tags: Vue
categories:
  - JavaScript
  - Vue
---
> 本文主要记录vue-diff的原理以及说明一个响应式数据更新的流程是怎么样的一个过程。

## 1. 数据改变到页面渲染的过程是怎么样的？

首先看下面的图片👇，这是执行click函数改变一个数据之后发生的函数调用栈，从图上的说明可以比较清楚个了解这个响应式过程的大概流程。下面简单讲解一下：

1. 改变数据，触发这个被劫持过的数据的<code>setter</code>方法
2. 执行这个数据的订阅中心（<code>dep</code>）的<code>notify</code>方法
3. <code>update</code>方法里执行<code>queueWatcher</code>方法把<code>watcher</code>推入队列
4. 执行<code>nextTick</code>方法开始更新视图
5. <code>run</code>方法里设置<code>dep.target</code>为当前订阅对象
6. 调用<code>get</code>方法调用当前<code>watcher</code>的<code>getter</code>执行更新方法
7. <code>updateComponent</code>方法里调用了<code>render</code>方法开始执行渲染页面
8. <code>patch</code>、<code>patchVnode</code>、<code>updateChildren</code>方法都是比较VNode更新渲染的函数，不过重点的diff过程在<code>updateChildren</code>方法里。
   ![](https://user-gold-cdn.xitu.io/2019/6/23/16b83a2e38faea15?w=1016&h=1066&f=png&s=181273)

## 2. vue-diff的具体实现

<code>patchVnode</code>、<code>updateChildren</code>方法在vue源码项目的<code>src/core/vdom/patch.js</code>文件中。

先介绍<code>patchVnode</code>方法，这是执行真正更新dom的方法，大概的执行逻辑如下

1. 判断vnode和oldVnode是否相等
2. 判断是否能重用vnode
3. 判断是否执行回调
4. 判断是否有children需要diff更新
5. 判断执行更新类型—新增dom、移除dom、更新textDom

```
  function patchVnode (
    oldVnode,
    vnode,
    insertedVnodeQueue,
    ownerArray,
    index,
    removeOnly
  ) {
    if (oldVnode === vnode) {
      return
    }

    if (isDef(vnode.elm) && isDef(ownerArray)) {
      // clone reused vnode
      vnode = ownerArray[index] = cloneVNode(vnode)
    }

    const elm = vnode.elm = oldVnode.elm

    if (isTrue(oldVnode.isAsyncPlaceholder)) {
      if (isDef(vnode.asyncFactory.resolved)) {
        hydrate(oldVnode.elm, vnode, insertedVnodeQueue)
      } else {
        vnode.isAsyncPlaceholder = true
      }
      return
    }

    // reuse element for static trees.
    // note we only do this if the vnode is cloned -
    // if the new node is not cloned it means the render functions have been
    // reset by the hot-reload-api and we need to do a proper re-render.
    if (isTrue(vnode.isStatic) &&
      isTrue(oldVnode.isStatic) &&
      vnode.key === oldVnode.key &&
      (isTrue(vnode.isCloned) || isTrue(vnode.isOnce))
    ) {
      vnode.componentInstance = oldVnode.componentInstance
      return
    }

    let i
    const data = vnode.data
    if (isDef(data) && isDef(i = data.hook) && isDef(i = i.prepatch)) {
      i(oldVnode, vnode)
    }

    const oldCh = oldVnode.children
    const ch = vnode.children
    if (isDef(data) && isPatchable(vnode)) {
      for (i = 0; i < cbs.update.length; ++i) cbs.update[i](oldVnode, vnode)
      if (isDef(i = data.hook) && isDef(i = i.update)) i(oldVnode, vnode)
    }
    if (isUndef(vnode.text)) {
      if (isDef(oldCh) && isDef(ch)) {
        if (oldCh !== ch) updateChildren(elm, oldCh, ch, insertedVnodeQueue, removeOnly)
      } else if (isDef(ch)) {
        if (process.env.NODE_ENV !== 'production') {
          checkDuplicateKeys(ch)
        }
        if (isDef(oldVnode.text)) nodeOps.setTextContent(elm, '')
        addVnodes(elm, null, ch, 0, ch.length - 1, insertedVnodeQueue)
      } else if (isDef(oldCh)) {
        removeVnodes(oldCh, 0, oldCh.length - 1)
      } else if (isDef(oldVnode.text)) {
        nodeOps.setTextContent(elm, '')
      }
    } else if (oldVnode.text !== vnode.text) {
      nodeOps.setTextContent(elm, vnode.text)
    }
    if (isDef(data)) {
      if (isDef(i = data.hook) && isDef(i = i.postpatch)) i(oldVnode, vnode)
    }
  }
```

接下来就是我们经常说的vue-diff所在的方法<code>updateChildren</code>，先从参数说起，分别是父元素dom，旧的vnode-list，新的vnode-list，需要插入的vnode队列，是否仅移除。

重点的逻辑在<code>while</code>循环里：

如何理解这个diff逻辑，其实是分别有新旧两个vnode-list，两个list都设定第一位和最后一位作为两个游标，通过一系列判断对比，不断逼近，当两个list的两个游标相交则循环结束。

至于具体判断的逻辑就不赘述了，代码已经写得非常清楚了，在这里比较有意思的<code>sameVnode</code>的判断，在使用<code>v-for</code>生成的vnode-list不设置<code>key</code>的时候，所有的对比更新几乎都会从第三和第四个判断分支进行，即代码中的<code>sameVnode(oldStartVnode, newStartVnode)</code>和<code>sameVnode(oldEndVnode, newEndVnode)</code>判断，下面看看<code>sameVnode</code>的方法，当我们不设置key的时候，判断的逻辑会通过tag类型和vnode的数据某些属性进行比较，通常来说都是相同的，这就是官方文档说的<font color=red>原地复用</font>逻辑，直接更新当前节点的内容，不需要对当前的节点进行移动。这对于节点内容相对简单的来说默认会更高效，但是当节点内容相对复杂的时候我们就需要对节点内容进行复用而不是重新生成，这时候我们就需要设置<code>key</code>来复用节点。

最后的一段判断<code>oldStartIdx > oldEndIdx</code>和<code>newStartIdx > newEndIdx</code>则说明符合这两个条件的时候我们当前vnode-list是从无到有或从有到无的变化。

图示：官方文档的说明（👇）

![](https://user-gold-cdn.xitu.io/2019/6/23/16b84467afb18dfd?w=1458&h=632&f=png&s=186334)

<code>sameVnode</code>方法定义

```
function sameVnode (a, b) {
  return (
    a.key === b.key && (
      (
        a.tag === b.tag &&
        a.isComment === b.isComment &&
        isDef(a.data) === isDef(b.data) &&
        sameInputType(a, b)
      ) || (
        isTrue(a.isAsyncPlaceholder) &&
        a.asyncFactory === b.asyncFactory &&
        isUndef(b.asyncFactory.error)
      )
    )
  )
}
```

<code>updateChildren</code>方法定义

```
  function updateChildren (parentElm, oldCh, newCh, insertedVnodeQueue, removeOnly) {
    let oldStartIdx = 0
    let newStartIdx = 0
    let oldEndIdx = oldCh.length - 1
    let oldStartVnode = oldCh[0]
    let oldEndVnode = oldCh[oldEndIdx]
    let newEndIdx = newCh.length - 1
    let newStartVnode = newCh[0]
    let newEndVnode = newCh[newEndIdx]
    let oldKeyToIdx, idxInOld, vnodeToMove, refElm

    // removeOnly is a special flag used only by <transition-group>
    // to ensure removed elements stay in correct relative positions
    // during leaving transitions
    const canMove = !removeOnly

    if (process.env.NODE_ENV !== 'production') {
      checkDuplicateKeys(newCh)
    }

    while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
      if (isUndef(oldStartVnode)) {
        oldStartVnode = oldCh[++oldStartIdx] // Vnode has been moved left
      } else if (isUndef(oldEndVnode)) {
        oldEndVnode = oldCh[--oldEndIdx]
      } else if (sameVnode(oldStartVnode, newStartVnode)) {
        patchVnode(oldStartVnode, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
        oldStartVnode = oldCh[++oldStartIdx]
        newStartVnode = newCh[++newStartIdx]
      } else if (sameVnode(oldEndVnode, newEndVnode)) {
        patchVnode(oldEndVnode, newEndVnode, insertedVnodeQueue, newCh, newEndIdx)
        oldEndVnode = oldCh[--oldEndIdx]
        newEndVnode = newCh[--newEndIdx]
      } else if (sameVnode(oldStartVnode, newEndVnode)) { // Vnode moved right
        patchVnode(oldStartVnode, newEndVnode, insertedVnodeQueue, newCh, newEndIdx)
        canMove && nodeOps.insertBefore(parentElm, oldStartVnode.elm, nodeOps.nextSibling(oldEndVnode.elm))
        oldStartVnode = oldCh[++oldStartIdx]
        newEndVnode = newCh[--newEndIdx]
      } else if (sameVnode(oldEndVnode, newStartVnode)) { // Vnode moved left
        patchVnode(oldEndVnode, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
        canMove && nodeOps.insertBefore(parentElm, oldEndVnode.elm, oldStartVnode.elm)
        oldEndVnode = oldCh[--oldEndIdx]
        newStartVnode = newCh[++newStartIdx]
      } else {
        if (isUndef(oldKeyToIdx)) oldKeyToIdx = createKeyToOldIdx(oldCh, oldStartIdx, oldEndIdx)
        idxInOld = isDef(newStartVnode.key)
          ? oldKeyToIdx[newStartVnode.key]
          : findIdxInOld(newStartVnode, oldCh, oldStartIdx, oldEndIdx)
        if (isUndef(idxInOld)) { // New element
          createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx)
        } else {
          vnodeToMove = oldCh[idxInOld]
          if (sameVnode(vnodeToMove, newStartVnode)) {
            patchVnode(vnodeToMove, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
            oldCh[idxInOld] = undefined
            canMove && nodeOps.insertBefore(parentElm, vnodeToMove.elm, oldStartVnode.elm)
          } else {
            // same key but different element. treat as new element
            createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx)
          }
        }
        newStartVnode = newCh[++newStartIdx]
      }
    }
    if (oldStartIdx > oldEndIdx) {
      refElm = isUndef(newCh[newEndIdx + 1]) ? null : newCh[newEndIdx + 1].elm
      addVnodes(parentElm, refElm, newCh, newStartIdx, newEndIdx, insertedVnodeQueue)
    } else if (newStartIdx > newEndIdx) {
      removeVnodes(oldCh, oldStartIdx, oldEndIdx)
    }
  }
```

## 总结

其实vue-diff的算法并不复杂，代码阅读起来也相对容易。在vue里从patch到视图的变化是实时的，即假如存在3个节点变化，vue并不是收集完所有的patch再一次性更新视图，而是在遍历diff的过程中patch直接更新视图。
