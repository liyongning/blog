# Vue 源码解读（3）—— 响应式原理

**当学习成为了习惯，知识也就变成了常识。** 感谢各位的 **点赞**、**收藏**和**评论**。

新视频和文章会第一时间在微信公众号发送，欢迎关注：[李永宁lyn](https://gitee.com/liyongning/typora-image-bed/raw/master/202202171742614.jpg)

文章已收录到 [github 仓库 liyongning/blog](https://github.com/liyongning/blog)，欢迎 Watch 和 Star。

## 封面

![](https://gitee.com/liyongning/typora-image-bed/raw/master/202202221351137.png)

## 前言

上一篇文章 [Vue 源码解读（2）—— Vue 初始化过程](https://mp.weixin.qq.com/s/KwYkA_k-RRYeFtgn6vyWpg) 详细讲解了 Vue 的初始化过程，明白了 `new Vue(options)` 都做了什么，其中关于 `数据响应式` 的实现用一句话简单的带过，而这篇文章则会详细讲解 Vue 数据响应式的实现原理。

## 目标

* 深入理解 Vue 数据响应式原理。

* methods、computed 和 watch 有什么区别？

## 源码解读

经过上一篇文章的学习，相信关于 `响应式原理` 源码阅读的入口位置大家都已经知道了，就是初始化过程中处理数据响应式这一步，即调用 `initState` 方法，在 `/src/core/instance/init.js` 文件中。

### initState

> /src/core/instance/state.js

```javascript
/**
 * 两件事：
 *   数据响应式的入口：分别处理 props、methods、data、computed、watch
 *   优先级：props、methods、data、computed 对象中的属性不能出现重复，优先级和列出顺序一致
 *         其中 computed 中的 key 不能和 props、data 中的 key 重复，methods 不影响
 */
export function initState (vm: Component) {
  vm._watchers = []
  const opts = vm.$options
  // 处理 props 对象，为 props 对象的每个属性设置响应式，并将其代理到 vm 实例上
  if (opts.props) initProps(vm, opts.props)
  // 处理 methos 对象，校验每个属性的值是否为函数、和 props 属性比对进行判重处理，最后得到 vm[key] = methods[key]
  if (opts.methods) initMethods(vm, opts.methods)
  /**
   * 做了三件事
   *   1、判重处理，data 对象上的属性不能和 props、methods 对象上的属性相同
   *   2、代理 data 对象上的属性到 vm 实例
   *   3、为 data 对象的上数据设置响应式 
   */
  if (opts.data) {
    initData(vm)
  } else {
    observe(vm._data = {}, true /* asRootData */)
  }
  /**
   * 三件事：
   *   1、为 computed[key] 创建 watcher 实例，默认是懒执行
   *   2、代理 computed[key] 到 vm 实例
   *   3、判重，computed 中的 key 不能和 data、props 中的属性重复
   */
  if (opts.computed) initComputed(vm, opts.computed)
  /**
   * 三件事：
   *   1、处理 watch 对象
   *   2、为 每个 watch.key 创建 watcher 实例，key 和 watcher 实例可能是 一对多 的关系
   *   3、如果设置了 immediate，则立即执行 回调函数
   */
  if (opts.watch && opts.watch !== nativeWatch) {
    initWatch(vm, opts.watch)
  }
    
  /**
   * 其实到这里也能看出，computed 和 watch 在本质是没有区别的，都是通过 watcher 去实现的响应式
   * 非要说有区别，那也只是在使用方式上的区别，简单来说：
   *   1、watch：适用于当数据变化时执行异步或者开销较大的操作时使用，即需要长时间等待的操作可以放在 watch 中
   *   2、computed：其中可以使用异步方法，但是没有任何意义。所以 computed 更适合做一些同步计算
   */
}

```

### initProps

> src/core/instance/state.js

```javascript
// 处理 props 对象，为 props 对象的每个属性设置响应式，并将其代理到 vm 实例上
function initProps (vm: Component, propsOptions: Object) {
  const propsData = vm.$options.propsData || {}
  const props = vm._props = {}
  // 缓存 props 的每个 key，性能优化
  // cache prop keys so that future props updates can iterate using Array
  // instead of dynamic object key enumeration.
  const keys = vm.$options._propKeys = []
  const isRoot = !vm.$parent
  // root instance props should be converted
  if (!isRoot) {
    toggleObserving(false)
  }
  // 遍历 props 对象
  for (const key in propsOptions) {
    // 缓存 key
    keys.push(key)
    // 获取 props[key] 的默认值
    const value = validateProp(key, propsOptions, propsData, vm)
    // 为 props 的每个 key 是设置数据响应式
    defineReactive(props, key, value)
    // static props are already proxied on the component's prototype
    // during Vue.extend(). We only need to proxy props defined at
    // instantiation here.
    if (!(key in vm)) {
      // 代理 key 到 vm 对象上
      proxy(vm, `_props`, key)
    }
  }
  toggleObserving(true)
}

```

### proxy

> /src/core/instance/state.js

```javascript
// 设置代理，将 key 代理到 target 上
export function proxy (target: Object, sourceKey: string, key: string) {
  sharedPropertyDefinition.get = function proxyGetter () {
    return this[sourceKey][key]
  }
  sharedPropertyDefinition.set = function proxySetter (val) {
    this[sourceKey][key] = val
  }
  Object.defineProperty(target, key, sharedPropertyDefinition)
}

```

### initMethods

> /src/core/instance/state.js

```javascript
/**
 * 做了以下三件事，其实最关键的就是第三件事情
 *   1、校验 methoss[key]，必须是一个函数
 *   2、判重
 *         methods 中的 key 不能和 props 中的 key 相同
 *         methos 中的 key 与 Vue 实例上已有的方法重叠，一般是一些内置方法，比如以 $ 和 _ 开头的方法
 *   3、将 methods[key] 放到 vm 实例上，得到 vm[key] = methods[key]
 */
function initMethods (vm: Component, methods: Object) {
  // 获取 props 配置项
  const props = vm.$options.props
  // 遍历 methods 对象
  for (const key in methods) {
    if (process.env.NODE_ENV !== 'production') {
      if (typeof methods[key] !== 'function') {
        warn(
          `Method "${key}" has type "${typeof methods[key]}" in the component definition. ` +
          `Did you reference the function correctly?`,
          vm
        )
      }
      if (props && hasOwn(props, key)) {
        warn(
          `Method "${key}" has already been defined as a prop.`,
          vm
        )
      }
      if ((key in vm) && isReserved(key)) {
        warn(
          `Method "${key}" conflicts with an existing Vue instance method. ` +
          `Avoid defining component methods that start with _ or $.`
        )
      }
    }
    vm[key] = typeof methods[key] !== 'function' ? noop : bind(methods[key], vm)
  }
}

```

### initData

> src/core/instance/state.js

```javascript
/**
 * 做了三件事
 *   1、判重处理，data 对象上的属性不能和 props、methods 对象上的属性相同
 *   2、代理 data 对象上的属性到 vm 实例
 *   3、为 data 对象的上数据设置响应式 
 */
function initData (vm: Component) {
  // 得到 data 对象
  let data = vm.$options.data
  data = vm._data = typeof data === 'function'
    ? getData(data, vm)
    : data || {}
  if (!isPlainObject(data)) {
    data = {}
    process.env.NODE_ENV !== 'production' && warn(
      'data functions should return an object:\n' +
      'https://vuejs.org/v2/guide/components.html#data-Must-Be-a-Function',
      vm
    )
  }
  /**
   * 两件事
   *   1、判重处理，data 对象上的属性不能和 props、methods 对象上的属性相同
   *   2、代理 data 对象上的属性到 vm 实例
   */
  const keys = Object.keys(data)
  const props = vm.$options.props
  const methods = vm.$options.methods
  let i = keys.length
  while (i--) {
    const key = keys[i]
    if (process.env.NODE_ENV !== 'production') {
      if (methods && hasOwn(methods, key)) {
        warn(
          `Method "${key}" has already been defined as a data property.`,
          vm
        )
      }
    }
    if (props && hasOwn(props, key)) {
      process.env.NODE_ENV !== 'production' && warn(
        `The data property "${key}" is already declared as a prop. ` +
        `Use prop default value instead.`,
        vm
      )
    } else if (!isReserved(key)) {
      proxy(vm, `_data`, key)
    }
  }
  // 为 data 对象上的数据设置响应式
  observe(data, true /* asRootData */)
}

export function getData (data: Function, vm: Component): any {
  // #7573 disable dep collection when invoking data getters
  pushTarget()
  try {
    return data.call(vm, vm)
  } catch (e) {
    handleError(e, vm, `data()`)
    return {}
  } finally {
    popTarget()
  }
}

```

### initComputed

> /src/core/instance/state.js

```javascript
const computedWatcherOptions = { lazy: true }

/**
 * 三件事：
 *   1、为 computed[key] 创建 watcher 实例，默认是懒执行
 *   2、代理 computed[key] 到 vm 实例
 *   3、判重，computed 中的 key 不能和 data、props 中的属性重复
 * @param {*} computed = {
 *   key1: function() { return xx },
 *   key2: {
 *     get: function() { return xx },
 *     set: function(val) {}
 *   }
 * }
 */
function initComputed (vm: Component, computed: Object) {
  // $flow-disable-line
  const watchers = vm._computedWatchers = Object.create(null)
  // computed properties are just getters during SSR
  const isSSR = isServerRendering()

  // 遍历 computed 对象
  for (const key in computed) {
    // 获取 key 对应的值，即 getter 函数
    const userDef = computed[key]
    const getter = typeof userDef === 'function' ? userDef : userDef.get
    if (process.env.NODE_ENV !== 'production' && getter == null) {
      warn(
        `Getter is missing for computed property "${key}".`,
        vm
      )
    }

    if (!isSSR) {
      // 为 computed 属性创建 watcher 实例
      watchers[key] = new Watcher(
        vm,
        getter || noop,
        noop,
        // 配置项，computed 默认是懒执行
        computedWatcherOptions
      )
    }

    if (!(key in vm)) {
      // 代理 computed 对象中的属性到 vm 实例
      // 这样就可以使用 vm.computedKey 访问计算属性了
      defineComputed(vm, key, userDef)
    } else if (process.env.NODE_ENV !== 'production') {
      // 非生产环境有一个判重处理，computed 对象中的属性不能和 data、props 中的属性相同
      if (key in vm.$data) {
        warn(`The computed property "${key}" is already defined in data.`, vm)
      } else if (vm.$options.props && key in vm.$options.props) {
        warn(`The computed property "${key}" is already defined as a prop.`, vm)
      }
    }
  }
}

/**
 * 代理 computed 对象中的 key 到 target（vm）上
 */
export function defineComputed (
  target: any,
  key: string,
  userDef: Object | Function
) {
  const shouldCache = !isServerRendering()
  // 构造属性描述符(get、set)
  if (typeof userDef === 'function') {
    sharedPropertyDefinition.get = shouldCache
      ? createComputedGetter(key)
      : createGetterInvoker(userDef)
    sharedPropertyDefinition.set = noop
  } else {
    sharedPropertyDefinition.get = userDef.get
      ? shouldCache && userDef.cache !== false
        ? createComputedGetter(key)
        : createGetterInvoker(userDef.get)
      : noop
    sharedPropertyDefinition.set = userDef.set || noop
  }
  if (process.env.NODE_ENV !== 'production' &&
      sharedPropertyDefinition.set === noop) {
    sharedPropertyDefinition.set = function () {
      warn(
        `Computed property "${key}" was assigned to but it has no setter.`,
        this
      )
    }
  }
  // 拦截对 target.key 的访问和设置
  Object.defineProperty(target, key, sharedPropertyDefinition)
}

/**
 * @returns 返回一个函数，这个函数在访问 vm.computedProperty 时会被执行，然后返回执行结果
 */
function createComputedGetter (key) {
  // computed 属性值会缓存的原理也是在这里结合 watcher.dirty、watcher.evalaute、watcher.update 实现的
  return function computedGetter () {
    // 得到当前 key 对应的 watcher
    const watcher = this._computedWatchers && this._computedWatchers[key]
    if (watcher) {
      // 计算 key 对应的值，通过执行 computed.key 的回调函数来得到
      // watcher.dirty 属性就是大家常说的 computed 计算结果会缓存的原理
      // <template>
      //   <div>{{ computedProperty }}</div>
      //   <div>{{ computedProperty }}</div>
      // </template>
      // 像这种情况下，在页面的一次渲染中，两个 dom 中的 computedProperty 只有第一个
      // 会执行 computed.computedProperty 的回调函数计算实际的值，
      // 即执行 watcher.evalaute，而第二个就不走计算过程了，
      // 因为上一次执行 watcher.evalute 时把 watcher.dirty 置为了 false，
      // 待页面更新后，wathcer.update 方法会将 watcher.dirty 重新置为 true，
      // 供下次页面更新时重新计算 computed.key 的结果
      if (watcher.dirty) {
        watcher.evaluate()
      }
      if (Dep.target) {
        watcher.depend()
      }
      return watcher.value
    }
  }
}

/**
 * 功能同 createComputedGetter 一样
 */
function createGetterInvoker(fn) {
  return function computedGetter () {
    return fn.call(this, this)
  }
}

```

### initWatch

> /src/core/instance/state.js

```javascript
/**
 * 处理 watch 对象的入口，做了两件事：
 *   1、遍历 watch 对象
 *   2、调用 createWatcher 函数
 * @param {*} watch = {
 *   'key1': function(val, oldVal) {},
 *   'key2': 'this.methodName',
 *   'key3': {
 *     handler: function(val, oldVal) {},
 *     deep: true
 *   },
 *   'key4': [
 *     'this.methodNanme',
 *     function handler1() {},
 *     {
 *       handler: function() {},
 *       immediate: true
 *     }
 *   ],
 *   'key.key5' { ... }
 * }
 */
function initWatch (vm: Component, watch: Object) {
  // 遍历 watch 对象
  for (const key in watch) {
    const handler = watch[key]
    if (Array.isArray(handler)) {
      // handler 为数组，遍历数组，获取其中的每一项，然后调用 createWatcher
      for (let i = 0; i < handler.length; i++) {
        createWatcher(vm, key, handler[i])
      }
    } else {
      createWatcher(vm, key, handler)
    }
  }
}

/**
 * 两件事：
 *   1、兼容性处理，保证 handler 肯定是一个函数
 *   2、调用 $watch 
 * @returns 
 */
function createWatcher (
  vm: Component,
  expOrFn: string | Function,
  handler: any,
  options?: Object
) {
  // 如果 handler 为对象，则获取其中的 handler 选项的值
  if (isPlainObject(handler)) {
    options = handler
    handler = handler.handler
  }
  // 如果 hander 为字符串，则说明是一个 methods 方法，获取 vm[handler]
  if (typeof handler === 'string') {
    handler = vm[handler]
  }
  return vm.$watch(expOrFn, handler, options)
}

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
  return function unwatchFn () {
    watcher.teardown()
  }
}
```

### observe

> /src/core/observer/index.js

```javascript
/**
 * 响应式处理的真正入口
 * 为对象创建观察者实例，如果对象已经被观察过，则返回已有的观察者实例，否则创建新的观察者实例
 * @param {*} value 对象 => {}
 */
export function observe (value: any, asRootData: ?boolean): Observer | void {
  // 非对象和 VNode 实例不做响应式处理
  if (!isObject(value) || value instanceof VNode) {
    return
  }
  let ob: Observer | void
  if (hasOwn(value, '__ob__') && value.__ob__ instanceof Observer) {
    // 如果 value 对象上存在 __ob__ 属性，则表示已经做过观察了，直接返回 __ob__ 属性
    ob = value.__ob__
  } else if (
    shouldObserve &&
    !isServerRendering() &&
    (Array.isArray(value) || isPlainObject(value)) &&
    Object.isExtensible(value) &&
    !value._isVue
  ) {
    // 创建观察者实例
    ob = new Observer(value)
  }
  if (asRootData && ob) {
    ob.vmCount++
  }
  return ob
}

```

### Observer

> /src/core/observer/index.js

```javascript
/**
 * 观察者类，会被附加到每个被观察的对象上，value.__ob__ = this
 * 而对象的各个属性则会被转换成 getter/setter，并收集依赖和通知更新
 */
export class Observer {
  value: any;
  dep: Dep;
  vmCount: number; // number of vms that have this object as root $data

  constructor (value: any) {
    this.value = value
    // 实例话一个 dep
    this.dep = new Dep()
    this.vmCount = 0
    // 在 value 对象上设置 __ob__ 属性
    def(value, '__ob__', this)
    if (Array.isArray(value)) {
      /**
       * value 为数组
       * hasProto = '__proto__' in {}
       * 用于判断对象是否存在 __proto__ 属性，通过 obj.__proto__ 可以访问对象的原型链
       * 但由于 __proto__ 不是标准属性，所以有些浏览器不支持，比如 IE6-10，Opera10.1
       * 为什么要判断，是因为一会儿要通过 __proto__ 操作数据的原型链
       * 覆盖数组默认的七个原型方法，以实现数组响应式
       */
      if (hasProto) {
        // 有 __proto__
        protoAugment(value, arrayMethods)
      } else {
        copyAugment(value, arrayMethods, arrayKeys)
      }
      this.observeArray(value)
    } else {
      // value 为对象，为对象的每个属性（包括嵌套对象）设置响应式
      this.walk(value)
    }
  }

  /**
   * 遍历对象上的每个 key，为每个 key 设置响应式
   * 仅当值为对象时才会走这里
   */
  walk (obj: Object) {
    const keys = Object.keys(obj)
    for (let i = 0; i < keys.length; i++) {
      defineReactive(obj, keys[i])
    }
  }

  /**
   * 遍历数组，为数组的每一项设置观察，处理数组元素为对象的情况
   */
  observeArray (items: Array<any>) {
    for (let i = 0, l = items.length; i < l; i++) {
      observe(items[i])
    }
  }
}

```

### defineReactive

> /src/core/observer/index.js

```javascript
/**
 * 拦截 obj[key] 的读取和设置操作：
 *   1、在第一次读取时收集依赖，比如执行 render 函数生成虚拟 DOM 时会有读取操作
 *   2、在更新时设置新值并通知依赖更新
 */
export function defineReactive (
  obj: Object,
  key: string,
  val: any,
  customSetter?: ?Function,
  shallow?: boolean
) {
  // 实例化 dep，一个 key 一个 dep
  const dep = new Dep()

  // 获取 obj[key] 的属性描述符，发现它是不可配置对象的话直接 return
  const property = Object.getOwnPropertyDescriptor(obj, key)
  if (property && property.configurable === false) {
    return
  }

  // 记录 getter 和 setter，获取 val 值
  const getter = property && property.get
  const setter = property && property.set
  if ((!getter || setter) && arguments.length === 2) {
    val = obj[key]
  }

  // 递归调用，处理 val 即 obj[key] 的值为对象的情况，保证对象中的所有 key 都被观察
  let childOb = !shallow && observe(val)
  // 响应式核心
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    // get 拦截对 obj[key] 的读取操作
    get: function reactiveGetter () {
      const value = getter ? getter.call(obj) : val
      /**
       * Dep.target 为 Dep 类的一个静态属性，值为 watcher，在实例化 Watcher 时会被设置
       * 实例化 Watcher 时会执行 new Watcher 时传递的回调函数（computed 除外，因为它懒执行）
       * 而回调函数中如果有 vm.key 的读取行为，则会触发这里的 读取 拦截，进行依赖收集
       * 回调函数执行完以后又会将 Dep.target 设置为 null，避免这里重复收集依赖
       */
      if (Dep.target) {
        // 依赖收集，在 dep 中添加 watcher，也在 watcher 中添加 dep
        dep.depend()
        // childOb 表示对象中嵌套对象的观察者对象，如果存在也对其进行依赖收集
        if (childOb) {
          // 这就是 this.key.chidlKey 被更新时能触发响应式更新的原因
          childOb.dep.depend()
          // 如果是 obj[key] 是 数组，则触发数组响应式
          if (Array.isArray(value)) {
            // 为数组项为对象的项添加依赖
            dependArray(value)
          }
        }
      }
      return value
    },
    // set 拦截对 obj[key] 的设置操作
    set: function reactiveSetter (newVal) {
      // 旧的 obj[key]
      const value = getter ? getter.call(obj) : val
      // 如果新老值一样，则直接 return，不跟新更不触发响应式更新过程
      /* eslint-disable no-self-compare */
      if (newVal === value || (newVal !== newVal && value !== value)) {
        return
      }
      /* eslint-enable no-self-compare */
      if (process.env.NODE_ENV !== 'production' && customSetter) {
        customSetter()
      }
      // setter 不存在说明该属性是一个只读属性，直接 return
      // #7981: for accessor properties without setter
      if (getter && !setter) return
      // 设置新值
      if (setter) {
        setter.call(obj, newVal)
      } else {
        val = newVal
      }
      // 对新值进行观察，让新值也是响应式的
      childOb = !shallow && observe(newVal)
      // 依赖通知更新
      dep.notify()
    }
  })
}

```

#### dependArray

> /src/core/observer/index.js

```javascript
/**
 * 遍历每个数组元素，递归处理数组项为对象的情况，为其添加依赖
 * 因为前面的递归阶段无法为数组中的对象元素添加依赖
 */
function dependArray (value: Array<any>) {
  for (let e, i = 0, l = value.length; i < l; i++) {
    e = value[i]
    e && e.__ob__ && e.__ob__.dep.depend()
    if (Array.isArray(e)) {
      dependArray(e)
    }
  }
}

```

### 数组响应式

> src/core/observer/array.js

```javascript
/**
 * 定义 arrayMethods 对象，用于增强 Array.prototype
 * 当访问 arrayMethods 对象上的那七个方法时会被拦截，以实现数组响应式
 */
import { def } from '../util/index'

// 备份 数组 原型对象
const arrayProto = Array.prototype
// 通过继承的方式创建新的 arrayMethods
export const arrayMethods = Object.create(arrayProto)

// 操作数组的七个方法，这七个方法可以改变数组自身
const methodsToPatch = [
  'push',
  'pop',
  'shift',
  'unshift',
  'splice',
  'sort',
  'reverse'
]

/**
 * 拦截变异方法并触发事件
 */
methodsToPatch.forEach(function (method) {
  // cache original method
  // 缓存原生方法，比如 push
  const original = arrayProto[method]
  // def 就是 Object.defineProperty，拦截 arrayMethods.method 的访问
  def(arrayMethods, method, function mutator (...args) {
    // 先执行原生方法，比如 push.apply(this, args)
    const result = original.apply(this, args)
    const ob = this.__ob__
    // 如果 method 是以下三个之一，说明是新插入了元素
    let inserted
    switch (method) {
      case 'push':
      case 'unshift':
        inserted = args
        break
      case 'splice':
        inserted = args.slice(2)
        break
    }
    // 对新插入的元素做响应式处理
    if (inserted) ob.observeArray(inserted)
    // 通知更新
    ob.dep.notify()
    return result
  })
})


```

#### def

> /src/core/util/lang.js

```javascript
/**
 * Define a property.
 */
export function def (obj: Object, key: string, val: any, enumerable?: boolean) {
  Object.defineProperty(obj, key, {
    value: val,
    enumerable: !!enumerable,
    writable: true,
    configurable: true
  })
}

```

#### protoAugment

> /src/core/observer/index.js

```javascript
/**
 * 设置 target.__proto__ 的原型对象为 src
 * 比如 数组对象，arr.__proto__ = arrayMethods
 */
function protoAugment (target, src: Object) {
  /* eslint-disable no-proto */
  target.__proto__ = src
  /* eslint-enable no-proto */
}

```

#### copyAugment

> /src/core/observer/index.js

```javascript
/**
 * 在目标对象上定义指定属性
 * 比如数组：为数组对象定义那七个方法
 */
function copyAugment (target: Object, src: Object, keys: Array<string>) {
  for (let i = 0, l = keys.length; i < l; i++) {
    const key = keys[i]
    def(target, key, src[key])
  }
}

```

### Dep

> /src/core/observer/dep.js

```javascript
import type Watcher from './watcher'
import { remove } from '../util/index'
import config from '../config'

let uid = 0

/**
 * 一个 dep 对应一个 obj.key
 * 在读取响应式数据时，负责收集依赖，每个 dep（或者说 obj.key）依赖的 watcher 有哪些
 * 在响应式数据更新时，负责通知 dep 中那些 watcher 去执行 update 方法
 */
export default class Dep {
  static target: ?Watcher;
  id: number;
  subs: Array<Watcher>;

  constructor () {
    this.id = uid++
    this.subs = []
  }

  // 在 dep 中添加 watcher
  addSub (sub: Watcher) {
    this.subs.push(sub)
  }

  removeSub (sub: Watcher) {
    remove(this.subs, sub)
  }

  // 像 watcher 中添加 dep
  depend () {
    if (Dep.target) {
      Dep.target.addDep(this)
    }
  }

  /**
   * 通知 dep 中的所有 watcher，执行 watcher.update() 方法
   */
  notify () {
    // stabilize the subscriber list first
    const subs = this.subs.slice()
    if (process.env.NODE_ENV !== 'production' && !config.async) {
      // subs aren't sorted in scheduler if not running async
      // we need to sort them now to make sure they fire in correct
      // order
      subs.sort((a, b) => a.id - b.id)
    }
    // 遍历 dep 中存储的 watcher，执行 watcher.update()
    for (let i = 0, l = subs.length; i < l; i++) {
      subs[i].update()
    }
  }
}

/**
 * 当前正在执行的 watcher，同一时间只会有一个 watcher 在执行
 * Dep.target = 当前正在执行的 watcher
 * 通过调用 pushTarget 方法完成赋值，调用 popTarget 方法完成重置（null)
 */
Dep.target = null
const targetStack = []

// 在需要进行依赖收集的时候调用，设置 Dep.target = watcher
export function pushTarget (target: ?Watcher) {
  targetStack.push(target)
  Dep.target = target
}

// 依赖收集结束调用，设置 Dep.target = null
export function popTarget () {
  targetStack.pop()
  Dep.target = targetStack[targetStack.length - 1]
}

```

### Watcher

> /src/core/observer/watcher.js

```javascript
/**
 * 一个组件一个 watcher（渲染 watcher）或者一个表达式一个 watcher（用户watcher）
 * 当数据更新时 watcher 会被触发，访问 this.computedProperty 时也会触发 watcher
 */
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
      // this.getter = function() { return this.xx }
      // 在 this.get 中执行 this.getter 时会触发依赖收集
      // 待后续 this.xx 更新时就会触发响应式
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
   * 执行 this.getter，并重新收集依赖
   * this.getter 是实例化 watcher 时传递的第二个参数，一个函数或者字符串，比如：updateComponent 或者 parsePath 返回的读取 this.xx 属性值的函数
   * 为什么要重新收集依赖？
   *   因为触发更新说明有响应式数据被更新了，但是被更新的数据虽然已经经过 observe 观察了，但是却没有进行依赖收集，
   *   所以，在更新页面时，会重新执行一次 render 函数，执行期间会触发读取操作，这时候进行依赖收集
   */
  get () {
    // 打开 Dep.target，Dep.target = this
    pushTarget(this)
    // value 为回调函数执行的结果
    let value
    const vm = this.vm
    try {
      // 执行回调函数，比如 updateComponent，进入 patch 阶段
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
      // 关闭 Dep.target，Dep.target = null
      popTarget()
      this.cleanupDeps()
    }
    return value
  }

  /**
   * Add a dependency to this directive.
   * 两件事：
   *   1、添加 dep 给自己（watcher）
   *   2、添加自己（watcher）到 dep
   */
  addDep (dep: Dep) {
    // 判重，如果 dep 已经存在则不重复添加
    const id = dep.id
    if (!this.newDepIds.has(id)) {
      // 缓存 dep.id，用于判重
      this.newDepIds.add(id)
      // 添加 dep
      this.newDeps.push(dep)
      // 避免在 dep 中重复添加 watcher，this.depIds 的设置在 cleanupDeps 方法中
      if (!this.depIds.has(id)) {
        // 添加 watcher 自己到 dep
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
   * 根据 watcher 配置项，决定接下来怎么走，一般是 queueWatcher
   */
  update () {
    /* istanbul ignore else */
    if (this.lazy) {
      // 懒执行时走这里，比如 computed

      // 将 dirty 置为 true，可以让 computedGetter 执行时重新计算 computed 回调函数的执行结果
      this.dirty = true
    } else if (this.sync) {
      // 同步执行，在使用 vm.$watch 或者 watch 选项时可以传一个 sync 选项，
      // 当为 true 时在数据更新时该 watcher 就不走异步更新队列，直接执行 this.run 
      // 方法进行更新
      // 这个属性在官方文档中没有出现
      this.run()
    } else {
      // 更新时一般都这里，将 watcher 放入 watcher 队列
      queueWatcher(this)
    }
  }

  /**
   * 由 刷新队列函数 flushSchedulerQueue 调用，完成如下几件事：
   *   1、执行实例化 watcher 传递的第二个参数，updateComponent 或者 获取 this.xx 的一个函数(parsePath 返回的函数)
   *   2、更新旧值为新值
   *   3、执行实例化 watcher 时传递的第三个参数，比如用户 watcher 的回调函数
   */
  run () {
    if (this.active) {
      // 调用 this.get 方法
      const value = this.get()
      if (
        value !== this.value ||
        // Deep watchers and watchers on Object/Arrays should fire even
        // when the value is the same, because the value may
        // have mutated.
        isObject(value) ||
        this.deep
      ) {
        // 更新旧值为新值
        const oldValue = this.value
        this.value = value

        if (this.user) {
          // 如果是用户 watcher，则执行用户传递的第三个参数 —— 回调函数，参数为 val 和 oldVal
          try {
            this.cb.call(this.vm, value, oldValue)
          } catch (e) {
            handleError(e, this.vm, `callback for watcher "${this.expression}"`)
          }
        } else {
          // 渲染 watcher，this.cb = noop，一个空函数
          this.cb.call(this.vm, value, oldValue)
        }
      }
    }
  }

  /**
   * 懒执行的 watcher 会调用该方法
   *   比如：computed，在获取 vm.computedProperty 的值时会调用该方法
   * 然后执行 this.get，即 watcher 的回调函数，得到返回值
   * this.dirty 被置为 false，作用是页面在本次渲染中只会一次 computed.key 的回调函数，
   *   这也是大家常说的 computed 和 methods 区别之一是 computed 有缓存的原理所在
   * 而页面更新后会 this.dirty 会被重新置为 true，这一步是在 this.update 方法中完成的
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

## 总结

**面试官 问**：Vue 响应式原理是怎么实现的？

**答**：

* 响应式的核心是通过 `Object.defineProperty` 拦截对数据的访问和设置

* 响应式的数据分为两类：

  * 对象，循环遍历对象的所有属性，为每个属性设置 getter、setter，以达到拦截访问和设置的目的，如果属性值依旧为对象，则递归为属性值上的每个 key 设置 getter、setter
  
    * 访问数据时（obj.key)进行依赖收集，在 dep 中存储相关的 watcher
    
    * 设置数据时由 dep 通知相关的 watcher 去更新
  
  * 数组，增强数组的那 7 个可以更改自身的原型方法，然后拦截对这些方法的操作
  
    * 添加新数据时进行响应式处理，然后由 dep 通知 watcher 去更新
    
    * 删除数据时，也要由 dep 通知 watcher 去更新
    

**面试官 问**：methods、computed 和 watch 有什么区别？

**答**：

```html
<!DOCTYPE html>
<html lang="en">

<head>
  <title>methods、computed、watch 有什么区别</title>
</head>

<body>
  <div id="app">
    <!-- methods -->
    <div>{{ returnMsg() }}</div>
    <div>{{ returnMsg() }}</div>
    <!-- computed -->
    <div>{{ getMsg }}</div>
    <div>{{ getMsg }}</div>
  </div>
  <script src="../../dist/vue.js"></script>
  <script>
    new Vue({
    el: '#app',
    data: {
      msg: 'test'
    },
    mounted() {
      setTimeout(() => {
        this.msg = 'msg is changed'
      }, 1000)
    },
    methods: {
      returnMsg() {
        console.log('methods: returnMsg')
        return this.msg
      }
    },
    computed: {
      getMsg() {
        console.log('computed: getMsg')
        return this.msg + ' hello computed'
      }
    },
    watch: {
      msg: function(val, oldVal) {
        console.log('watch: msg')
        new Promise(resolve => {
          setTimeout(() => {
            this.msg = 'msg is changed by watch'
          }, 1000)
        })
      }
    }
  })
  </script>
</body>

</html>

```

[点击查看动图演示](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9c957654bb484ae7ba4ace1b912cff03~tplv-k3u1fbpfcp-watermark.awebp)，动图地址：https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9c957654bb484ae7ba4ace1b912cff03~tplv-k3u1fbpfcp-watermark.awebp

示例其实就是答案了

* 使用场景

  * methods 一般用于封装一些较为复杂的处理逻辑（同步、异步）
  
  * computed 一般用于封装一些简单的同步逻辑，将经过处理的数据返回，然后显示在模版中，以减轻模版的重量
  
  * watch 一般用于当需要在数据变化时执行异步或开销较大的操作

* 区别

  * methods VS computed
  
    > 通过示例会发现，如果在一次渲染中，有多个地方使用了同一个 methods 或 computed 属性，methods 会被执行多次，而 computed 的回调函数则只会被执行一次。
    
    > 通过阅读源码我们知道，在一次渲染中，多次访问 computedProperty，只会在第一次执行 computed 属性的回调函数，后续的其它访问，则直接使用第一次的执行结果（watcher.value），而这一切的实现原理则是通过对 watcher.dirty 属性的控制实现的。而 methods，每一次的访问则是简单的方法调用（this.xxMethods）。
  
  * computed VS watch
  
    > 通过阅读源码我们知道，computed 和 watch 的本质是一样的，内部都是通过 Watcher 来实现的，其实没什么区别，非要说区别的化就两点：1、使用场景上的区别，2、computed 默认是懒执行的，切不可更改。
  
  * methods VS watch
  
    > methods 和 watch 之间其实没什么可比的，完全是两个东西，不过在使用上可以把 watch 中一些逻辑抽到 methods 中，提高代码的可读性。
    
## 链接

* [配套视频，关注微信公众号回复](https://gitee.com/liyongning/typora-image-bed/raw/master/202202171742614.jpg)："精通 Vue 技术栈源码原理视频版" 获取
* [精通 Vue 技术栈源码原理 专栏](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzA3NTk4NjQ1OQ==&action=getalbum&album_id=2273541436891693065#wechat_redirect)
* [github 仓库 liyongning/Vue](https://github.com/liyongning/Vue) 欢迎 Star



感谢各位的：**点赞**、**收藏**和**评论**，我们下期见。

***

**当学习成为了习惯，知识也就变成了常识。** 感谢各位的 **点赞**、**收藏**和**评论**。

新视频和文章会第一时间在微信公众号发送，欢迎关注：[李永宁lyn](https://gitee.com/liyongning/typora-image-bed/raw/master/202202171742614.jpg)

文章已收录到 [github 仓库 liyongning/blog](https://github.com/liyongning/blog)，欢迎 Watch 和 Star。

