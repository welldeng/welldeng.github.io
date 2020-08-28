---
title: 学习vue-router源码记录-2
date: 2019-06-16 22:42:32
tags: Vue
categories:
  - JavaScript
  - Vue
---
 >继上一遍文章大概介绍了vue-router里面的概念，这一篇文章主要详细介绍路由跳转中发生了什么。

 路由跳转执行的代码主要在<code>./base.js</code>文件里，详细看<code>transitionTo</code>方法。

```
transitionTo (location: RawLocation, onComplete?: Function, onAbort?: Function) {
    const route = this.router.match(location, this.current)
    this.confirmTransition(route, () => {
      this.updateRoute(route)
      onComplete && onComplete(route)
      this.ensureURL()

      // fire ready cbs once
      if (!this.ready) {
        this.ready = true
        this.readyCbs.forEach(cb => { cb(route) })
      }
    }, err => {
      if (onAbort) {
        onAbort(err)
      }
      if (err && !this.ready) {
        this.ready = true
        this.readyErrorCbs.forEach(cb => { cb(err) })
      }
    })
  }
```

<code>transitionTo</code>代码非常简单，执行<code>this.route.match</code>，通过比较新的路由和就得路由拿到一个<code>route</code>对象，然后执行<code>confirmTransition</code>确认路由跳转。

下面看看这个<code>route</code>对象的定义

```
declare type Route = {
  path: string;
  name: ?string;
  hash: string;
  query: Dictionary<string>;
  params: Dictionary<string>;
  fullPath: string;
  matched: Array<RouteRecord>;
  redirectedFrom?: string;
  meta?: any;
}
```

对照代码，我们主要看<code>matched</code>这个属性，在上一篇文章里面已经介绍过<code>RouteRecord</code>对象的定义。路由一开始会执行<code>createMatcher</code>生成一个路由映射表，因此<code>matched</code>里面存放就是我们<font color=red>将要跳转到的路由匹配上的路由配置对象</font>。

举一个简单的例子说明：

```
[{
    path: '/parent',
    component: Parent,
    children: [
    { path: 'foo', component: Foo },
    { path: 'bar', component: Bar },
    ],
}]
```

假设我们配置了以下的路由，<code>createMatcher</code>会生成根据path建立的映射表

```
pathMap = {
    '/parent': RouteRecord,
    '/parent/foo': RouteRecord,
    '/parent/bar': RouteRecord,
}
```

假如我们发生了一个从<code>/parent</code>路由跳转到<code>/parent/foo</code>路由，首先执行以下代码生成的<code>route</code>对象

```
const route = this.router.match(location, this.current)
```

因此根据我们假设的配置，这里的<code>route</code>里的<code>matched</code>将会包含<code>/parent</code>和<code>/parent/foo</code>的<code>RouteRecord</code>。至于具体的<code>match</code>方法代码就不详细解释了。

继续讲路由的跳转，生成<code>route</code>对象后将会执行一个确认的方法<code>confirmTransition</code>。

由于这个方法代码比较长，我们拆开来说明，首先看这个方法的入参说明，接受三个参数，<code>route</code>对象在前面已经生成过了，另外两个是执行完成的回调方法和退出的回调方法。

```
confirmTransition (route: Route, onComplete: Function, onAbort?: Function)
```

在代码的一开始首先判断当前路由与跳转的路由是否是同一个路由，如果是则直接退出。

```
    const current = this.current
    const abort = err => {
      if (isError(err)) {
        if (this.errorCbs.length) {
          this.errorCbs.forEach(cb => { cb(err) })
        } else {
          warn(false, 'uncaught error during route navigation:')
          console.error(err)
        }
      }
      onAbort && onAbort(err)
    }
    if (
      isSameRoute(route, current) &&
      // in the case the route map has been dynamically appended to
      route.matched.length === current.matched.length
    ) {
      this.ensureURL()
      return abort()
    }

```

这里的<code>ensureURL</code>方法定义在<code>HTML5History</code>的原型链上，实际上执行的是保存路由变化历史记录，根据<code>push</code>是<code>true</code>或<code>false</code>来确定执行<code>pushState</code>还是<code>replaceState</code>。这一方法在执行完路由跳转后同样会执行一次。

```
  HTML5History.prototype.ensureURL = function ensureURL (push) {
    if (getLocation(this.base) !== this.current.fullPath) {
      var current = cleanPath(this.base + this.current.fullPath);
      push ? pushState(current) : replaceState(current);
    }
  };
```

