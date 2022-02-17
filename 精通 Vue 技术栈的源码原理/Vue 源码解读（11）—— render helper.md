# Vue 源码解读（11）—— render helper

**当学习成为了习惯，知识也就变成了常识。** 感谢各位的 **关注**、**点赞**、**收藏**和**评论**。

新视频和文章会第一时间在微信公众号发送，欢迎关注：[李永宁lyn](https://gitee.com/liyongning/typora-image-bed/raw/master/202202171742614.jpg)

文章已收录到 [github 仓库 liyongning/blog](https://github.com/liyongning/blog)，欢迎 Watch 和 Star。

## 封面

![](https://gitee.com/liyongning/typora-image-bed/raw/master/202203041203968.png)

## 前言

上一篇文章 [Vue 源码解读（10）—— 编译器 之 生成渲染函数](https://mp.weixin.qq.com/s?__biz=MzA3NTk4NjQ1OQ==&mid=2247485133&idx=1&sn=deccbdabb8a564b5f0da7f0d5e0646a2&chksm=9f6965b9a81eecafa871f2e697dc2d95a47bc2f09dcd2a783bfe81719a02ba5828a889da51ba#rd) 最后讲到组件更新时，需要先执行编译器生成的渲染函数得到组件的 vnode。

渲染函数之所以能生成 vnode 是通过其中的 `_c、_l、、_v、_s` 等方法实现的。比如：

* 普通的节点被编译成了可执行 _c 函数

* v-for 节点被编译成了可执行的 _l 函数

* ...

但是到目前为止我们都不清楚这些方法的原理，它们是如何生成 vnode 的？只知道它们是 Vue 实例方法，今天我们就从源码中找答案。

## 目标

在 Vue 编译器的基础上，进一步深入理解一个组件是如何通过这些运行时的工具方法（render helper）生成 VNode 的

## 源码解读

### 入口

我们知道这些方法是 Vue 实例方法，按照之前对源码的了解，实例方法一般都放在 `/src/core/instance` 目录下。其实之前在 [Vue 源码解读（6）—— 实例方法](https://mp.weixin.qq.com/s?__biz=MzA3NTk4NjQ1OQ==&mid=2247484965&idx=1&sn=ee411798a09d3facc39c5c7328a4160e&chksm=9f696551a81eec4714704bc7fb3be4c5afb02d1900f196caed0b15979a0fa0799bd2f2975561#rd) 阅读中见到过 `render helper`，在文章的最后。

> /src/core/instance/render.js

```javascript
export function renderMixin (Vue: Class<Component>) {
  // install runtime convenience helpers
  // 在组件实例上挂载一些运行时需要用到的工具方法
  installRenderHelpers(Vue.prototype)
  
  // ...
}

```

### installRenderHelpers

> /src/core/instance/render-helpers/index.js

```javascript
/**
 * 在实例上挂载简写的渲染工具函数，这些都是运行时代码
 * 这些工具函数在编译器生成的渲染函数中被使用到了
 * @param {*} target Vue 实例
 */
export function installRenderHelpers(target: any) {
  /**
   * v-once 指令的运行时帮助程序，为 VNode 加上打上静态标记
   * 有点多余，因为含有 v-once 指令的节点都被当作静态节点处理了，所以也不会走这儿
   */
  target._o = markOnce
  // 将值转换为数字
  target._n = toNumber
  /**
   * 将值转换为字符串形式，普通值 => String(val)，对象 => JSON.stringify(val)
   */
  target._s = toString
  /**
   * 运行时渲染 v-for 列表的帮助函数，循环遍历 val 值，依次为每一项执行 render 方法生成 VNode，最终返回一个 VNode 数组
   */
  target._l = renderList
  target._t = renderSlot
  /**
   * 判断两个值是否相等
   */
  target._q = looseEqual
  /**
   * 相当于 indexOf 方法
   */
  target._i = looseIndexOf
  /**
   * 运行时负责生成静态树的 VNode 的帮助程序，完成了以下两件事
   *   1、执行 staticRenderFns 数组中指定下标的渲染函数，生成静态树的 VNode 并缓存，下次在渲染时从缓存中直接读取（isInFor 必须为 true）
   *   2、为静态树的 VNode 打静态标记
   */
  target._m = renderStatic
  target._f = resolveFilter
  target._k = checkKeyCodes
  target._b = bindObjectProps
  /**
   * 为文本节点创建 VNode
   */
  target._v = createTextVNode
  /**
   * 为空节点创建 VNode
   */
  target._e = createEmptyVNode
}

```

### _o = markOnce

> /src/core/instance/render-helpers/render-static.js

```javascript
/**
 * Runtime helper for v-once.
 * Effectively it means marking the node as static with a unique key.
 * v-once 指令的运行时帮助程序，为 VNode 加上打上静态标记
 * 有点多余，因为含有 v-once 指令的节点都被当作静态节点处理了，所以也不会走这儿
 */
export function markOnce (
  tree: VNode | Array<VNode>,
  index: number,
  key: string
) {
  markStatic(tree, `__once__${index}${key ? `_${key}` : ``}`, true)
  return tree
}

```

#### markStatic

> /src/core/instance/render-helpers/render-static.js

```javascript
/**
 * 为 VNode 打静态标记，在 VNode 上添加三个属性：
 * { isStatick: true, key: xx, isOnce: true or false } 
 */
function markStatic (
  tree: VNode | Array<VNode>,
  key: string,
  isOnce: boolean
) {
  if (Array.isArray(tree)) {
    // tree 为 VNode 数组，循环遍历其中的每个 VNode，为每个 VNode 做静态标记
    for (let i = 0; i < tree.length; i++) {
      if (tree[i] && typeof tree[i] !== 'string') {
        markStaticNode(tree[i], `${key}_${i}`, isOnce)
      }
    }
  } else {
    markStaticNode(tree, key, isOnce)
  }
}

```

#### markStaticNode

> /src/core/instance/render-helpers/render-static.js

```javascript
/**
 * 标记静态 VNode
 */
function markStaticNode (node, key, isOnce) {
  node.isStatic = true
  node.key = key
  node.isOnce = isOnce
}

```

### _l = renderList

> /src/core/instance/render-helpers/render-list.js

```javascript
/**
 * Runtime helper for rendering v-for lists.
 * 运行时渲染 v-for 列表的帮助函数，循环遍历 val 值，依次为每一项执行 render 方法生成 VNode，最终返回一个 VNode 数组
 */
export function renderList (
  val: any,
  render: (
    val: any,
    keyOrIndex: string | number,
    index?: number
  ) => VNode
): ?Array<VNode> {
  let ret: ?Array<VNode>, i, l, keys, key
  if (Array.isArray(val) || typeof val === 'string') {
    // val 为数组或者字符串
    ret = new Array(val.length)
    for (i = 0, l = val.length; i < l; i++) {
      ret[i] = render(val[i], i)
    }
  } else if (typeof val === 'number') {
    // val 为一个数值，则遍历 0 - val 的所有数字
    ret = new Array(val)
    for (i = 0; i < val; i++) {
      ret[i] = render(i + 1, i)
    }
  } else if (isObject(val)) {
    // val 为一个对象，遍历对象
    if (hasSymbol && val[Symbol.iterator]) {
      // val 为一个可迭代对象
      ret = []
      const iterator: Iterator<any> = val[Symbol.iterator]()
      let result = iterator.next()
      while (!result.done) {
        ret.push(render(result.value, ret.length))
        result = iterator.next()
      }
    } else {
      // val 为一个普通对象
      keys = Object.keys(val)
      ret = new Array(keys.length)
      for (i = 0, l = keys.length; i < l; i++) {
        key = keys[i]
        ret[i] = render(val[key], key, i)
      }
    }
  }
  if (!isDef(ret)) {
    ret = []
  }
  // 返回 VNode 数组
  (ret: any)._isVList = true
  return ret
}

```

### _m = renderStatic

> /src/core/instance/render-helpers/render-static.js

```javascript
/**
 * Runtime helper for rendering static trees.
 * 运行时负责生成静态树的 VNode 的帮助程序，完成了以下两件事
 *   1、执行 staticRenderFns 数组中指定下标的渲染函数，生成静态树的 VNode 并缓存，下次在渲染时从缓存中直接读取（isInFor 必须为 true）
 *   2、为静态树的 VNode 打静态标记
 * @param { number} index 表示当前静态节点的渲染函数在 staticRenderFns 数组中的下标索引
 * @param { boolean} isInFor 表示当前静态节点是否被包裹在含有 v-for 指令的节点内部
 */
 export function renderStatic (
  index: number,
  isInFor: boolean
): VNode | Array<VNode> {
  // 缓存，静态节点第二次被渲染时就从缓存中直接获取已缓存的 VNode
  const cached = this._staticTrees || (this._staticTrees = [])
  let tree = cached[index]
  // if has already-rendered static tree and not inside v-for,
  // we can reuse the same tree.
  // 如果当前静态树已经被渲染过一次（即有缓存）而且没有被包裹在 v-for 指令所在节点的内部，则直接返回缓存的 VNode
  if (tree && !isInFor) {
    return tree
  }
  // 执行 staticRenderFns 数组中指定元素（静态树的渲染函数）生成该静态树的 VNode，并缓存
  // otherwise, render a fresh tree.
  tree = cached[index] = this.$options.staticRenderFns[index].call(
    this._renderProxy,
    null,
    this // for render fns generated for functional component templates
  )
  // 静态标记，为静态树的 VNode 打标记，即添加 { isStatic: true, key: `__static__${index}`, isOnce: false }
  markStatic(tree, `__static__${index}`, false)
  return tree
}

```

### _c

> /src/core/instance/render.js

```javascript
/**
 * 定义 _c，它是 createElement 的一个柯里化方法
 * @param {*} a 标签名
 * @param {*} b 属性的 JSON 字符串
 * @param {*} c 子节点数组
 * @param {*} d 节点的规范化类型
 * @returns VNode or Array<VNode>
 */
vm._c = (a, b, c, d) => createElement(vm, a, b, c, d, false)
```

#### createElement

> /src/core/vdom/create-element.js

```javascript
/**
 * 生成组件或普通标签的 vnode，一个包装函数，不用管
 * wrapper function for providing a more flexible interface
 * without getting yelled at by flow
 */
export function createElement(
  context: Component,
  tag: any,
  data: any,
  children: any,
  normalizationType: any,
  alwaysNormalize: boolean
): VNode | Array<VNode> {
  if (Array.isArray(data) || isPrimitive(data)) {
    normalizationType = children
    children = data
    data = undefined
  }
  if (isTrue(alwaysNormalize)) {
    normalizationType = ALWAYS_NORMALIZE
  }
  // 执行 _createElement 方法创建组件的 VNode
  return _createElement(context, tag, data, children, normalizationType)
}

```

#### _createElement 

> /src/core/vdom/create-element.js

```javascript
/**
 * 生成 vnode，
 *   1、平台保留标签和未知元素执行 new Vnode() 生成 vnode
 *   2、组件执行 createComponent 生成 vnode
 *     2.1 函数式组件执行自己的 render 函数生成 VNode
 *     2.2 普通组件则实例化一个 VNode，并且在其 data.hook 对象上设置 4 个方法，在组件的 patch 阶段会被调用，
 *         从而进入子组件的实例化、挂载阶段，直至完成渲染
 * @param {*} context 上下文
 * @param {*} tag 标签
 * @param {*} data 属性 JSON 字符串
 * @param {*} children 子节点数组
 * @param {*} normalizationType 节点规范化类型
 * @returns VNode or Array<VNode>
 */
export function _createElement(
  context: Component,
  tag?: string | Class<Component> | Function | Object,
  data?: VNodeData,
  children?: any,
  normalizationType?: number
): VNode | Array<VNode> {
  if (isDef(data) && isDef((data: any).__ob__)) {
    // 属性不能是一个响应式对象
    process.env.NODE_ENV !== 'production' && warn(
      `Avoid using observed data object as vnode data: ${JSON.stringify(data)}\n` +
      'Always create fresh vnode data objects in each render!',
      context
    )
    // 如果属性是一个响应式对象，则返回一个空节点的 VNode
    return createEmptyVNode()
  }
  // object syntax in v-bind
  if (isDef(data) && isDef(data.is)) {
    tag = data.is
  }
  if (!tag) {
    // 动态组件的 is 属性是一个假值时 tag 为 false，则返回一个空节点的 VNode
    // in case of component :is set to falsy value
    return createEmptyVNode()
  }
  // 检测唯一键 key，只能是字符串或者数字
  // warn against non-primitive key
  if (process.env.NODE_ENV !== 'production' &&
    isDef(data) && isDef(data.key) && !isPrimitive(data.key)
  ) {
    if (!__WEEX__ || !('@binding' in data.key)) {
      warn(
        'Avoid using non-primitive value as key, ' +
        'use string/number value instead.',
        context
      )
    }
  }

  // 子节点数组中只有一个函数时，将它当作默认插槽，然后清空子节点列表
  // support single function children as default scoped slot
  if (Array.isArray(children) &&
    typeof children[0] === 'function'
  ) {
    data = data || {}
    data.scopedSlots = { default: children[0] }
    children.length = 0
  }
  // 将子元素进行标准化处理
  if (normalizationType === ALWAYS_NORMALIZE) {
    children = normalizeChildren(children)
  } else if (normalizationType === SIMPLE_NORMALIZE) {
    children = simpleNormalizeChildren(children)
  }

   /**
   * 这里开始才是重点，前面的都不需要关注，基本上是一些异常处理或者优化等
   */

  let vnode, ns
  if (typeof tag === 'string') {
    // 标签是字符串时，该标签有三种可能：
    //   1、平台保留标签
    //   2、自定义组件
    //   3、不知名标签
    let Ctor
    // 命名空间
    ns = (context.$vnode && context.$vnode.ns) || config.getTagNamespace(tag)
    if (config.isReservedTag(tag)) {
      // tag 是平台原生标签
      // platform built-in elements
      if (process.env.NODE_ENV !== 'production' && isDef(data) && isDef(data.nativeOn)) {
        // v-on 指令的 .native 只在组件上生效
        warn(
          `The .native modifier for v-on is only valid on components but it was used on <${tag}>.`,
          context
        )
      }
      // 实例化一个 VNode
      vnode = new VNode(
        config.parsePlatformTagName(tag), data, children,
        undefined, undefined, context
      )
    } else if ((!data || !data.pre) && isDef(Ctor = resolveAsset(context.$options, 'components', tag))) {
      // tag 是一个自定义组件
      // 在 this.$options.components 对象中找到指定标签名称的组件构造函数
      // 创建组件的 VNode，函数式组件直接执行其 render 函数生成 VNode，
      // 普通组件则实例化一个 VNode，并且在其 data.hook 对象上设置了 4 个方法，在组件的 patch 阶段会被调用，
      // 从而进入子组件的实例化、挂载阶段，直至完成渲染
      // component
      vnode = createComponent(Ctor, data, context, children, tag)
    } else {
      // 不知名的一个标签，但也生成 VNode，因为考虑到在运行时可能会给一个合适的名字空间
      // unknown or unlisted namespaced elements
      // check at runtime because it may get assigned a namespace when its
      // parent normalizes children
      vnode = new VNode(
        tag, data, children,
        undefined, undefined, context
      )
    }
  } else {
     // tag 为非字符串，比如可能是一个组件的配置对象或者是一个组件的构造函数
    // direct component options / constructor
    vnode = createComponent(tag, data, context, children)
  }
  // 返回组件的 VNode
  if (Array.isArray(vnode)) {
    return vnode
  } else if (isDef(vnode)) {
    if (isDef(ns)) applyNS(vnode, ns)
    if (isDef(data)) registerDeepBindings(data)
    return vnode
  } else {
    return createEmptyVNode()
  }
}

```

#### createComponent

> /src/core/vdom/create-component.js

```javascript
/**
 * 创建组件的 VNode，
 *   1、函数式组件通过执行其 render 方法生成组件的 VNode
 *   2、普通组件通过 new VNode() 生成其 VNode，但是普通组件有一个重要操作是在 data.hook 对象上设置了四个钩子函数，
 *      分别是 init、prepatch、insert、destroy，在组件的 patch 阶段会被调用，
 *      比如 init 方法，调用时会进入子组件实例的创建挂载阶段，直到完成渲染
 * @param {*} Ctor 组件构造函数
 * @param {*} data 属性组成的 JSON 字符串
 * @param {*} context 上下文
 * @param {*} children 子节点数组
 * @param {*} tag 标签名
 * @returns VNode or Array<VNode>
 */
export function createComponent(
  Ctor: Class<Component> | Function | Object | void,
  data: ?VNodeData,
  context: Component,
  children: ?Array<VNode>,
  tag?: string
): VNode | Array<VNode> | void {
  // 组件构造函数不存在，直接结束
  if (isUndef(Ctor)) {
    return
  }

  // Vue.extend
  const baseCtor = context.$options._base

  // 当 Ctor 为配置对象时，通过 Vue.extend 将其转为构造函数
  // plain options object: turn it into a constructor
  if (isObject(Ctor)) {
    Ctor = baseCtor.extend(Ctor)
  }

  // 如果到这个为止，Ctor 仍然不是一个函数，则表示这是一个无效的组件定义
  // if at this stage it's not a constructor or an async component factory,
  // reject.
  if (typeof Ctor !== 'function') {
    if (process.env.NODE_ENV !== 'production') {
      warn(`Invalid Component definition: ${String(Ctor)}`, context)
    }
    return
  }

  // 异步组件
  // async component
  let asyncFactory
  if (isUndef(Ctor.cid)) {
    asyncFactory = Ctor
    Ctor = resolveAsyncComponent(asyncFactory, baseCtor)
    if (Ctor === undefined) {
      // 为异步组件返回一个占位符节点，组件被渲染为注释节点，但保留了节点的所有原始信息，这些信息将用于异步服务器渲染 和 hydration
      return createAsyncPlaceholder(
        asyncFactory,
        data,
        context,
        children,
        tag
      )
    }
  }

  // 节点的属性 JSON 字符串
  data = data || {}

  // 这里其实就是组件做选项合并的地方，即编译器将组件编译为渲染函数，渲染时执行 render 函数，然后执行其中的 _c，就会走到这里了
  // 解析构造函数选项，并合基类选项，以防止在组件构造函数创建后应用全局混入
  // resolve constructor options in case global mixins are applied after
  // component constructor creation
  resolveConstructorOptions(Ctor)

  // 将组件的 v-model 的信息（值和回调）转换为 data.attrs 对象的属性、值和 data.on 对象上的事件、回调
  // transform component v-model data into props & events
  if (isDef(data.model)) {
    transformModel(Ctor.options, data)
  }

  // 提取 props 数据，得到 propsData 对象，propsData[key] = val
  // 以组件 props 配置中的属性为 key，父组件中对应的数据为 value
  // extract props
  const propsData = extractPropsFromVNodeData(data, Ctor, tag)

  // 函数式组件
  // functional component
  if (isTrue(Ctor.options.functional)) {
    /**
     * 执行函数式组件的 render 函数生成组件的 VNode，做了以下 3 件事：
     *   1、设置组件的 props 对象
     *   2、设置函数式组件的渲染上下文，传递给函数式组件的 render 函数
     *   3、调用函数式组件的 render 函数生成 vnode
     */
    return createFunctionalComponent(Ctor, propsData, data, context, children)
  }

  // 获取事件监听器对象 data.on，因为这些监听器需要作为子组件监听器处理，而不是 DOM 监听器
  // extract listeners, since these needs to be treated as
  // child component listeners instead of DOM listeners
  const listeners = data.on
  // 将带有 .native 修饰符的事件对象赋值给 data.on
  // replace with listeners with .native modifier
  // so it gets processed during parent component patch.
  data.on = data.nativeOn

  if (isTrue(Ctor.options.abstract)) {
    // 如果是抽象组件，则值保留 props、listeners 和 slot
    // abstract components do not keep anything
    // other than props & listeners & slot

    // work around flow
    const slot = data.slot
    data = {}
    if (slot) {
      data.slot = slot
    }
  }

  /**
   * 在组件的 data 对象上设置 hook 对象，
   * hook 对象增加四个属性，init、prepatch、insert、destroy，
   * 负责组件的创建、更新、销毁，这些方法在组件的 patch 阶段会被调用
   * install component management hooks onto the placeholder node
   */
  installComponentHooks(data)

  const name = Ctor.options.name || tag
  // 实例化组件的 VNode，对于普通组件的标签名会比较特殊，vue-component-${cid}-${name}
  const vnode = new VNode(
    `vue-component-${Ctor.cid}${name ? `-${name}` : ''}`,
    data, undefined, undefined, undefined, context,
    { Ctor, propsData, listeners, tag, children },
    asyncFactory
  )

  // Weex specific: invoke recycle-list optimized @render function for
  // extracting cell-slot template.
  // https://github.com/Hanks10100/weex-native-directive/tree/master/component
  /* istanbul ignore if */
  if (__WEEX__ && isRecyclableComponent(vnode)) {
    return renderRecyclableComponentTemplate(vnode)
  }

  return vnode
}

```

#### resolveConstructorOptions 

> /src/core/instance/init.js

```javascript
/**
 * 从构造函数上解析配置项
 */
export function resolveConstructorOptions (Ctor: Class<Component>) {
  // 从实例构造函数上获取选项
  let options = Ctor.options
  if (Ctor.super) {
    const superOptions = resolveConstructorOptions(Ctor.super)
    // 缓存
    const cachedSuperOptions = Ctor.superOptions
    if (superOptions !== cachedSuperOptions) {
      // 说明基类的配置项发生了更改
      // super option changed,
      // need to resolve new options.
      Ctor.superOptions = superOptions
      // check if there are any late-modified/attached options (#4976)
      // 找到更改的选项
      const modifiedOptions = resolveModifiedOptions(Ctor)
      // update base extend options
      if (modifiedOptions) {
        // 将更改的选项和 extend 选项合并
        extend(Ctor.extendOptions, modifiedOptions)
      }
      // 将新的选项赋值给 options
      options = Ctor.options = mergeOptions(superOptions, Ctor.extendOptions)
      if (options.name) {
        options.components[options.name] = Ctor
      }
    }
  }
  return options
}

```

#### resolveModifiedOptions

> /src/core/instance/init.js

```javascript
/**
 * 解析构造函数选项中后续被修改或者增加的选项
 */
function resolveModifiedOptions (Ctor: Class<Component>): ?Object {
  let modified
  // 构造函数选项
  const latest = Ctor.options
  // 密封的构造函数选项，备份
  const sealed = Ctor.sealedOptions
  // 对比两个选项，记录不一致的选项
  for (const key in latest) {
    if (latest[key] !== sealed[key]) {
      if (!modified) modified = {}
      modified[key] = latest[key]
    }
  }
  return modified
}

```

#### transformModel

> src/core/vdom/create-component.js

```javascript
/**
 * 将组件的 v-model 的信息（值和回调）转换为 data.attrs 对象的属性、值和 data.on 对象上的事件、回调
 * transform component v-model info (value and callback) into
 * prop and event handler respectively.
 */
function transformModel(options, data: any) {
  // model 的属性和事件，默认为 value 和 input
  const prop = (options.model && options.model.prop) || 'value'
  const event = (options.model && options.model.event) || 'input'
    // 在 data.attrs 对象上存储 v-model 的值
    ; (data.attrs || (data.attrs = {}))[prop] = data.model.value
  // 在 data.on 对象上存储 v-model 的事件
  const on = data.on || (data.on = {})
  // 已存在的事件回调函数
  const existing = on[event]
  // v-model 中事件对应的回调函数
  const callback = data.model.callback
  // 合并回调函数
  if (isDef(existing)) {
    if (
      Array.isArray(existing)
        ? existing.indexOf(callback) === -1
        : existing !== callback
    ) {
      on[event] = [callback].concat(existing)
    }
  } else {
    on[event] = callback
  }
}

```

#### extractPropsFromVNodeData

> /src/core/vdom/helpers/extract-props.js

```javascript
/**
 * <comp :msg="hello vue"></comp>
 * 
 * 提取 props，得到 res[key] = val 
 * 
 * 以 props 配置中的属性为 key，父组件中对应的的数据为 value
 * 当父组件中数据更新时，触发响应式更新，重新执行 render，生成新的 vnode，又走到这里
 * 这样子组件中相应的数据就会被更新 
 */
export function extractPropsFromVNodeData (
  data: VNodeData, // { msg: 'hello vue' }
  Ctor: Class<Component>, // 组件构造函数
  tag?: string // 组件标签名
): ?Object {
  // 组件的 props 选项，{ props: { msg: { type: String, default: xx } } }
  
  // 这里只提取原始值，验证和默认值在子组件中处理
  // we are only extracting raw values here.
  // validation and default values are handled in the child
  // component itself.
  const propOptions = Ctor.options.props
  if (isUndef(propOptions)) {
    // 未定义 props 直接返回
    return
  }
  // 以组件 props 配置中的属性为 key，父组件传递下来的值为 value
  // 当父组件中数据更新时，触发响应式更新，重新执行 render，生成新的 vnode，又走到这里
  // 这样子组件中相应的数据就会被更新
  const res = {}
  const { attrs, props } = data
  if (isDef(attrs) || isDef(props)) {
    // 遍历 propsOptions
    for (const key in propOptions) {
      // 将小驼峰形式的 key 转换为 连字符 形式
      const altKey = hyphenate(key)
      // 提示，如果声明的 props 为小驼峰形式（testProps），但由于 html 不区分大小写，所以在 html 模版中应该使用 test-props 代替 testProps
      if (process.env.NODE_ENV !== 'production') {
        const keyInLowerCase = key.toLowerCase()
        if (
          key !== keyInLowerCase &&
          attrs && hasOwn(attrs, keyInLowerCase)
        ) {
          tip(
            `Prop "${keyInLowerCase}" is passed to component ` +
            `${formatComponentName(tag || Ctor)}, but the declared prop name is` +
            ` "${key}". ` +
            `Note that HTML attributes are case-insensitive and camelCased ` +
            `props need to use their kebab-case equivalents when using in-DOM ` +
            `templates. You should probably use "${altKey}" instead of "${key}".`
          )
        }
      }
      checkProp(res, props, key, altKey, true) ||
      checkProp(res, attrs, key, altKey, false)
    }
  }
  return res
}

```

#### checkProp

> /src/core/vdom/helpers/extract-props.js

```javascript
/**
 * 得到 res[key] = val
 */
function checkProp (
  res: Object,
  hash: ?Object,
  key: string,
  altKey: string,
  preserve: boolean
): boolean {
  if (isDef(hash)) {
    // 判断 hash（props/attrs）对象中是否存在 key 或 altKey
    // 存在则设置给 res => res[key] = hash[key]
    if (hasOwn(hash, key)) {
      res[key] = hash[key]
      if (!preserve) {
        delete hash[key]
      }
      return true
    } else if (hasOwn(hash, altKey)) {
      res[key] = hash[altKey]
      if (!preserve) {
        delete hash[altKey]
      }
      return true
    }
  }
  return false
}

```

#### createFunctionalComponent

> /src/core/vdom/create-functional-component.js

```javascript
installRenderHelpers(FunctionalRenderContext.prototype)

/**
 * 执行函数式组件的 render 函数生成组件的 VNode，做了以下 3 件事：
 *   1、设置组件的 props 对象
 *   2、设置函数式组件的渲染上下文，传递给函数式组件的 render 函数
 *   3、调用函数式组件的 render 函数生成 vnode
 * 
 * @param {*} Ctor 组件的构造函数 
 * @param {*} propsData 额外的 props 对象
 * @param {*} data 节点属性组成的 JSON 字符串
 * @param {*} contextVm 上下文
 * @param {*} children 子节点数组
 * @returns Vnode or Array<VNode>
 */
export function createFunctionalComponent (
  Ctor: Class<Component>,
  propsData: ?Object,
  data: VNodeData,
  contextVm: Component,
  children: ?Array<VNode>
): VNode | Array<VNode> | void {
  // 组件配置项
  const options = Ctor.options
  // 获取 props 对象
  const props = {}
  // 组件本身的 props 选项
  const propOptions = options.props
  // 设置函数式组件的 props 对象
  if (isDef(propOptions)) {
    // 说明该函数式组件本身提供了 props 选项，则将 props.key 的值设置为组件上传递下来的对应 key 的值
    for (const key in propOptions) {
      props[key] = validateProp(key, propOptions, propsData || emptyObject)
    }
  } else {
    // 当前函数式组件没有提供 props 选项，则将组件上的 attribute 自动解析为 props
    if (isDef(data.attrs)) mergeProps(props, data.attrs)
    if (isDef(data.props)) mergeProps(props, data.props)
  }

  // 实例化函数式组件的渲染上下文
  const renderContext = new FunctionalRenderContext(
    data,
    props,
    children,
    contextVm,
    Ctor
  )

  // 调用 render 函数，生成 vnode，并给 render 函数传递 _c 和 渲染上下文
  const vnode = options.render.call(null, renderContext._c, renderContext)

  // 在最后生成的 VNode 对象上加一些标记，表示该 VNode 是一个函数式组件生成的，最后返回 VNode
  if (vnode instanceof VNode) {
    return cloneAndMarkFunctionalResult(vnode, data, renderContext.parent, options, renderContext)
  } else if (Array.isArray(vnode)) {
    const vnodes = normalizeChildren(vnode) || []
    const res = new Array(vnodes.length)
    for (let i = 0; i < vnodes.length; i++) {
      res[i] = cloneAndMarkFunctionalResult(vnodes[i], data, renderContext.parent, options, renderContext)
    }
    return res
  }
}

```

#### installComponentHooks

> /src/core/vdom/create-component.js

```javascript
const hooksToMerge = Object.keys(componentVNodeHooks)
/**
 * 在组件的 data 对象上设置 hook 对象，
 * hook 对象增加四个属性，init、prepatch、insert、destroy，
 * 负责组件的创建、更新、销毁
 */
 function installComponentHooks(data: VNodeData) {
  const hooks = data.hook || (data.hook = {})
  // 遍历 hooksToMerge 数组，hooksToMerge = ['init', 'prepatch', 'insert' 'destroy']
  for (let i = 0; i < hooksToMerge.length; i++) {
    // 比如 key = init
    const key = hooksToMerge[i]
    // 从 data.hook 对象中获取 key 对应的方法
    const existing = hooks[key]
    // componentVNodeHooks 对象中 key 对象的方法
    const toMerge = componentVNodeHooks[key]
    // 合并用户传递的 hook 方法和框架自带的 hook 方法，其实就是分别执行两个方法
    if (existing !== toMerge && !(existing && existing._merged)) {
      hooks[key] = existing ? mergeHook(toMerge, existing) : toMerge
    }
  }
}

function mergeHook(f1: any, f2: any): Function {
  const merged = (a, b) => {
    // flow complains about extra args which is why we use any
    f1(a, b)
    f2(a, b)
  }
  merged._merged = true
  return merged
}

```

#### componentVNodeHooks

> /src/core/vdom/create-component.js

```javascript
// patch 期间在组件 vnode 上调用内联钩子
// inline hooks to be invoked on component VNodes during patch
const componentVNodeHooks = {
  // 初始化
  init(vnode: VNodeWithData, hydrating: boolean): ?boolean {
    if (
      vnode.componentInstance &&
      !vnode.componentInstance._isDestroyed &&
      vnode.data.keepAlive
    ) {
      // 被 keep-alive 包裹的组件
      // kept-alive components, treat as a patch
      const mountedNode: any = vnode // work around flow
      componentVNodeHooks.prepatch(mountedNode, mountedNode)
    } else {
      // 创建组件实例，即 new vnode.componentOptions.Ctor(options) => 得到 Vue 组件实例
      const child = vnode.componentInstance = createComponentInstanceForVnode(
        vnode,
        activeInstance
      )
      // 执行组件的 $mount 方法，进入挂载阶段，接下来就是通过编译器得到 render 函数，接着走挂载、patch 这条路，直到组件渲染到页面
      child.$mount(hydrating ? vnode.elm : undefined, hydrating)
    }
  },

  // 更新 VNode，用新的 VNode 配置更新旧的 VNode 上的各种属性
  prepatch(oldVnode: MountedComponentVNode, vnode: MountedComponentVNode) {
    // 新 VNode 的组件配置项
    const options = vnode.componentOptions
    // 老 VNode 的组件实例
    const child = vnode.componentInstance = oldVnode.componentInstance
    // 用 vnode 上的属性更新 child 上的各种属性
    updateChildComponent(
      child,
      options.propsData, // updated props
      options.listeners, // updated listeners
      vnode, // new parent vnode
      options.children // new children
    )
  },

  // 执行组件的 mounted 声明周期钩子
  insert(vnode: MountedComponentVNode) {
    const { context, componentInstance } = vnode
    // 如果组件未挂载，则调用 mounted 声明周期钩子
    if (!componentInstance._isMounted) {
      componentInstance._isMounted = true
      callHook(componentInstance, 'mounted')
    }
    // 处理 keep-alive 组件的异常情况
    if (vnode.data.keepAlive) {
      if (context._isMounted) {
        // vue-router#1212
        // During updates, a kept-alive component's child components may
        // change, so directly walking the tree here may call activated hooks
        // on incorrect children. Instead we push them into a queue which will
        // be processed after the whole patch process ended.
        queueActivatedComponent(componentInstance)
      } else {
        activateChildComponent(componentInstance, true /* direct */)
      }
    }
  },

  /**
   * 销毁组件
   *   1、如果组件被 keep-alive 组件包裹，则使组件失活，不销毁组件实例，从而缓存组件的状态
   *   2、如果组件没有被 keep-alive 包裹，则直接调用实例的 $destroy 方法销毁组件
   */
  destroy (vnode: MountedComponentVNode) {
    // 从 vnode 上获取组件实例
    const { componentInstance } = vnode
    if (!componentInstance._isDestroyed) {
      // 如果组件实例没有被销毁
      if (!vnode.data.keepAlive) {
        // 组件没有被 keep-alive 组件包裹，则直接调用 $destroy 方法销毁组件
        componentInstance.$destroy()
      } else {
        // 负责让组件失活，不销毁组件实例，从而缓存组件的状态
        deactivateChildComponent(componentInstance, true /* direct */)
      }
    }
  }
}

```

#### createComponentInstanceForVnode

> /src/core/vdom/create-component.js

```javascript
/**
 * new vnode.componentOptions.Ctor(options) => 得到 Vue 组件实例 
 */
export function createComponentInstanceForVnode(
  // we know it's MountedComponentVNode but flow doesn't
  vnode: any,
  // activeInstance in lifecycle state
  parent: any
): Component {
  const options: InternalComponentOptions = {
    _isComponent: true,
    _parentVnode: vnode,
    parent
  }
  // 检查内联模版渲染函数
  const inlineTemplate = vnode.data.inlineTemplate
  if (isDef(inlineTemplate)) {
    options.render = inlineTemplate.render
    options.staticRenderFns = inlineTemplate.staticRenderFns
  }
  // new VueComponent(options) => Vue 实例
  return new vnode.componentOptions.Ctor(options)
}

```

## 总结

**面试官 问**：一个组件是如何变成 VNode？

**答**：

* 组件实例初始化，最后执行 $mount 进入挂载阶段

* 如果是只包含运行时的 vue.js，只直接进入挂载阶段，因为这时候的组件已经变成了渲染函数，编译过程通过模块打包器 + vue-loader + vue-template-compiler 完成的

* 如果没有使用预编译，则必须使用全量的 vue.js

* 挂载时如果发现组件配置项上没有 render 选项，则进入编译阶段

* 将模版字符串编译成 AST 语法树，其实就是一个普通的 JS 对象

* 然后优化 AST，遍历 AST 对象，标记每一个节点是否为静态静态；然后再进一步标记出静态根节点，在组件后续更新时会跳过这些静态节点的更新，以提高性能

* 接下来从 AST 生成渲染函数，生成的渲染函数有两部分组成：

  * 负责生成动态节点 VNode 的 render 函数
  
  * 还有一个 staticRenderFns 数组，里面每一个元素都是一个生成静态节点 VNode 的函数，这些函数会作为 render 函数的组成部分，负责生成静态节点的 VNode
  
* 接下来将渲染函数放到组件的配置对象上，进入挂载阶段，即执行 mountComponent 方法

* 最终负责渲染组件和更新组件的是一个叫 updateComponent 方法，该方法每次执行前首先需要执行 vm._render 函数，该函数负责执行编译器生成的 render，得到组件的 VNode

* 将一个组件生成 VNode 的具体工作是由 render 函数中的 `_c、_o、_l、_m` 等方法完成的，这些方法都被挂载到 Vue 实例上面，负责在运行时生成组件 VNode

> **提示**：到这里首先要明白什么是 VNode，一句话描述就是 —— 组件模版的 JS 对象表现形式，它就是一个普通的 JS 对象，详细描述了组件中各节点的信息

> 下面说的有点多，其实记住一句就可以了，设置组件配置信息，然后通过 `new VNode(组件信息)` 生成组件的 VNode

* _c，负责生成组件或 HTML 元素的 VNode，_c 是所有 render helper 方法中最复杂，也是最核心的一个方法，其它的 _xx 都是它的组成部分

  * 接收标签、属性 JSON 字符串、子节点数组、节点规范化类型作为参数
  
  * 如果标签是平台保留标签或者一个未知的元素，则直接 `new VNode(标签信息)` 得到 VNode
  
  * 如果标签是一个组件，则执行 createComponent 方法生成 VNode
  
    * 函数式组件执行自己的 render 函数生成 VNode
    
    * 普通组件则实例化一个 VNode，并且在在 data.hook 对象上设置 4 个方法，在组件的 patch 阶段会被调用，从而进入子组件的实例化、挂载阶段，然后进行编译生成渲染函数，直至完成渲染
    
    * 当然生成 VNode 之前会进行一些配置处理比如：
    
        * 子组件选项合并，合并全局配置项到组件配置项上
    
        * 处理自定义组件的 v-model
    
        * 处理组件的 props，提取组件的 props 数据，以组件的 props 配置中的属性为 key，父组件中对应的数据为 value 生成一个 propsData 对象；当组件更新时生成新的 VNode，又会进行这一步，这就是 props 响应式的原理
    
        * 处理其它数据，比如监听器
    
        * 安装内置的 init、prepatch、insert、destroy 钩子到 data.hooks 对象上，组件 patch 阶段会用到这些钩子方法

* _l，运行时渲染 v-for 列表的帮助函数，循环遍历 val 值，依次为每一项执行 render 方法生成 VNode，最终返回一个 VNode 数组

* _m，负责生成静态节点的 VNode，即执行 staticRenderFns 数组中指定下标的函数

**简单总结 render helper 的作用就是**：在 Vue 实例上挂载一些运行时的工具方法，这些方法用在编译器生成的渲染函数中，用于生成组件的 VNode。

好了，到这里，一个组件从初始化开始到最终怎么变成 VNode 就讲完了，最后剩下的就是 patch 阶段了，下一篇文章将讲述如何将组件的 VNode 渲染到页面上。

## 链接

* [配套视频，微信公众号回复](https://gitee.com/liyongning/typora-image-bed/raw/master/202202171742614.jpg)："精通 Vue 技术栈源码原理视频版" 获取
* [精通 Vue 技术栈源码原理 专栏](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzA3NTk4NjQ1OQ==&action=getalbum&album_id=2273541436891693065#wechat_redirect)
* [github 仓库 liyongning/Vue](https://github.com/liyongning/Vue) 欢迎 Star



感谢各位的：**关注**、**点赞**、**收藏**和**评论**，我们下期见。

***

**当学习成为了习惯，知识也就变成了常识。** 感谢各位的 **关注**、 **点赞**、**收藏**和**评论**。

新视频和文章会第一时间在微信公众号发送，欢迎关注：[李永宁lyn](https://gitee.com/liyongning/typora-image-bed/raw/master/202202171742614.jpg)

文章已收录到 [github 仓库 liyongning/blog](https://github.com/liyongning/blog)，欢迎 Watch 和 Star。