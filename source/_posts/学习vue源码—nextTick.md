---
title: 学习vue源码—nextTick
date: 2019-07-07 22:40:14
tags: Vue
categories:
  - JavaScript
  - Vue
---
> 这一篇主要讲讲<code>nextTick</code>源码，看看该方法的实现，以及为何能在这个方法里保证拿到<code>DOM</code>节点。

<code>nextTick</code>方法在<code>./src/core/util/next-tick.js</code>，下面为部分源码展示：

1. <code>nextTick</code>方法接受两个入参，分别是回调方法<code>cb</code>和上下文<code>ctx</code>;
2. 函数部分逻辑，首先不管是否存在<code>cb</code>参数都会往队列推入一个函数，后续任务队列根据<code>cb</code>参数判断是否调用<code>cb</code>或者是否执行<code>_resolve(ctx)</code>修改<code>promise</code>状态；
3. 判断<code>pending</code>状态是否执行任务
4. 最后则是该函数的返回值为一个<code>promise</code>


```
export function nextTick (cb?: Function, ctx?: Object) {
  let _resolve
  callbacks.push(() => {
    if (cb) {
      try {
        cb.call(ctx)
      } catch (e) {
        handleError(e, ctx, 'nextTick')
      }
    } else if (_resolve) {
      _resolve(ctx)
    }
  })
  if (!pending) {
    pending = true
    timerFunc()
  }
  // $flow-disable-line
  if (!cb && typeof Promise !== 'undefined') {
    return new Promise(resolve => {
      _resolve = resolve
    })
  }
}
```

先来说说调用<code>nextTick</code>的返回值，因为返回值是一个<code>promise</code>，所以我们可以使用<code>then</code>的写法或者<code>async/await</code>的写法，加上使用<code>cb</code>的写法，存在三种写法。


```
this.$nextTick(function() {
    // do something
})

or

this.$nextTick().then((ctx)=> {
    // do something
})

or

await this.$nextTick()
// do something
```

接下来则是<code>nextTick</code>里比较重要的方法<code>timerFunc</code>的实现：

1. 优先使用原生<code>Promise</code>；
2. 后使用<code>MutationObserver</code>；
3. 再后使用<code>setImmediate</code>;
4. 最后使用<code>setTimeout</code>;

```
if (typeof Promise !== 'undefined' && isNative(Promise)) {
  const p = Promise.resolve()
  timerFunc = () => {
    p.then(flushCallbacks)
    // In problematic UIWebViews, Promise.then doesn't completely break, but
    // it can get stuck in a weird state where callbacks are pushed into the
    // microtask queue but the queue isn't being flushed, until the browser
    // needs to do some other work, e.g. handle a timer. Therefore we can
    // "force" the microtask queue to be flushed by adding an empty timer.
    if (isIOS) setTimeout(noop)
  }
  isUsingMicroTask = true
} else if (!isIE && typeof MutationObserver !== 'undefined' && (
  isNative(MutationObserver) ||
  // PhantomJS and iOS 7.x
  MutationObserver.toString() === '[object MutationObserverConstructor]'
)) {
  // Use MutationObserver where native Promise is not available,
  // e.g. PhantomJS, iOS7, Android 4.4
  // (#6466 MutationObserver is unreliable in IE11)
  let counter = 1
  const observer = new MutationObserver(flushCallbacks)
  const textNode = document.createTextNode(String(counter))
  observer.observe(textNode, {
    characterData: true
  })
  timerFunc = () => {
    counter = (counter + 1) % 2
    textNode.data = String(counter)
  }
  isUsingMicroTask = true
} else if (typeof setImmediate !== 'undefined' && isNative(setImmediate)) {
  // Fallback to setImmediate.
  // Techinically it leverages the (macro) task queue,
  // but it is still a better choice than setTimeout.
  timerFunc = () => {
    setImmediate(flushCallbacks)
  }
} else {
  // Fallback to setTimeout.
  timerFunc = () => {
    setTimeout(flushCallbacks, 0)
  }
}
```

从代码中<code>isUsingMicroTask</code>中可以看到只有<code>Promise</code>、<code>MutationObserver</code>属于微任务，另外两个则属于宏任务；看到该方法的实现我们就可以知道为什么在<code>nextTick</code>方法中能保证拿到<code>DOM</code>。

两种场景的解释：

1. 在<code>vue</code>第一次初始化的时候，我们在<code>beforeCreated</code>和<code>created</code>生命周期里想要使用<code>DOM</code>则必须使用<code>nextTick</code>，这是因为初始化的过程属于宏任务，整个函数调用栈未清空，<code>nextTick</code>的回调属于微任务，所以<code>nextTick</code>的回调必须在整个初始化结束后才会执行。
2. 在修改<code>data</code>数据后，又如何保证获取修改后的数据<code>DOM</code>？修改<code>data</code>数据实际上是触发组件实例的<code>watcher</code>执行<code>update</code>更新，而在<code>update</code>里面又执行了<code>queueWatcher</code>，下面👇则是<code>queueWatcher</code>方法的代码，在代码里面我们可以看到最后实际上也是调用<code>nextTick(flushSchedulerQueue)</code>。因此，想获取<code>data</code>修改后的<code>DOM</code>，调用<code>nextTick</code>能保证这种任务执行的顺序。

了解<code>watcher</code>可以看这篇[https://juejin.im/post/5d181bafe51d457753138219](https://juejin.im/post/5d181bafe51d457753138219)。

```
export function queueWatcher (watcher: Watcher) {
  const id = watcher.id
  if (has[id] == null) {
    has[id] = true
    if (!flushing) {
      queue.push(watcher)
    } else {
      // if already flushing, splice the watcher based on its id
      // if already past its id, it will be run next immediately.
      let i = queue.length - 1
      while (i > index && queue[i].id > watcher.id) {
        i--
      }
      queue.splice(i + 1, 0, watcher)
    }
    // queue the flush
    if (!waiting) {
      waiting = true

      if (process.env.NODE_ENV !== 'production' && !config.async) {
        flushSchedulerQueue()
        return
      }
      nextTick(flushSchedulerQueue)
    }
  }
}
```

其实<code>queueWatcher</code>方法里面的逻辑还告诉了我们另外一个框架知识点：

**为什么我们同时修改多个data属性，不会多次更新视图？**

在<code>update</code>方法里，因为最后实际上调用<code>nextTick</code>执行微任务去更新视图，了解过<code>event loop</code>机制的应该知道，必须等待当前宏任务的调用栈清空才去执行微任务，这也就是为什么当我们同时修改多个<code>data</code>属性时候，该判断<code>if (has[id] == null) </code>防止重复添加更新任务，并且利用了<code>event loop</code>机制在合适的时机去更新视图。
