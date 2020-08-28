---
title: 学习vue源码—mvvm
date: 2019-06-30 22:43:12
tags: Vue
categories:
  - JavaScript
  - Vue
---
> 这一篇主要是讲解一下vue里mvvm的原理，以及如何理解vue实现mvvm。

稍微有去了解过vue是如何双向绑定的我们都很容易知道vue是通过<code>Object.defineProperty</code>劫持<code>data</code>属性的<code>setter</code>和<code>getter</code>，但是这仅仅只是实现的一部分，在这个实现里我们还要理解<code>dep</code>（订阅中心）和<code>watcher</code>（订阅者）的概念。

### <code>dep</code>—订阅中心

<code>dep</code>代码在<code>./src/core/observer/dep.js</code>文件里，下面简单讲解一下：

1. <code>dep</code>的定义参考了观察者设计模式，每一个<code>dep</code>有自己的唯一标识<code>id</code>和订阅者列表<code>subs</code>。
2. <code>addSub</code>和<code>removeSub</code>用来管理订阅者列表<code>subs</code>。
3. <code>depend</code>用来收集<code>watcher</code>（订阅者）。
4. <code>notify</code>用来通知<code>watcher</code>（订阅者）执行更新。
5. <code>Dep.target</code>刚开始看是比较难理解的一个概念，<code>Dep.target</code>其实是调用当前<code>dep</code>对应属性的<code>watcher</code>。举个例子：假如<code>data</code>有个属性<code>name</code>，那么当<code>data.name</code>的<code>getter</code>被触发时，我们需要知道是谁在调用这个<code>data.name</code>的<code>getter</code>，这就是<code>Dep.target</code>。

```
export default class Dep {
  static target: ?Watcher;
  id: number;
  subs: Array<Watcher>;

  constructor () {
    this.id = uid++
    this.subs = []
  }

  addSub (sub: Watcher) {
    this.subs.push(sub)
  }

  removeSub (sub: Watcher) {
    remove(this.subs, sub)
  }

  depend () {
    if (Dep.target) {
      Dep.target.addDep(this)
    }
  }

  notify () {
    // stabilize the subscriber list first
    const subs = this.subs.slice()
    if (process.env.NODE_ENV !== 'production' && !config.async) {
      // subs aren't sorted in scheduler if not running async
      // we need to sort them now to make sure they fire in correct
      // order
      subs.sort((a, b) => a.id - b.id)
    }
    for (let i = 0, l = subs.length; i < l; i++) {
      subs[i].update()
    }
  }
}
// The current target watcher being evaluated.
// This is globally unique because only one watcher
// can be evaluated at a time.
Dep.target = null
const targetStack = []

export function pushTarget (target: ?Watcher) {
  targetStack.push(target)
  Dep.target = target
}

export function popTarget () {
  targetStack.pop()
  Dep.target = targetStack[targetStack.length - 1]
}

```

### <code>watcher</code>—订阅者

<code>watcher</code>代码在<code>./src/core/observer/watcher.js</code>文件里，关于<code>watcher</code>的选项配置就不细说了，在这里我们只需要重点关注其中的<code>get</code>
、<code>update</code>、<code>run</code>、<code>evaluate</code>这几个方法。

这几个方法的作用稍后解释，现在我们要先理解怎样才会产生一个<code>watcher</code>。在vue里面，有三种类型的<code>watcher</code>：

1. 每一个组件的实例都是一个<code>watcher</code>
2. 在组件的<code>watch</code>选项中声明的<code>watcher</code>
3. 计算属性所使用的依赖值会给对应的依赖值添加一个<code>watcher</code>

讲完<code>watcher</code>的来源后我们再来看这几个方法的讲解：

1. 先从<code>update</code>讲起，当某个响应属性发生变化时触发<code>setter</code>后，执行<code>dep.notify</code>通知每个<code>watcher</code>执行<code>update</code>，代码比较简单，三个逻辑分支，判断<code>this.lazy</code>，这是应用于<font color=red>计算属性</font>时会触发的逻辑分支，<code>this.sync
   </code>则用于判断同步执行<code>watcher</code>的回调，否则推入<code>queueWatcher</code>后续执行。
2. <code>run</code>和<code>evaluate</code>都是会调用<code>get</code>方法，只是<code>run</code>方法是用于组件实例的<code>watcher</code>和<code>watch</code>选项中声明的<code>watcher</code>，<code>watch</code>选项中声明的<code>watcher</code>的<code>this.user</code>为<code>true</code>，在<code>run</code>方法中的<code>this.cb.call(this.vm, value, oldValue)</code>这段代码则是我们<code>watch</code>选项中触发的回调。至于<code>evaluate</code>方法则更加简单了，调用<code>get</code>方法然后设置<code>this.dirty</code>为<code>false</code>则是为了后续其他地方使用这个计算属性的时候不需要重新计算，这也是<font color=red>计算属性缓存</font>的一部分逻辑。
3. 接下来讲讲<code>get</code>方法，<code>pushTarget(this)</code>这段则是设置<code>Dep.target</code>为当前<code>watcher</code>实例，其实就是告诉<code>dep</code>是谁在获取属性。<code>value = this.getter.call(vm, vm)</code>则是获取当前值，在这里三种类型的<code>watcher</code>的<code>getter</code>是不一样的。
4. 最后提一下，计算属性的值一般是在组件实例的<code>watcher</code>执行<code>getter</code>的过程中执行计算的。

