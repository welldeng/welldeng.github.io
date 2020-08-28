---
title: 学习vue-router源码记录-1
date: 2019-06-07 22:42:32
tags: Vue
categories:
  - JavaScript
  - Vue
---
> 因为本人开发中使用的是VUE技术栈，最近也是开始源码的学习，以此记录个人理解，若行文有误，请多多指教。

## 1.new Router和install

在vue中我们使用vue-router时需要先进行<code>new Router()</code>，执行<code>new Router()</code>后主要执行代码看看VueRouter class定义的<code>constructor</code>方法。

```
constructor (options: RouterOptions = {}) {
    this.app = null
    this.apps = []
    this.options = options
    this.beforeHooks = []
    this.resolveHooks = []
    this.afterHooks = []
    this.matcher = createMatcher(options.routes || [], this)

    let mode = options.mode || 'hash'
    this.fallback = mode === 'history' && !supportsPushState && options.fallback !== false
    if (this.fallback) {
      mode = 'hash'
    }
    if (!inBrowser) {
      mode = 'abstract'
    }
    this.mode = mode

    switch (mode) {
      case 'history':
        this.history = new HTML5History(this, options.base)
        break
      case 'hash':
        this.history = new HashHistory(this, options.base, this.fallback)
        break
      case 'abstract':
        this.history = new AbstractHistory(this, options.base)
        break
      default:
        if (process.env.NODE_ENV !== 'production') {
          assert(false, `invalid mode: ${mode}`)
        }
    }
  }
```

从代码里面我们可以看到在<code>new</code>的时候确定了使用何种路由模式，并且根据传入options创建<code>matcher</code>。

接下来看看当使用<code>vue.use()</code>执行install方法做了什么：

```
export function install (Vue) {
  if (install.installed && _Vue === Vue) return
  install.installed = true

  _Vue = Vue

  const isDef = v => v !== undefined

  const registerInstance = (vm, callVal) => {
    let i = vm.$options._parentVnode
    if (isDef(i) && isDef(i = i.data) && isDef(i = i.registerRouteInstance)) {
      i(vm, callVal)
    }
  }

  Vue.mixin({
    beforeCreate () {
      if (isDef(this.$options.router)) {
        this._routerRoot = this
        this._router = this.$options.router
        this._router.init(this)
        Vue.util.defineReactive(this, '_route', this._router.history.current)
      } else {
        this._routerRoot = (this.$parent && this.$parent._routerRoot) || this
      }
      registerInstance(this, this)
    },
    destroyed () {
      registerInstance(this)
    }
  })

  Object.defineProperty(Vue.prototype, '$router', {
    get () { return this._routerRoot._router }
  })

  Object.defineProperty(Vue.prototype, '$route', {
    get () { return this._routerRoot._route }
  })

  Vue.component('RouterView', View)
  Vue.component('RouterLink', Link)

  const strats = Vue.config.optionMergeStrategies
  // use the same hook merging strategy for route hooks
  strats.beforeRouteEnter = strats.beforeRouteLeave = strats.beforeRouteUpdate = strats.created
}
```

<code>install</code>方法里主要是<code>Vue.mixin</code>给每个组件混入了<code>beforeCreate</code>和 <code>destroyed </code>方法，在Vue的原型链上增加了<code>$router</code>和<code>$route</code>对象，这就是为什么我们使用Vue的时候在this上可以拿到这两个对象，注册了<code>router-view</code>和<code>router-link</code>两个组件。

## 2. matcher和route

接下来看看<code>matcher</code>的定义：

```
export type Matcher = {
  match: (raw: RawLocation, current?: Route, redirectedFrom?: Location) => Route;
  addRoutes: (routes: Array<RouteConfig>) => void;
};
```

<code>matcher</code>暴露了<code>match</code>方法<code>addRoutes</code>方法，从方法的名字上看，match方法是用于路由匹配，addRoutes则是用来添加路由配置。

在执行<code>creatMatcher()</code>里第一代段代码生成了<code>pathList</code>、<code>pathMap</code>、<code>nameMap</code>这三个对象，这是后面路由执行匹配非常重要的配置。

```
  const { pathList, pathMap, nameMap } = createRouteMap(routes)
```

<code>pathMap</code>和<code>nameMap</code>分别是以route配置的path和name为key生成的一个映射表，对应value为<code>RouteRecord</code>实例。

下面看看<code>RouteRecord</code>的定义：

```
declare type RouteRecord = {
  path: string;
  regex: RouteRegExp;
  components: Dictionary<any>;
  instances: Dictionary<any>;
  name: ?string;
  parent: ?RouteRecord;
  redirect: ?RedirectOption;
  matchAs: ?string;
  beforeEnter: ?NavigationGuard;
  meta: any;
  props: boolean | Object | Function | Dictionary<boolean | Object | Function>;
}
```

结合代码里的实际数据对照理解各个属性的含义：

![](https://user-gold-cdn.xitu.io/2019/6/7/16b318773604a842?w=1598&h=522&f=png&s=132269)

| key        | value                                                        |
| ---------- | ------------------------------------------------------------ |
| path       | 传入的路径值                                                 |
| regex      | 根据path生成的正则匹配规则                                   |
| components | path对应的组件                                               |
| instances  | 执行路由守卫方法时传入的route实例                            |
| name       | route的name                                                  |
| parent     | route的父级，是一个递归的对象，从最底层一直到最顶层，无则为undefined |
| redirect   | 重定向的路径                                                 |
| matchAs    | 用于匹配alias                                                |
| props      | 传入路由的参数                                               |

结合以上解释，我们可以得出<code>vue-router</code>一个大概的运行概念。

1. 执行<code>new Router()</code>生成路由配置对象<code>routedRecord</code>

2. 路由匹配根据<code>route</code>对象的<code>regex</code>进行匹配

3. 根据<code>route</code>的<code>parent</code>对象递归获取<code>component</code>组件生成<code>render Tree</code>

4. 执行各组件对应的导航守卫方法

此文大概简述了vue-router是如何执行的，但是对于router跳转的具体执行并没有进行深入解释，下一篇文章将会详细说明router跳转之后是如何执行。
