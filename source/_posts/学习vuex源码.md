---
title: 学习vuex源码
date: 2020-08-27 17:43:10
tags: Vue
categories:
  - JavaScript
---
> 这一篇主要是讲解<code>vuex</code>的大概实现，以及一些细节的说明。

### <code>vuex</code>是如何实现的？

先从<code>install</code>方法看，安装插件的方法实现比较简单，调用<code>applyMixin</code>，最后执行的是这段逻辑<code>Vue.mixin({ beforeCreate: vuexInit })</code>，实际上是在每个组件创建时混入<code>store</code>实例。所以我们可以在每个组件上获取到<code>store</code>实例上的数据。

```
if (version >= 2) {
Vue.mixin({ beforeCreate: vuexInit })
} else {
// override init and inject vuex init procedure
// for 1.x backwards compatibility.
const _init = Vue.prototype._init
Vue.prototype._init = function (options = {}) {
  options.init = options.init
    ? [vuexInit].concat(options.init)
    : vuexInit
  _init.call(this, options)
}
}

function vuexInit () {
const options = this.$options
// store injection
if (options.store) {
  this.$store = typeof options.store === 'function'
    ? options.store()
    : options.store
} else if (options.parent && options.parent.$store) {
  this.$store = options.parent.$store
}
}
```

```
function install (_Vue) {
  if (Vue && _Vue === Vue) {
    if (process.env.NODE_ENV !== 'production') {
      console.error(
        '[vuex] already installed. Vue.use(Vuex) should be called only once.'
      )
    }
    return
  }
  Vue = _Vue
  applyMixin(Vue)
}
```

接下来看看<code>store</code>这个类的代码：

在这个类上暴露了<code>commit</code>和<code>dispatch</code>方法，并且绑定了调用上下文为<code>store</code>的实例。<code>dispatch</code>支持异步更新数据是因为它内部的实现就是使用了<code>promise</code>。

```
// bind commit and dispatch to self
const store = this
const { dispatch, commit } = this
this.dispatch = function boundDispatch (type, payload) {
  return dispatch.call(store, type, payload)
}
this.commit = function boundCommit (type, payload, options) {
  return commit.call(store, type, payload, options)
}
```

下面两个方法是<code>vuex</code>的实现主要逻辑，<code>installModule</code>的具体逻辑不细说，从注释上我们可以知道，这是一个初始化整个<code>vuex</code>配置生成对象的方法。根据配置进行递归执行。

```
// init root module.
// this also recursively registers all sub-modules
// and collects all module getters inside this._wrappedGetters
installModule(this, state, [], this._modules.root)

// initialize the store vm, which is responsible for the reactivity
// (also registers _wrappedGetters as computed properties)
resetStoreVM(this, state)
```

<code>resetStoreVM</code>方法比较重要，下面是该方法的详细代码：

1. <code>vuex</code>中的<code>getter</code>实际上是使用<code>vue</code>计算属性实现的，<code>Object.defineProperty</code>里定义了<code>getter</code>的<code>get</code>方法。
2. <code>store._vm</code>实际就是<code>vuex</code>把<code>installModule</code>生成的对象改造成响应式的数据，通过一个新的<code>vue</code>实例。
3. 最后则是当我们重置<code>vuex</code>的响应数据，需要销毁旧的实例，回收内存。

```
function resetStoreVM (store, state, hot) {
  const oldVm = store._vm

  // bind store public getters
  store.getters = {}
  const wrappedGetters = store._wrappedGetters
  const computed = {}
  forEachValue(wrappedGetters, (fn, key) => {
    // use computed to leverage its lazy-caching mechanism
    // direct inline function use will lead to closure preserving oldVm.
    // using partial to return function with only arguments preserved in closure enviroment.
    computed[key] = partial(fn, store)
    Object.defineProperty(store.getters, key, {
      get: () => store._vm[key],
      enumerable: true // for local getters
    })
  })

  // use a Vue instance to store the state tree
  // suppress warnings just in case the user has added
  // some funky global mixins
  const silent = Vue.config.silent
  Vue.config.silent = true
  store._vm = new Vue({
    data: {
      $$state: state
    },
    computed
  })
  Vue.config.silent = silent

  // enable strict mode for new vm
  if (store.strict) {
    enableStrictMode(store)
  }

  if (oldVm) {
    if (hot) {
      // dispatch changes in all subscribed watchers
      // to force getter re-evaluation for hot reloading.
      store._withCommit(() => {
        oldVm._data.$$state = null
      })
    }
    Vue.nextTick(() => oldVm.$destroy())
  }
}
```

### 修改<code>vuex</code>的数据是如何响应到视图？

1. <code>vuex</code>的<code>state</code>和<code>getter</code>实际上就是对应的<code>store._vm</code>（<code>vue</code>实例）中<code>data</code>和<code>computed</code>，因此<code>getter</code>所依赖的<code>state</code>最后是通过<code>watcher</code>管理的。
2. 当我们在组件中使用<code>vuex</code>，<code>vuex</code>生成的实例对象在<code>vue</code>实例化过程中被改造成响应式的数据，当我们有多个页面组件使用了<code>vuex</code>的数据，其实也是通过<code>watcher</code>管理，因此当我们使用<code>commit</code>或<code>dispatch</code>修改数据，最后触发了<code>setter</code>去通知所有订阅者（<code>watcher</code>）更新。

### 总结

<code>vuex</code>的设计其实并不复杂，简单的来讲，就是一个对象，通过内部的方法管理内部的属性和读取内部的属性。

而实现的过程，则是通过一系列方法把我们的配置生成一个<code>root</code>对象，然后利用<code>vue</code>实现内部数据的响应与依赖管理。而这里比较核心的部分则是当数据发生变化时，如何响应到对应的视图部分以及<code>getter</code>的依赖管理，这些逻辑的实现最后都是通过<code>watcher</code>。

