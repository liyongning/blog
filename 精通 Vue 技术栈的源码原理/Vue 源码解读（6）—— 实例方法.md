# Vue 源码解读（6）—— 实例方法

**当学习成为了习惯，知识也就变成了常识。** 感谢各位的 **关注**、**点赞**、**收藏**和**评论**。

新视频和文章会第一时间在微信公众号发送，欢迎关注：[李永宁lyn](https://gitee.com/liyongning/typora-image-bed/raw/master/202202171742614.jpg)

文章已收录到 [github 仓库 liyongning/blog](https://github.com/liyongning/blog)，欢迎 Watch 和 Star。

## 封面

![](https://gitee.com/liyongning/typora-image-bed/raw/master/202202241706743.png)

## 前言

上一篇文章 [Vue 源码解读（5）—— 全局 API](https://mp.weixin.qq.com/s?__biz=MzA3NTk4NjQ1OQ==&mid=2247484876&idx=1&sn=942fdeaa321e22e97d47a441682cc2c8&chksm=9f6966b8a81eefae7975e8dbca4cbbf092d5da81806a94084247cc3c5bccb340a5ea60c29543#rd) 详细介绍了 Vue 的各个全局 API 的实现原理，本篇文章将会详细介绍各个实例方法的实现原理。

## 目标

深入理解以下实例方法的实现原理。

* vm.$set

* vm.$delete

* vm.$watch

* vm.$on

* vm.$emit

* vm.$off

* vm.$once

* vm._update

* vm.$forceUpdate

* vm.$destroy

* vm.$nextTick

* vm._render

## 源码解读

### 入口

> /src/core/instance/index.js

该文件是 Vue 实例的入口文件，包括 Vue 构造函数的定义、各个实例方法的初始化。

```javascript
// Vue 的构造函数
function Vue (options) {
  // 调用 Vue.prototype._init 方法，该方法是在 initMixin 中定义的
  this._init(options)
}

// 定义 Vue.prototype._init 方法
initMixin(Vue)
/**
 * 定义：
 *   Vue.prototype.$data
 *   Vue.prototype.$props
 *   Vue.prototype.$set
 *   Vue.prototype.$delete
 *   Vue.prototype.$watch
 */
stateMixin(Vue)
/**
 * 定义 事件相关的 方法：
 *   Vue.prototype.$on
 *   Vue.prototype.$once
 *   Vue.prototype.$off
 *   Vue.prototype.$emit
 */
eventsMixin(Vue)
/**
 * 定义：
 *   Vue.prototype._update
 *   Vue.prototype.$forceUpdate
 *   Vue.prototype.$destroy
 */
lifecycleMixin(Vue)
/**
 * 执行 installRenderHelpers，在 Vue.prototype 对象上安装运行时便利程序
 * 
 * 定义：
 *   Vue.prototype.$nextTick
 *   Vue.prototype._render
 */
renderMixin(Vue)

```

### vm.\$data、vm.\$props

> src/core/instance/state.js

这是两个实例属性，不是实例方法，这里简单介绍以下，当然其本身实现也很简单

```javascript
// data
const dataDef = {}
dataDef.get = function () { return this._data }
// props
const propsDef = {}
propsDef.get = function () { return this._props }
// 将 data 属性和 props 属性挂载到 Vue.prototype 对象上
// 这样在程序中就可以通过 this.$data 和 this.$props 来访问 data 和 props 对象了
Object.defineProperty(Vue.prototype, '$data', dataDef)
Object.defineProperty(Vue.prototype, '$props', propsDef)

```

### vm.$set

> /src/core/instance/state.js

```javascript
Vue.prototype.$set = set
```

#### set

> /src/core/observer/index.js

```javascript
/**
 * 通过 Vue.set 或者 this.$set 方法给 target 的指定 key 设置值 val
 * 如果 target 是对象，并且 key 原本不存在，则为新 key 设置响应式，然后执行依赖通知
 */
export function set (target: Array<any> | Object, key: any, val: any): any {
  if (process.env.NODE_ENV !== 'production' &&
    (isUndef(target) || isPrimitive(target))
  ) {
    warn(`Cannot set reactive property on undefined, null, or primitive value: ${(target: any)}`)
  }
  // 更新数组指定下标的元素，Vue.set(array, idx, val)，通过 splice 方法实现响应式更新
  if (Array.isArray(target) && isValidArrayIndex(key)) {
    target.length = Math.max(target.length, key)
    target.splice(key, 1, val)
    return val
  }
  // 更新对象已有属性，Vue.set(obj, key, val)，执行更新即可
  if (key in target && !(key in Object.prototype)) {
    target[key] = val
    return val
  }
  const ob = (target: any).__ob__
  // 不能向 Vue 实例或者 $data 添加动态添加响应式属性，vmCount 的用处之一，
  // this.$data 的 ob.vmCount = 1，表示根组件，其它子组件的 vm.vmCount 都是 0
  if (target._isVue || (ob && ob.vmCount)) {
    process.env.NODE_ENV !== 'production' && warn(
      'Avoid adding reactive properties to a Vue instance or its root $data ' +
      'at runtime - declare it upfront in the data option.'
    )
    return val
  }
  // target 不是响应式对象，新属性会被设置，但是不会做响应式处理
  if (!ob) {
    target[key] = val
    return val
  }
  // 给对象定义新属性，通过 defineReactive 方法设置响应式，并触发依赖更新
  defineReactive(ob.value, key, val)
  ob.dep.notify()
  return val
}

```

### vm.$delete

> /src/core/instance/state.js

```javascript
Vue.prototype.$delete = del
```

#### del

> /src/core/observer/index.js


```javascript
/**
 * 通过 Vue.delete 或者 vm.$delete 删除 target 对象的指定 key
 * 数组通过 splice 方法实现，对象则通过 delete 运算符删除指定 key，并执行依赖通知
 */
export function del (target: Array<any> | Object, key: any) {
  if (process.env.NODE_ENV !== 'production' &&
    (isUndef(target) || isPrimitive(target))
  ) {
    warn(`Cannot delete reactive property on undefined, null, or primitive value: ${(target: any)}`)
  }

  // target 为数组，则通过 splice 方法删除指定下标的元素
  if (Array.isArray(target) && isValidArrayIndex(key)) {
    target.splice(key, 1)
    return
  }
  const ob = (target: any).__ob__

  // 避免删除 Vue 实例的属性或者 $data 的数据
  if (target._isVue || (ob && ob.vmCount)) {
    process.env.NODE_ENV !== 'production' && warn(
      'Avoid deleting properties on a Vue instance or its root $data ' +
      '- just set it to null.'
    )
    return
  }
  // 如果属性不存在直接结束
  if (!hasOwn(target, key)) {
    return
  }
  // 通过 delete 运算符删除对象的属性
  delete target[key]
  if (!ob) {
    return
  }
  // 执行依赖通知
  ob.dep.notify()
}

```

### vm.$watch

> /src/core/instance/state.js

```javascript
/**
 * 创建 watcher，返回 unwatch，共完成如下 5 件事：
 *   1、兼容性处理，保证最后 new Watcher 时的 cb 为函数
 *   2、标示用户 watcher
 *   3、创建 watcher 实例
 *   4、如果设置了 immediate，则立即执行一次 cb
 *   5、返回 unwatch
 * @param {*} expOrFn key
 * @param {*} cb 回调函数
 * @param {*} options 配置项，用户直接调用 this.$watch 时可能会传递一个 配置项
 * @returns 返回 unwatch 函数，用于取消 watch 监听
 */
Vue.prototype.$watch = function (
  expOrFn: string | Function,
  cb: any,
  options?: Object
): Function {
  const vm: Component = this
  // 兼容性处理，因为用户调用 vm.$watch 时设置的 cb 可能是对象
  if (isPlainObject(cb)) {
    return createWatcher(vm, expOrFn, cb, options)
  }
  // options.user 表示用户 watcher，还有渲染 watcher，即 updateComponent 方法中实例化的 watcher
  options = options || {}
  options.user = true
  // 创建 watcher
  const watcher = new Watcher(vm, expOrFn, cb, options)
  // 如果用户设置了 immediate 为 true，则立即执行一次回调函数
  if (options.immediate) {
    try {
      cb.call(vm, watcher.value)
    } catch (error) {
      handleError(error, vm, `callback for immediate watcher "${watcher.expression}"`)
    }
  }
  // 返回一个 unwatch 函数，用于解除监听
  return function unwatchFn() {
    watcher.teardown()
  }
}

```

### vm.$on

> /src/core/instance/events.js

```javascript
const hookRE = /^hook:/
/**
 * 监听实例上的自定义事件，vm._event = { eventName: [fn1, ...], ... }
 * @param {*} event 单个的事件名称或者有多个事件名组成的数组
 * @param {*} fn 当 event 被触发时执行的回调函数
 * @returns 
 */
Vue.prototype.$on = function (event: string | Array<string>, fn: Function): Component {
  const vm: Component = this
  if (Array.isArray(event)) {
    // event 是有多个事件名组成的数组，则遍历这些事件，依次递归调用 $on
    for (let i = 0, l = event.length; i < l; i++) {
      vm.$on(event[i], fn)
    }
  } else {
    // 将注册的事件和回调以键值对的形式存储到 vm._event 对象中 vm._event = { eventName: [fn1, ...] }
    (vm._events[event] || (vm._events[event] = [])).push(fn)
    // hookEvent，提供从外部为组件实例注入声明周期方法的机会
    // 比如从组件外部为组件的 mounted 方法注入额外的逻辑
    // 该能力是结合 callhook 方法实现的
    if (hookRE.test(event)) {
      vm._hasHookEvent = true
    }
  }
  return vm
}

```
关于 hookEvent，下一篇文章会详细介绍。

### vm.$emit

> /src/core/instance/events.js

```javascript
/**
 * 触发实例上的指定事件，vm._event[event] => cbs => loop cbs => cb(args)
 * @param {*} event 事件名
 * @returns 
 */
Vue.prototype.$emit = function (event: string): Component {
  const vm: Component = this
  if (process.env.NODE_ENV !== 'production') {
    // 将事件名转换为小些
    const lowerCaseEvent = event.toLowerCase()
    // 意思是说，HTML 属性不区分大小写，所以你不能使用 v-on 监听小驼峰形式的事件名（eventName），而应该使用连字符形式的事件名（event-name)
    if (lowerCaseEvent !== event && vm._events[lowerCaseEvent]) {
      tip(
        `Event "${lowerCaseEvent}" is emitted in component ` +
        `${formatComponentName(vm)} but the handler is registered for "${event}". ` +
        `Note that HTML attributes are case-insensitive and you cannot use ` +
        `v-on to listen to camelCase events when using in-DOM templates. ` +
        `You should probably use "${hyphenate(event)}" instead of "${event}".`
      )
    }
  }
  // 从 vm._event 对象上拿到当前事件的回调函数数组，并一次调用数组中的回调函数，并且传递提供的参数
  let cbs = vm._events[event]
  if (cbs) {
    cbs = cbs.length > 1 ? toArray(cbs) : cbs
    const args = toArray(arguments, 1)
    const info = `event handler for "${event}"`
    for (let i = 0, l = cbs.length; i < l; i++) {
      invokeWithErrorHandling(cbs[i], vm, args, vm, info)
    }
  }
  return vm
}

```

### vm.$off

> /src/core/instance/events.js

```javascript
/**
 * 移除自定义事件监听器，即从 vm._event 对象中找到对应的事件，移除所有事件 或者 移除指定事件的回调函数
 * @param {*} event 
 * @param {*} fn 
 * @returns 
 */
Vue.prototype.$off = function (event?: string | Array<string>, fn?: Function): Component {
  const vm: Component = this
  // vm.$off() 移除实例上的所有监听器 => vm._events = {}
  if (!arguments.length) {
    vm._events = Object.create(null)
    return vm
  }
  // 移除一些事件 event = [event1, ...]，遍历 event 数组，递归调用 vm.$off
  if (Array.isArray(event)) {
    for (let i = 0, l = event.length; i < l; i++) {
      vm.$off(event[i], fn)
    }
    return vm
  }
  // 除了 vm.$off() 之外，最终都会走到这里，移除指定事件
  const cbs = vm._events[event]
  if (!cbs) {
    // 表示没有注册过该事件
    return vm
  }
  if (!fn) {
    // 没有提供 fn 回调函数，则移除该事件的所有回调函数，vm._event[event] = null
    vm._events[event] = null
    return vm
  }
  // 移除指定事件的指定回调函数，就是从事件的回调数组中找到该回调函数，然后删除
  let cb
  let i = cbs.length
  while (i--) {
    cb = cbs[i]
    if (cb === fn || cb.fn === fn) {
      cbs.splice(i, 1)
      break
    }
  }
  return vm
}

```

### vm.$once

> /src/core/instance/events.js

```javascript
/**
 * 监听一个自定义事件，但是只触发一次。一旦触发之后，监听器就会被移除
 * vm.$on + vm.$off
 * @param {*} event 
 * @param {*} fn 
 * @returns 
 */
Vue.prototype.$once = function (event: string, fn: Function): Component {
  const vm: Component = this

  // 调用 $on，只是 $on 的回调函数被特殊处理了，触发时，执行回调函数，先移除事件监听，然后执行你设置的回调函数
  function on() {
    vm.$off(event, on)
    fn.apply(vm, arguments)
  }
  on.fn = fn
  vm.$on(event, on)
  return vm
}

```

### vm._update

> /src/core/instance/lifecycle.js

```javascript
/**
 * 负责更新页面，页面首次渲染和后续更新的入口位置，也是 patch 的入口位置 
 */
Vue.prototype._update = function (vnode: VNode, hydrating?: boolean) {
  const vm: Component = this
  const prevEl = vm.$el
  const prevVnode = vm._vnode
  const restoreActiveInstance = setActiveInstance(vm)
  vm._vnode = vnode
  // Vue.prototype.__patch__ is injected in entry points
  // based on the rendering backend used.
  if (!prevVnode) {
    // 首次渲染，即初始化页面时走这里
    vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */)
  } else {
    // 响应式数据更新时，即更新页面时走这里
    vm.$el = vm.__patch__(prevVnode, vnode)
  }
  restoreActiveInstance()
  // update __vue__ reference
  if (prevEl) {
    prevEl.__vue__ = null
  }
  if (vm.$el) {
    vm.$el.__vue__ = vm
  }
  // if parent is an HOC, update its $el as well
  if (vm.$vnode && vm.$parent && vm.$vnode === vm.$parent._vnode) {
    vm.$parent.$el = vm.$el
  }
  // updated hook is called by the scheduler to ensure that children are
  // updated in a parent's updated hook.
}

```

### vm.$forceUpdate

> /src/core/instance/lifecycle.js

```javascript
/**
 * 直接调用 watcher.update 方法，迫使组件重新渲染。
 * 它仅仅影响实例本身和插入插槽内容的子组件，而不是所有子组件
 */
Vue.prototype.$forceUpdate = function () {
  const vm: Component = this
  if (vm._watcher) {
    vm._watcher.update()
  }
}

```

### vm.$destroy

> /src/core/instance/lifecycle.js

```javascript
/**
 * 完全销毁一个实例。清理它与其它实例的连接，解绑它的全部指令及事件监听器。
 */
Vue.prototype.$destroy = function () {
  const vm: Component = this
  if (vm._isBeingDestroyed) {
    // 表示实例已经销毁
    return
  }
  // 调用 beforeDestroy 钩子
  callHook(vm, 'beforeDestroy')
  // 标识实例已经销毁
  vm._isBeingDestroyed = true
  // 把自己从老爹（$parent)的肚子里（$children）移除
  const parent = vm.$parent
  if (parent && !parent._isBeingDestroyed && !vm.$options.abstract) {
    remove(parent.$children, vm)
  }
  // 移除依赖监听
  if (vm._watcher) {
    vm._watcher.teardown()
  }
  let i = vm._watchers.length
  while (i--) {
    vm._watchers[i].teardown()
  }
  // remove reference from data ob
  // frozen object may not have observer.
  if (vm._data.__ob__) {
    vm._data.__ob__.vmCount--
  }
  // call the last hook...
  vm._isDestroyed = true
  // 调用 __patch__，销毁节点
  vm.__patch__(vm._vnode, null)
  // 调用 destroyed 钩子
  callHook(vm, 'destroyed')
  // 关闭实例的所有事件监听
  vm.$off()
  // remove __vue__ reference
  if (vm.$el) {
    vm.$el.__vue__ = null
  }
  // release circular reference (#6759)
  if (vm.$vnode) {
    vm.$vnode.parent = null
  }
}

```

### vm.$nextTick

> /src/core/instance/render.js

```javascript
Vue.prototype.$nextTick = function (fn: Function) {
  return nextTick(fn, this)
}
```

#### nextTick

> /src/core/util/next-tick.js

```javascript
const callbacks = []
/**
 * 完成两件事：
 *   1、用 try catch 包装 flushSchedulerQueue 函数，然后将其放入 callbacks 数组
 *   2、如果 pending 为 false，表示现在浏览器的任务队列中没有 flushCallbacks 函数
 *     如果 pending 为 true，则表示浏览器的任务队列中已经被放入了 flushCallbacks 函数，
 *     待执行 flushCallbacks 函数时，pending 会被再次置为 false，表示下一个 flushCallbacks 函数可以进入
 *     浏览器的任务队列了
 * pending 的作用：保证在同一时刻，浏览器的任务队列中只有一个 flushCallbacks 函数
 * @param {*} cb 接收一个回调函数 => flushSchedulerQueue
 * @param {*} ctx 上下文
 * @returns 
 */
export function nextTick (cb?: Function, ctx?: Object) {
  let _resolve
  // 用 callbacks 数组存储经过包装的 cb 函数
  callbacks.push(() => {
    if (cb) {
      // 用 try catch 包装回调函数，便于错误捕获
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
    // 执行 timerFunc，在浏览器的任务队列中（首选微任务队列）放入 flushCallbacks 函数
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

### vm._render

> /src/core/instance/render.js

```javascript
/**
 * 通过执行 render 函数生成 VNode
 * 不过里面加了大量的异常处理代码
 */
Vue.prototype._render = function (): VNode {
  const vm: Component = this
  const { render, _parentVnode } = vm.$options

  if (_parentVnode) {
    vm.$scopedSlots = normalizeScopedSlots(
      _parentVnode.data.scopedSlots,
      vm.$slots,
      vm.$scopedSlots
    )
  }

  // 设置父 vnode。这使得渲染函数可以访问占位符节点上的数据。
  vm.$vnode = _parentVnode
  // render self
  let vnode
  try {
    currentRenderingInstance = vm
    // 执行 render 函数，生成 vnode
    vnode = render.call(vm._renderProxy, vm.$createElement)
  } catch (e) {
    handleError(e, vm, `render`)
    // 到这儿，说明执行 render 函数时出错了
    // 开发环境渲染错误信息，生产环境返回之前的 vnode，以防止渲染错误导致组件空白
    /* istanbul ignore else */
    if (process.env.NODE_ENV !== 'production' && vm.$options.renderError) {
      try {
        vnode = vm.$options.renderError.call(vm._renderProxy, vm.$createElement, e)
      } catch (e) {
        handleError(e, vm, `renderError`)
        vnode = vm._vnode
      }
    } else {
      vnode = vm._vnode
    }
  } finally {
    currentRenderingInstance = null
  }
  // 如果返回的 vnode 是数组，并且只包含了一个元素，则直接打平
  if (Array.isArray(vnode) && vnode.length === 1) {
    vnode = vnode[0]
  }
  // render 函数出错时，返回一个空的 vnode
  if (!(vnode instanceof VNode)) {
    if (process.env.NODE_ENV !== 'production' && Array.isArray(vnode)) {
      warn(
        'Multiple root nodes returned from render function. Render function ' +
        'should return a single root node.',
        vm
      )
    }
    vnode = createEmptyVNode()
  }
  // set parent
  vnode.parent = _parentVnode
  return vnode
}

```

### installRenderHelpers

> src/core/instance/render-helpers/index.js

该方法负责在实例上安装大量和渲染相关的简写的工具函数，这些工具函数用在编译器生成的渲染函数中，比如 v-for 编译后的 vm._l，还有大家最熟悉的 h 函数（vm._c)，不过它没在这里声明，是在 initRender 函数中声明的。

installRenderHelpers 方法是在 renderMixin 中被调用的。

```javascript
/**
 * 在实例上挂载简写的渲染工具函数
 * @param {*} target Vue 实例
 */
export function installRenderHelpers (target: any) {
  target._o = markOnce
  target._n = toNumber
  target._s = toString
  target._l = renderList
  target._t = renderSlot
  target._q = looseEqual
  target._i = looseIndexOf
  target._m = renderStatic
  target._f = resolveFilter
  target._k = checkKeyCodes
  target._b = bindObjectProps
  target._v = createTextVNode
  target._e = createEmptyVNode
  target._u = resolveScopedSlots
  target._g = bindObjectListeners
  target._d = bindDynamicKeys
  target._p = prependModifier
}

```
如果对某个方法感兴趣，可以自行深究。

## 总结


* **面试官 问**：vm.$set(obj, key, val) 做了什么？

  **答**：
  
  vm.\$set 用于向响应式对象添加一个新的 property，并确保这个新的 property 同样是响应式的，并触发视图更新。由于 Vue 无法探测对象新增属性或者通过索引为数组新增一个元素，比如：`this.obj.newProperty = 'val'`、`this.arr[3] = 'val'`。所以这才有了 vm.$set，它是 Vue.set 的别名。

  * 为对象添加一个新的响应式数据：调用 defineReactive 方法为对象增加响应式数据，然后执行 dep.notify 进行依赖通知，更新视图

  * 为数组添加一个新的响应式数据：通过 splice 方法实现
  
<hr />

* **面试官 问**：vm.$delete(obj, key)  做了什么？

  **答**：
  
  vm.$delete 用于删除对象上的属性。如果对象是响应式的，且能确保能触发视图更新。该方法主要用于避开 Vue 不能检测属性被删除的情况。它是 Vue.delete 的别名。

  * 删除数组指定下标的元素，内部通过 splice 方法来完成

  * 删除对象上的指定属性，则是先通过 delete 运算符删除该属性，然后执行 dep.notify 进行依赖通知，更新视图

<hr />

* **面试官 问**：vm.$watch(expOrFn, callback, [options]) 做了什么？

  答：

  vm.$watch 负责观察 Vue 实例上的一个表达式或者一个函数计算结果的变化。当其发生变化时，回调函数就会被执行，并为回调函数传递两个参数，第一个为更新后的新值，第二个为老值。
  
  这里需要 **注意** 一点的是：如果观察的是一个对象，比如：数组，当你用数组方法，比如 push 为数组新增一个元素时，回调函数被触发时传递的新值和老值相同，因为它们指向同一个引用，所以在观察一个对象并且在回调函数中有新老值是否相等的判断时需要注意。
  
  vm.$watch 的第一个参数只接收简单的响应式数据的键路径，对于更复杂的表达式建议使用函数作为第一个参数。

  至于 vm.$watch 的内部原理是：

  * 设置 options.user = true，标志是一个用户 watcher

  * 实例化一个 Watcher 实例，当检测到数据更新时，通过 watcher 去触发回调函数的执行，并传递新老值作为回调函数的参数

  * 返回一个 unwatch 函数，用于取消观察

<hr />

* **面试官 问**：vm.$on(event, callback) 做了什么？

  **答**：
  
  监听当前实例上的自定义事件，事件可由 vm.\$emit 触发，回调函数会接收所有传入事件触发函数（vm.$emit）的额外参数。
  
  vm.$on 的原理很简单，就是处理传递的 event 和 callback 两个参数，将注册的事件和回调函数以键值对的形式存储到 vm._event 对象中，vm._events = { eventName: [cb1, cb2, ...], ... }。

<hr />

* **面试官 问**：vm.$emit(eventName, [...args]) 做了什么？

  **答**：

  触发当前实例上的指定事件，附加参数都会传递给事件的回调函数。

  其内部原理就是执行 `vm._events[eventName]` 中所有的回调函数。

  > 备注：从 $on 和 $emit 的实现原理也能看出，组件的自定义事件其实是谁触发谁监听，所以在这会儿再回头看 [Vue 源码解读（2）—— Vue 初始化过程](https://mp.weixin.qq.com/s?__biz=MzA3NTk4NjQ1OQ==&mid=2247484817&idx=1&sn=d6357aed6fd2f7e7cc8e47f8ad79ea96&chksm=9f6966e5a81eeff3eb2ac169a41fba0960e9cc8863c193e8608501a873bead743021d37d71f7#rd) 中关于 initEvent 的解释就会明白在说什么，因为组件自定义事件的处理内部用的就是 vm.$on、vm.$emit。

<hr />

* **面试官 问**：vm.$off([event, callback]) 做了什么？

  **答**：

  移除自定义事件监听器，即移除 vm._events 对象上相关数据。

  * 如果没有提供参数，则移除实例的所有事件监听

  * 如果只提供了 event 参数，则移除实例上该事件的所有监听器

  * 如果两个参数都提供了，则移除实例上该事件对应的监听器

<hr />

* **面试官 问**：vm.$once(event, callback)  做了什么？

  **答**：

  监听一个自定义事件，但是该事件只会被触发一次。一旦触发以后监听器就会被移除。

  其内部的实现原理是：

  * 包装用户传递的回调函数，当包装函数执行的时候，除了会执行用户回调函数之外还会执行 `vm.$off(event, 包装函数)` 移除该事件

  * 用 `vm.$on(event, 包装函数)` 注册事件

<hr />

* **面试官 问**：vm._update(vnode, hydrating)  做了什么？

  **答**：

  官方文档没有说明该 API，这是一个用于源码内部的实例方法，负责更新页面，是页面渲染的入口，其内部根据是否存在 prevVnode 来决定是首次渲染，还是页面更新，从而在调用 \_\_patch\_\_ 函数时传递不同的参数。该方法在业务开发中不会用到。

<hr />

* **面试官 问**：vm.$forceUpdate()  做了什么？

  **答**：

  迫使 Vue 实例重新渲染，它仅仅影响组件实例本身和插入插槽内容的子组件，而不是所有子组件。其内部原理到也简单，就是直接调用 `vm._watcher.update()`，它就是 `watcher.update()` 方法，执行该方法触发组件更新。

<hr />

* **面试官 问**：vm.$destroy()  做了什么？

  **答**：

  负责完全销毁一个实例。清理它与其它实例的连接，解绑它的全部指令和事件监听器。在执行过程中会调用 `beforeDestroy` 和 `destroy` 两个钩子函数。在大多数业务开发场景下用不到该方法，一般都通过 v-if 指令来操作。其内部原理是：

  * 调用 beforeDestroy 钩子函数

  * 将自己从老爹肚子里（$parent）移除，从而销毁和老爹的关系

  * 通过 watcher.teardown() 来移除依赖监听

  * 通过 vm.\_\_patch\_\_(vnode, null) 方法来销毁节点

  * 调用 destroyed 钩子函数

  * 通过 `vm.$off` 方法移除所有的事件监听

<hr />

* **面试官 问**：vm.$nextTick(cb)  做了什么？

  **答**：

  vm.$nextTick 是 Vue.nextTick 的别名，其作用是延迟回调函数 cb 的执行，一般用于 `this.key = newVal` 更改数据后，想立即获取更改过后的 DOM 数据：
  ```javascript
  this.key = 'new val'
  
  Vue.nextTick(function() {
    // DOM 更新了
  })
  ```

  其内部的执行过程是：

  * `this.key = 'new val'`，触发依赖通知更新，将负责更新的 watcher 放入 watcher 队列

  * 将刷新 watcher 队列的函数放到 callbacks 数组中

  * 在浏览器的异步任务队列中放入一个刷新 callbacks 数组的函数

  * **vm.$nextTick(cb)** 来插队，直接将 cb 函数放入 callbacks 数组

  * 待将来的某个时刻执行刷新 callbacks 数组的函数

  * 然后执行 callbacks 数组中的众多函数，触发 watcher.run 的执行，更新 DOM

  * 由于 cb 函数是在后面放到 callbacks 数组，所以这就保证了先完成的 DOM 更新，再执行 cb 函数

<hr />

* **面试官 问**：vm._render  做了什么？

  **答**：

  官方文档没有提供该方法，它是一个用于源码内部的实例方法，负责生成 vnode。其关键代码就一行，执行 render 函数生成 vnode。不过其中加了大量的异常处理代码。

## 链接

* [配套视频，微信公众号回复](https://gitee.com/liyongning/typora-image-bed/raw/master/202202171742614.jpg)："精通 Vue 技术栈源码原理视频版" 获取
* [精通 Vue 技术栈源码原理 专栏](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzA3NTk4NjQ1OQ==&action=getalbum&album_id=2273541436891693065#wechat_redirect)
* [github 仓库 liyongning/Vue](https://github.com/liyongning/Vue) 欢迎 Star



感谢各位的：**关注**、**点赞**、**收藏**和**评论**，我们下期见。

***

**当学习成为了习惯，知识也就变成了常识。** 感谢各位的 **关注**、**点赞**、**收藏**和**评论**。

新视频和文章会第一时间在微信公众号发送，欢迎关注：[李永宁lyn](https://gitee.com/liyongning/typora-image-bed/raw/master/202202171742614.jpg)

文章已收录到 [github 仓库 liyongning/blog](https://github.com/liyongning/blog)，欢迎 Watch 和 Star。