| watcher类型 | getter                                 |
| ----------- | -------------------------------------- |
| 组件实例    | render函数                             |
| watch       | 执行parsePath方法生成的函数            |
| 计算属性    | 执行createComputedGetter方法生成的函数 |

```
export default class Watcher {
  vm: Component;
  expression: string;
  cb: Function;
  id: number;
  deep: boolean;
  user: boolean;
  lazy: boolean;
  sync: boolean;
  dirty: boolean;
  active: boolean;
  deps: Array<Dep>;
  newDeps: Array<Dep>;
  depIds: SimpleSet;
  newDepIds: SimpleSet;
  before: ?Function;
  getter: Function;
  value: any;

  constructor (
    vm: Component,
    expOrFn: string | Function,
    cb: Function,
    options?: ?Object,
    isRenderWatcher?: boolean
  ) {
    this.vm = vm
    if (isRenderWatcher) {
      vm._watcher = this
    }
    vm._watchers.push(this)
    // options
    if (options) {
      this.deep = !!options.deep
      this.user = !!options.user
      this.lazy = !!options.lazy
      this.sync = !!options.sync
      this.before = options.before
    } else {
      this.deep = this.user = this.lazy = this.sync = false
    }
    this.cb = cb
    this.id = ++uid // uid for batching
    this.active = true
    this.dirty = this.lazy // for lazy watchers
    this.deps = []
    this.newDeps = []
    this.depIds = new Set()
    this.newDepIds = new Set()
    this.expression = process.env.NODE_ENV !== 'production'
      ? expOrFn.toString()
      : ''
    // parse expression for getter
    if (typeof expOrFn === 'function') {
      this.getter = expOrFn
    } else {
      this.getter = parsePath(expOrFn)
      if (!this.getter) {
        this.getter = noop
        process.env.NODE_ENV !== 'production' && warn(
          `Failed watching path: "${expOrFn}" ` +
          'Watcher only accepts simple dot-delimited paths. ' +
          'For full control, use a function instead.',
          vm
        )
      }
    }
    this.value = this.lazy
      ? undefined
      : this.get()
  }

  /**
   * Evaluate the getter, and re-collect dependencies.
   */
  get () {
    pushTarget(this)
    let value
    const vm = this.vm
    try {
      value = this.getter.call(vm, vm)
    } catch (e) {
      if (this.user) {
        handleError(e, vm, `getter for watcher "${this.expression}"`)
      } else {
        throw e
      }
    } finally {
      // "touch" every property so they are all tracked as
      // dependencies for deep watching
      if (this.deep) {
        traverse(value)
      }
      popTarget()
      this.cleanupDeps()
    }
    return value
  }

  /**
   * Add a dependency to this directive.
   */
  addDep (dep: Dep) {
    const id = dep.id
    if (!this.newDepIds.has(id)) {
      this.newDepIds.add(id)
      this.newDeps.push(dep)
      if (!this.depIds.has(id)) {
        dep.addSub(this)
      }
    }
  }

  /**
   * Clean up for dependency collection.
   */
  cleanupDeps () {
    let i = this.deps.length
    while (i--) {
      const dep = this.deps[i]
      if (!this.newDepIds.has(dep.id)) {
        dep.removeSub(this)
      }
    }
    let tmp = this.depIds
    this.depIds = this.newDepIds
    this.newDepIds = tmp
    this.newDepIds.clear()
    tmp = this.deps
    this.deps = this.newDeps
    this.newDeps = tmp
    this.newDeps.length = 0
  }

  /**
   * Subscriber interface.
   * Will be called when a dependency changes.
   */
  update () {
    /* istanbul ignore else */
    if (this.lazy) {
      this.dirty = true
    } else if (this.sync) {
      this.run()
    } else {
      queueWatcher(this)
    }
  }

  /**
   * Scheduler job interface.
   * Will be called by the scheduler.
   */
  run () {
    if (this.active) {
      const value = this.get()
      if (
        value !== this.value ||
        // Deep watchers and watchers on Object/Arrays should fire even
        // when the value is the same, because the value may
        // have mutated.
        isObject(value) ||
        this.deep
      ) {
        // set new value
        const oldValue = this.value
        this.value = value
        if (this.user) {
          try {
            this.cb.call(this.vm, value, oldValue)
          } catch (e) {
            handleError(e, this.vm, `callback for watcher "${this.expression}"`)
          }
        } else {
          this.cb.call(this.vm, value, oldValue)
        }
      }
    }
  }

  /**
   * Evaluate the value of the watcher.
   * This only gets called for lazy watchers.
   */
  evaluate () {
    this.value = this.get()
    this.dirty = false
  }

  /**
   * Depend on all deps collected by this watcher.
   */
  depend () {
    let i = this.deps.length
    while (i--) {
      this.deps[i].depend()
    }
  }

  /**
   * Remove self from all dependencies' subscriber list.
   */
  teardown () {
    if (this.active) {
      // remove self from vm's watcher list
      // this is a somewhat expensive operation so we skip it
      // if the vm is being destroyed.
      if (!this.vm._isBeingDestroyed) {
        remove(this.vm._watchers, this)
      }
      let i = this.deps.length
      while (i--) {
        this.deps[i].removeSub(this)
      }
      this.active = false
    }
  }
}

```

### 总结

其实学习vue的mvvm，重点在于<code>dep</code>和<code>watcher</code>的理解，要明白这两个类的实例在双向绑定的过程中扮演的是一个什么样角色，单纯从代码上可能不太容易理解这样设计的意图，但是如果能有一个比较具象化的东西来对应，相信对你的理解会有非常大的帮助。