继续看后面的代码，首先通过<code>resolveQueue</code>对比<code>this.current</code>和<code>route</code>对象的<code>matched</code>提取三种变化的组件队列。根据命名我们直接可得知<code>updated</code>、<code>deactivated</code>、<code>activated</code>分别对应更新的组件、失效的组件、激活的组件。然后生成一个需要执行方法的队列<code>queue</code>，根据这个队列的生成定义，我们可以看出执行方法的顺序，至于通过<code>extractLeaveGuards</code>和<code>extractUpdateHooks</code>方法提取组件里的守卫函数就不细说了。

1. 在失活的组件里调用离开守卫。

2. 调用全局的 beforeEach 守卫。

3. 在重用的组件里调用 beforeRouteUpdate 守卫

4. 在激活的路由配置里调用 beforeEnter。

5. 解析异步路由组件。

```
    const {
      updated,
      deactivated,
      activated
    } = resolveQueue(this.current.matched, route.matched)

    const queue: Array<?NavigationGuard> = [].concat(
      // in-component leave guards
      extractLeaveGuards(deactivated),
      // global before hooks
      this.router.beforeHooks,
      // in-component update hooks
      extractUpdateHooks(updated),
      // in-config enter guards
      activated.map(m => m.beforeEnter),
      // async components
      resolveAsyncComponents(activated)
    )
```

看看<code>resolveQueue</code>是如何提取变化的组件。比较<code>current</code>和<code>next</code>确定一个变化的位置<code>i</code>，<code>next</code>里的从<code>0</code>到<code>i</code>则是<code>updated</code>的部分，<code>i</code>之后的则是<code>activated</code>的部分，而<code>current</code>里<code>i</code>之后的则是<code>deactivated</code>的部分。

```
function resolveQueue (
  current: Array<RouteRecord>,
  next: Array<RouteRecord>
): {
  updated: Array<RouteRecord>,
  activated: Array<RouteRecord>,
  deactivated: Array<RouteRecord>
} {
  let i
  const max = Math.max(current.length, next.length)
  for (i = 0; i < max; i++) {
    if (current[i] !== next[i]) {
      break
    }
  }
  return {
    updated: next.slice(0, i),
    activated: next.slice(i),
    deactivated: current.slice(i)
  }
}
```

接下来就是生成迭代器方法<code>iterator</code>，执行<code>runQueue</code>方法。

```
    this.pending = route
    const iterator = (hook: NavigationGuard, next) => {
      if (this.pending !== route) {
        return abort()
      }
      try {
        hook(route, current, (to: any) => {
          if (to === false || isError(to)) {
            // next(false) -> abort navigation, ensure current URL
            this.ensureURL(true)
            abort(to)
          } else if (
            typeof to === 'string' ||
            (typeof to === 'object' && (
              typeof to.path === 'string' ||
              typeof to.name === 'string'
            ))
          ) {
            // next('/') or next({ path: '/' }) -> redirect
            abort()
            if (typeof to === 'object' && to.replace) {
              this.replace(to)
            } else {
              this.push(to)
            }
          } else {
            // confirm transition and pass on the value
            next(to)
          }
        })
      } catch (e) {
        abort(e)
      }
    }
    
    runQueue(queue, iterator, () => {
      const postEnterCbs = []
      const isValid = () => this.current === route
      // wait until async components are resolved before
      // extracting in-component enter guards
      const enterGuards = extractEnterGuards(activated, postEnterCbs, isValid)
      const queue = enterGuards.concat(this.router.resolveHooks)
      runQueue(queue, iterator, () => {
        if (this.pending !== route) {
          return abort()
        }
        this.pending = null
        onComplete(route)
        if (this.router.app) {
          this.router.app.$nextTick(() => {
            postEnterCbs.forEach(cb => { cb() })
          })
        }
      })
    })
```

<code>runQueue</code>方法的代码并不复杂，一个递归执行队列的方法，使用<code>iterator(fn参数)</code>执行<code>queue</code>，<code>iterator</code>里给<code>hook</code>传入的参数分别代表<code>to</code>、<code>from</code>、<code>next</code>，在队列执行完后执行传入的回调方法。这里执行过程代表了vue-router的守卫函数的执行函数。

```
export function runQueue (queue: Array<?NavigationGuard>, fn: Function, cb: Function) {
  const step = index => {
    if (index >= queue.length) {
      cb()
    } else {
      if (queue[index]) {
        fn(queue[index], () => {
          step(index + 1)
        })
      } else {
        step(index + 1)
      }
    }
  }
  step(0)
}
```

综上所述，路由跳转的变化大概上已经解释完，当然这并不是完整的执行逻辑，只是路由跳转大概的过程差不多就是如此。
