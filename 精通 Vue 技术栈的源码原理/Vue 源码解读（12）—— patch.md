# Vue 源码解读（12）—— patch

**当学习成为了习惯，知识也就变成了常识。** 感谢各位的 **关注**、**点赞**、**收藏**和**评论**。

新视频和文章会第一时间在微信公众号发送，欢迎关注：[李永宁lyn](https://gitee.com/liyongning/typora-image-bed/raw/master/202202171742614.jpg)

文章已收录到 [github 仓库 liyongning/blog](https://github.com/liyongning/blog)，欢迎 Watch 和 Star。

## 封面

![](https://gitee.com/liyongning/typora-image-bed/raw/master/202203041216411.png)

## 前言

前面我们说到，当组件更新时，实例化渲染 watcher 时传递的 `updateComponent` 方法会被执行：

```javascript
const updateComponent = () => {
  // 执行 vm._render() 函数，得到 虚拟 VNode，并将 VNode 传递给 vm._update 方法，接下来就该到 patch 阶段了
  vm._update(vm._render(), hydrating)
}

```

首先会先执行 vm._render() 函数，得到组件的 VNode，并将 VNode 传递给 vm._update 方法，接下来就该进入到 patch 阶段了。今天我们就来深入理解组件更新时 patch 的执行过程。

## 历史

1.x 版本的 Vue 没有 VNode 和 diff 算法，那个版本的 Vue 的核心只有响应式原理：`Object.defineProperty`、`Dep`、`Watcher`。

* `Object.defineProperty`: 负责数据的拦截。getter 时进行依赖收集，setter 时让 dep 通知 watcher 去更新

* `Dep`：Vue data 选项返回的对象，对象的 key 和 dep 一一对应

* `Watcher`：key 和 watcher 时一对多的关系，组件模版中每使用一次 key 就会生成一个 watcher

```html
<template>
  <div class="wrapper">
    <!-- 模版中每引用一次响应式数据，就会生成一个 watcher -->
    <!-- watcher 1 -->
    <div class="msg1">{{ msg }}</div>
    <!-- watcher 2 -->
    <div class="msg2">{{ msg }}</div>
  </div>
</template>

<script>
export default {
  data() {
    return {
      // 和 dep 一一对应，和 watcher 一 对 多
      msg: 'Hello Vue 1.0'
    }
  }
}
</script>

```

当数据更新时，dep 通知 watcher 去直接更新 DOM，因为这个版本的 watcher 和 DOM 时一一对应关系，watcher 可以非常明确的知道这个 key 在组件模版中的位置，因此可以做到定向更新，所以它的更新效率是非常高的。

虽然更新效率高，但随之也产生了严重的问题，无法完成一个企业级应用，理由很简单：当你的页面足够复杂时，会包含很多的组件，在这种架构下就意味这一个页面会产生大量的 watcher，这非常耗资源。

这时就在 Vue 2.0 中通过引入 VNode 和 diff 算法去解决 1.x 中的问题。将 watcher 的粒度放大，变成一个组件一个 watcher（就是我们说的渲染 watcher），这时候你页面再大，watcher 也很少，这就解决了复杂页面 watcher 太多导致性能下降的问题。

当响应式数据更新时，dep 通知 watcher 去更新，这时候问题就来了，Vue 1.x 中 watcher 和 key 一一对应，可以明确知道去更新什么地方，但是 Vue 2.0 中 watcher 对应的是一整个组件，更新的数据在组件的的什么位置，watcher 并不知道。这时候就需要 VNode 出来解决问题。

通过引入 VNode，当组件中数据更新时，会为组件生成一个新的 VNode，通过比对新老两个 VNode，找出不一样的地方，然后执行 DOM 操作更新发生变化的节点，这个过程就是大家熟知的 diff。

以上就是 Vue 2.0 为什么会引入 VNode 和 diff 算法的历史原因了，也是 Vue 1.x 到 2.x 的一个发展历程。

## 目标

* 深入理解 Vue 的 patch 阶段，理解其 diff 算法的原理。

## 源码解读

### 入口

> /src/core/instance/lifecycle.js

```javascript
const updateComponent = () => {
  // 执行 vm._render() 函数，得到 VNode，并将 VNode 传递给 _update 方法，接下来就该到 patch 阶段了
  vm._update(vm._render(), hydrating)
}

```

### vm._update

> /src/core/instance/lifecycle.js

```javascript
/**
 * 页面首次渲染和后续更新的入口位置，也是 patch 的入口位置 
 */
Vue.prototype._update = function (vnode: VNode, hydrating?: boolean) {
  const vm: Component = this
  // 页面的挂载点，真实的元素
  const prevEl = vm.$el
  // 老 VNode
  const prevVnode = vm._vnode
  const restoreActiveInstance = setActiveInstance(vm)
  // 新 VNode
  vm._vnode = vnode
  // Vue.prototype.__patch__ is injected in entry points
  // based on the rendering backend used.
  if (!prevVnode) {
    // 老 VNode 不存在，表示首次渲染，即初始化页面时走这里
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

### vm.\_\_patch\_\_

> /src/platforms/web/runtime/index.js

```javascript
/ 在 Vue 原型链上安装 web 平台的 patch 函数
Vue.prototype.__patch__ = inBrowser ? patch : noop
```

### patch

> /src/platforms/web/runtime/patch.js

```javascript
// patch 工厂函数，为其传入平台特有的一些操作，然后返回一个 patch 函数
export const patch: Function = createPatchFunction({ nodeOps, modules })
```

#### nodeOps

> src/platforms/web/runtime/node-ops.js

```javascript
/**
 * web 平台的 DOM 操作 API
 */

/**
 * 创建标签名为 tagName 的元素节点
 */
export function createElement (tagName: string, vnode: VNode): Element {
  // 创建元素节点
  const elm = document.createElement(tagName)
  if (tagName !== 'select') {
    return elm
  }
  // false or null will remove the attribute but undefined will not
  // 如果是 select 元素，则为它设置 multiple 属性
  if (vnode.data && vnode.data.attrs && vnode.data.attrs.multiple !== undefined) {
    elm.setAttribute('multiple', 'multiple')
  }
  return elm
}

// 创建带命名空间的元素节点
export function createElementNS (namespace: string, tagName: string): Element {
  return document.createElementNS(namespaceMap[namespace], tagName)
}

// 创建文本节点
export function createTextNode (text: string): Text {
  return document.createTextNode(text)
}

// 创建注释节点
export function createComment (text: string): Comment {
  return document.createComment(text)
}

// 在指定节点前插入节点
export function insertBefore (parentNode: Node, newNode: Node, referenceNode: Node) {
  parentNode.insertBefore(newNode, referenceNode)
}

/**
 * 移除指定子节点
 */
export function removeChild (node: Node, child: Node) {
  node.removeChild(child)
}

/**
 * 添加子节点
 */
export function appendChild (node: Node, child: Node) {
  node.appendChild(child)
}

/**
 * 返回指定节点的父节点
 */
export function parentNode (node: Node): ?Node {
  return node.parentNode
}

/**
 * 返回指定节点的下一个兄弟节点
 */
export function nextSibling (node: Node): ?Node {
  return node.nextSibling
}

/**
 * 返回指定节点的标签名 
 */
export function tagName (node: Element): string {
  return node.tagName
}

/**
 * 为指定节点设置文本 
 */
export function setTextContent (node: Node, text: string) {
  node.textContent = text
}

/**
 * 为节点设置指定的 scopeId 属性，属性值为 ''
 */
export function setStyleScope (node: Element, scopeId: string) {
  node.setAttribute(scopeId, '')
}

```

#### modules

> /src/platforms/web/runtime/modules 和 /src/core/vdom/modules


平台特有的一些操作，比如：attr、class、style、event 等，还有核心的 directive 和 ref，它们会向外暴露一些特有的方法，比如：create、activate、update、remove、destroy，这些方法在 patch 阶段时会被调用，从而做相应的操作，比如 创建 attr、指令等。这部分内容太多了，这里就不一一列举了，在阅读 patch 的过程中如有需要可回头深入阅读，比如操作节点的属性的时候，就去读 attr 相关的代码。

### createPatchFunction

**提示**：由于该函数的代码量较大， 所以调整了一下代码结构，方便阅读和理解

> /src/core/vdom/patch.js

```javascript
const hooks = ['create', 'activate', 'update', 'remove', 'destroy']

/**
 * 工厂函数，注入平台特有的一些功能操作，并定义一些方法，然后返回 patch 函数
 */
export function createPatchFunction (backend) {
  let i, j
  const cbs = {}

  /**
   * modules: { ref, directives, 平台特有的一些操纵，比如 attr、class、style 等 }
   * nodeOps: { 对元素的增删改查 API }
   */
  const { modules, nodeOps } = backend

  /**
   * hooks = ['create', 'activate', 'update', 'remove', 'destroy']
   * 遍历这些钩子，然后从 modules 的各个模块中找到相应的方法，比如：directives 中的 create、update、destroy 方法
   * 让这些方法放到 cb[hook] = [hook 方法] 中，比如: cb.create = [fn1, fn2, ...]
   * 然后在合适的时间调用相应的钩子方法完成对应的操作
   */
  for (i = 0; i < hooks.length; ++i) {
    // 比如 cbs.create = []
    cbs[hooks[i]] = []
    for (j = 0; j < modules.length; ++j) {
      if (isDef(modules[j][hooks[i]])) {
        // 遍历各个 modules，找出各个 module 中的 create 方法，然后添加到 cbs.create 数组中
        cbs[hooks[i]].push(modules[j][hooks[i]])
      }
    }
  }
  /**
   * vm.__patch__
   *   1、新节点不存在，老节点存在，调用 destroy，销毁老节点
   *   2、如果 oldVnode 是真实元素，则表示首次渲染，创建新节点，并插入 body，然后移除老节点
   *   3、如果 oldVnode 不是真实元素，则表示更新阶段，执行 patchVnode
   */
  return patch
}
```

#### patch

> src/core/vdom/patch.js

```javascript
/**
 * vm.__patch__
 *   1、新节点不存在，老节点存在，调用 destroy，销毁老节点
 *   2、如果 oldVnode 是真实元素，则表示首次渲染，创建新节点，并插入 body，然后移除老节点
 *   3、如果 oldVnode 不是真实元素，则表示更新阶段，执行 patchVnode
 */
function patch(oldVnode, vnode, hydrating, removeOnly) {
  // 如果新节点不存在，老节点存在，则调用 destroy，销毁老节点
  if (isUndef(vnode)) {
    if (isDef(oldVnode)) invokeDestroyHook(oldVnode)
    return
  }

  let isInitialPatch = false
  const insertedVnodeQueue = []

  if (isUndef(oldVnode)) {
    // 新的 VNode 存在，老的 VNode 不存在，这种情况会在一个组件初次渲染的时候出现，比如：
    // <div id="app"><comp></comp></div>
    // 这里的 comp 组件初次渲染时就会走这儿
    // empty mount (likely as component), create new root element
    isInitialPatch = true
    createElm(vnode, insertedVnodeQueue)
  } else {
    // 判断 oldVnode 是否为真实元素
    const isRealElement = isDef(oldVnode.nodeType)
    if (!isRealElement && sameVnode(oldVnode, vnode)) {
      // 不是真实元素，但是老节点和新节点是同一个节点，则是更新阶段，执行 patch 更新节点
      patchVnode(oldVnode, vnode, insertedVnodeQueue, null, null, removeOnly)
    } else {
      // 是真实元素，则表示初次渲染
      if (isRealElement) {
        // 挂载到真实元素以及处理服务端渲染的情况
        // mounting to a real element
        // check if this is server-rendered content and if we can perform
        // a successful hydration.
        if (oldVnode.nodeType === 1 && oldVnode.hasAttribute(SSR_ATTR)) {
          oldVnode.removeAttribute(SSR_ATTR)
          hydrating = true
        }
        if (isTrue(hydrating)) {
          if (hydrate(oldVnode, vnode, insertedVnodeQueue)) {
            invokeInsertHook(vnode, insertedVnodeQueue, true)
            return oldVnode
          } else if (process.env.NODE_ENV !== 'production') {
            warn(
              'The client-side rendered virtual DOM tree is not matching ' +
              'server-rendered content. This is likely caused by incorrect ' +
              'HTML markup, for example nesting block-level elements inside ' +
              '<p>, or missing <tbody>. Bailing hydration and performing ' +
              'full client-side render.'
            )
          }
        }
        // 走到这儿说明不是服务端渲染，或者 hydration 失败，则根据 oldVnode 创建一个 vnode 节点
        // either not server-rendered, or hydration failed.
        // create an empty node and replace it
        oldVnode = emptyNodeAt(oldVnode)
      }

      // 拿到老节点的真实元素
      const oldElm = oldVnode.elm
      // 获取老节点的父元素，即 body
      const parentElm = nodeOps.parentNode(oldElm)

      // 基于新 vnode 创建整棵 DOM 树并插入到 body 元素下
      createElm(
        vnode,
        insertedVnodeQueue,
        // extremely rare edge case: do not insert if old element is in a
        // leaving transition. Only happens when combining transition +
        // keep-alive + HOCs. (#4590)
        oldElm._leaveCb ? null : parentElm,
        nodeOps.nextSibling(oldElm)
      )

      // 递归更新父占位符节点元素
      if (isDef(vnode.parent)) {
        let ancestor = vnode.parent
        const patchable = isPatchable(vnode)
        while (ancestor) {
          for (let i = 0; i < cbs.destroy.length; ++i) {
            cbs.destroy[i](ancestor)
          }
          ancestor.elm = vnode.elm
          if (patchable) {
            for (let i = 0; i < cbs.create.length; ++i) {
              cbs.create[i](emptyNode, ancestor)
            }
            // #6513
            // invoke insert hooks that may have been merged by create hooks.
            // e.g. for directives that uses the "inserted" hook.
            const insert = ancestor.data.hook.insert
            if (insert.merged) {
              // start at index 1 to avoid re-invoking component mounted hook
              for (let i = 1; i < insert.fns.length; i++) {
                insert.fns[i]()
              }
            }
          } else {
            registerRef(ancestor)
          }
          ancestor = ancestor.parent
        }
      }

      // 移除老节点
      if (isDef(parentElm)) {
        removeVnodes([oldVnode], 0, 0)
      } else if (isDef(oldVnode.tag)) {
        invokeDestroyHook(oldVnode)
      }
    }
  }

  invokeInsertHook(vnode, insertedVnodeQueue, isInitialPatch)
  return vnode.elm
}

```

#### invokeDestroyHook

> src/core/vdom/patch.js

```javascript
/**
 * 销毁节点：
 *   执行组件的 destroy 钩子，即执行 $destroy 方法 
 *   执行组件各个模块(style、class、directive 等）的 destroy 方法
 *   如果 vnode 还存在子节点，则递归调用 invokeDestroyHook
 */
function invokeDestroyHook(vnode) {
  let i, j
  const data = vnode.data
  if (isDef(data)) {
    if (isDef(i = data.hook) && isDef(i = i.destroy)) i(vnode)
    for (i = 0; i < cbs.destroy.length; ++i) cbs.destroy[i](vnode)
  }
  if (isDef(i = vnode.children)) {
    for (j = 0; j < vnode.children.length; ++j) {
      invokeDestroyHook(vnode.children[j])
    }
  }
}

```

#### sameVnode

> src/core/vdom/patch.js

```javascript
/**
 * 判读两个节点是否相同 
 */
function sameVnode (a, b) {
  return (
    // key 必须相同，需要注意的是 undefined === undefined => true
    a.key === b.key && (
      (
        // 标签相同
        a.tag === b.tag &&
        // 都是注释节点
        a.isComment === b.isComment &&
        // 都有 data 属性
        isDef(a.data) === isDef(b.data) &&
        // input 标签的情况
        sameInputType(a, b)
      ) || (
        // 异步占位符节点
        isTrue(a.isAsyncPlaceholder) &&
        a.asyncFactory === b.asyncFactory &&
        isUndef(b.asyncFactory.error)
      )
    )
  )
}

```

#### emptyNodeAt

> src/core/vdom/patch.js

```javascript
/**
 * 为元素(elm)创建一个空的 vnode
 */
function emptyNodeAt(elm) {
  return new VNode(nodeOps.tagName(elm).toLowerCase(), {}, [], undefined, elm)
}

```

#### createElm

> src/core/vdom/patch.js

```javascript
/**
 * 基于 vnode 创建整棵 DOM 树，并插入到父节点上
 */
function createElm(
  vnode,
  insertedVnodeQueue,
  parentElm,
  refElm,
  nested,
  ownerArray,
  index
) {
  if (isDef(vnode.elm) && isDef(ownerArray)) {
    // This vnode was used in a previous render!
    // now it's used as a new node, overwriting its elm would cause
    // potential patch errors down the road when it's used as an insertion
    // reference node. Instead, we clone the node on-demand before creating
    // associated DOM element for it.
    vnode = ownerArray[index] = cloneVNode(vnode)
  }

  vnode.isRootInsert = !nested // for transition enter check
  /**
   * 重点
   * 1、如果 vnode 是一个组件，则执行 init 钩子，创建组件实例并挂载，
   *   然后为组件执行各个模块的 create 钩子
   *   如果组件被 keep-alive 包裹，则激活组件
   * 2、如果是一个普通元素，则什么也不错
   */
  if (createComponent(vnode, insertedVnodeQueue, parentElm, refElm)) {
    return
  }

  // 获取 data 对象
  const data = vnode.data
  // 所有的孩子节点
  const children = vnode.children
  const tag = vnode.tag
  if (isDef(tag)) {
    // 未知标签
    if (process.env.NODE_ENV !== 'production') {
      if (data && data.pre) {
        creatingElmInVPre++
      }
      if (isUnknownElement(vnode, creatingElmInVPre)) {
        warn(
          'Unknown custom element: <' + tag + '> - did you ' +
          'register the component correctly? For recursive components, ' +
          'make sure to provide the "name" option.',
          vnode.context
        )
      }
    }

    // 创建新节点
    vnode.elm = vnode.ns
      ? nodeOps.createElementNS(vnode.ns, tag)
      : nodeOps.createElement(tag, vnode)
    setScope(vnode)

    // 递归创建所有子节点（普通元素、组件）
    createChildren(vnode, children, insertedVnodeQueue)
    if (isDef(data)) {
      invokeCreateHooks(vnode, insertedVnodeQueue)
    }
    // 将节点插入父节点
    insert(parentElm, vnode.elm, refElm)

    if (process.env.NODE_ENV !== 'production' && data && data.pre) {
      creatingElmInVPre--
    }
  } else if (isTrue(vnode.isComment)) {
    // 注释节点，创建注释节点并插入父节点
    vnode.elm = nodeOps.createComment(vnode.text)
    insert(parentElm, vnode.elm, refElm)
  } else {
    // 文本节点，创建文本节点并插入父节点
    vnode.elm = nodeOps.createTextNode(vnode.text)
    insert(parentElm, vnode.elm, refElm)
  }
}

```

#### createComponent

> src/core/vdom/patch.js

```javascript
/**
 * 如果 vnode 是一个组件，则执行 init 钩子，创建组件实例，并挂载
 * 然后为组件执行各个模块的 create 方法
 * @param {*} vnode 组件新的 vnode
 * @param {*} insertedVnodeQueue 数组
 * @param {*} parentElm oldVnode 的父节点
 * @param {*} refElm oldVnode 的下一个兄弟节点
 * @returns 如果 vnode 是一个组件并且组件创建成功，则返回 true，否则返回 undefined
 */
function createComponent(vnode, insertedVnodeQueue, parentElm, refElm) {
  // 获取 vnode.data 对象
  let i = vnode.data
  if (isDef(i)) {
    // 验证组件实例是否已经存在 && 被 keep-alive 包裹
    const isReactivated = isDef(vnode.componentInstance) && i.keepAlive
    // 执行 vnode.data.init 钩子函数，该函数在讲 render helper 时讲过
    // 如果是被 keep-alive 包裹的组件：则再执行 prepatch 钩子，用 vnode 上的各个属性更新 oldVnode 上的相关属性
    // 如果是组件没有被 keep-alive 包裹或者首次渲染，则初始化组件，并进入挂载阶段
    if (isDef(i = i.hook) && isDef(i = i.init)) {
      i(vnode, false /* hydrating */)
    }
    // after calling the init hook, if the vnode is a child component
    // it should've created a child instance and mounted it. the child
    // component also has set the placeholder vnode's elm.
    // in that case we can just return the element and be done.
    if (isDef(vnode.componentInstance)) {
      // 如果 vnode 是一个子组件，则调用 init 钩子之后会创建一个组件实例，并挂载
      // 这时候就可以给组件执行各个模块的的 create 钩子了
      initComponent(vnode, insertedVnodeQueue)
      // 将组件的 DOM 节点插入到父节点内
      insert(parentElm, vnode.elm, refElm)
      if (isTrue(isReactivated)) {
        // 组件被 keep-alive 包裹的情况，激活组件
        reactivateComponent(vnode, insertedVnodeQueue, parentElm, refElm)
      }
      return true
    }
  }
}

```

#### insert

> src/core/vdom/patch.js

```javascript
/**
 * 向父节点插入节点 
 */
function insert(parent, elm, ref) {
  if (isDef(parent)) {
    if (isDef(ref)) {
      if (nodeOps.parentNode(ref) === parent) {
        nodeOps.insertBefore(parent, elm, ref)
      }
    } else {
      nodeOps.appendChild(parent, elm)
    }
  }
}

```

#### removeVnodes

> src/core/vdom/patch.js

```javascript
/**
 * 移除指定索引范围（startIdx —— endIdx）内的节点 
 */
function removeVnodes(vnodes, startIdx, endIdx) {
  for (; startIdx <= endIdx; ++startIdx) {
    const ch = vnodes[startIdx]
    if (isDef(ch)) {
      if (isDef(ch.tag)) {
        removeAndInvokeRemoveHook(ch)
        invokeDestroyHook(ch)
      } else { // Text node
        removeNode(ch.elm)
      }
    }
  }
}

```

#### patchVnode

> src/core/vdom/patch.js

```javascript
/**
 * 更新节点
 *   全量的属性更新
 *   如果新老节点都有孩子，则递归执行 diff
 *   如果新节点有孩子，老节点没孩子，则新增新节点的这些孩子节点
 *   如果老节点有孩子，新节点没孩子，则删除老节点的这些孩子
 *   更新文本节点
 */
function patchVnode(
  oldVnode,
  vnode,
  insertedVnodeQueue,
  ownerArray,
  index,
  removeOnly
) {
  // 老节点和新节点相同，直接返回
  if (oldVnode === vnode) {
    return
  }

  if (isDef(vnode.elm) && isDef(ownerArray)) {
    // clone reused vnode
    vnode = ownerArray[index] = cloneVNode(vnode)
  }

  const elm = vnode.elm = oldVnode.elm

  // 异步占位符节点
  if (isTrue(oldVnode.isAsyncPlaceholder)) {
    if (isDef(vnode.asyncFactory.resolved)) {
      hydrate(oldVnode.elm, vnode, insertedVnodeQueue)
    } else {
      vnode.isAsyncPlaceholder = true
    }
    return
  }

  // 跳过静态节点的更新
  // reuse element for static trees.
  // note we only do this if the vnode is cloned -
  // if the new node is not cloned it means the render functions have been
  // reset by the hot-reload-api and we need to do a proper re-render.
  if (isTrue(vnode.isStatic) &&
    isTrue(oldVnode.isStatic) &&
    vnode.key === oldVnode.key &&
    (isTrue(vnode.isCloned) || isTrue(vnode.isOnce))
  ) {
    // 新旧节点都是静态的而且两个节点的 key 一样，并且新节点被 clone 了 或者 新节点有 v-once指令，则重用这部分节点
    vnode.componentInstance = oldVnode.componentInstance
    return
  }

  // 执行组件的 prepatch 钩子
  let i
  const data = vnode.data
  if (isDef(data) && isDef(i = data.hook) && isDef(i = i.prepatch)) {
    i(oldVnode, vnode)
  }

  // 老节点的孩子
  const oldCh = oldVnode.children
  // 新节点的孩子
  const ch = vnode.children
  // 全量更新新节点的属性，Vue 3.0 在这里做了很多的优化
  if (isDef(data) && isPatchable(vnode)) {
    // 执行新节点所有的属性更新
    for (i = 0; i < cbs.update.length; ++i) cbs.update[i](oldVnode, vnode)
    if (isDef(i = data.hook) && isDef(i = i.update)) i(oldVnode, vnode)
  }
  if (isUndef(vnode.text)) {
    // 新节点不是文本节点
    if (isDef(oldCh) && isDef(ch)) {
      // 如果新老节点都有孩子，则递归执行 diff 过程
      if (oldCh !== ch) updateChildren(elm, oldCh, ch, insertedVnodeQueue, removeOnly)
    } else if (isDef(ch)) {
      // 老孩子不存在，新孩子存在，则创建这些新孩子节点
      if (process.env.NODE_ENV !== 'production') {
        checkDuplicateKeys(ch)
      }
      if (isDef(oldVnode.text)) nodeOps.setTextContent(elm, '')
      addVnodes(elm, null, ch, 0, ch.length - 1, insertedVnodeQueue)
    } else if (isDef(oldCh)) {
      // 老孩子存在，新孩子不存在，则移除这些老孩子节点
      removeVnodes(oldCh, 0, oldCh.length - 1)
    } else if (isDef(oldVnode.text)) {
      // 老节点是文本节点，则将文本内容置空
      nodeOps.setTextContent(elm, '')
    }
  } else if (oldVnode.text !== vnode.text) {
    // 新节点是文本节点，则更新文本节点
    nodeOps.setTextContent(elm, vnode.text)
  }
  if (isDef(data)) {
    if (isDef(i = data.hook) && isDef(i = i.postpatch)) i(oldVnode, vnode)
  }
}

```

#### updateChildren

> src/core/vdom/patch.js

```javascript
/**
 * diff 过程:
 *   diff 优化：做了四种假设，假设新老节点开头结尾有相同节点的情况，一旦命中假设，就避免了一次循环，以提高执行效率
 *             如果不幸没有命中假设，则执行遍历，从老节点中找到新开始节点
 *             找到相同节点，则执行 patchVnode，然后将老节点移动到正确的位置
 *   如果老节点先于新节点遍历结束，则剩余的新节点执行新增节点操作
 *   如果新节点先于老节点遍历结束，则剩余的老节点执行删除操作，移除这些老节点
 */
function updateChildren(parentElm, oldCh, newCh, insertedVnodeQueue, removeOnly) {
  // 老节点的开始索引
  let oldStartIdx = 0
  // 新节点的开始索引
  let newStartIdx = 0
  // 老节点的结束索引
  let oldEndIdx = oldCh.length - 1
  // 第一个老节点
  let oldStartVnode = oldCh[0]
  // 最后一个老节点
  let oldEndVnode = oldCh[oldEndIdx]
  // 新节点的结束索引
  let newEndIdx = newCh.length - 1
  // 第一个新节点
  let newStartVnode = newCh[0]
  // 最后一个新节点
  let newEndVnode = newCh[newEndIdx]
  let oldKeyToIdx, idxInOld, vnodeToMove, refElm

  // removeOnly是一个特殊的标志，仅由 <transition-group> 使用，以确保被移除的元素在离开转换期间保持在正确的相对位置
  const canMove = !removeOnly

  if (process.env.NODE_ENV !== 'production') {
    // 检查新节点的 key 是否重复
    checkDuplicateKeys(newCh)
  }

  // 遍历新老两组节点，只要有一组遍历完（开始索引超过结束索引）则跳出循环
  while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
    if (isUndef(oldStartVnode)) {
      // 如果节点被移动，在当前索引上可能不存在，检测这种情况，如果节点不存在则调整索引
      oldStartVnode = oldCh[++oldStartIdx] // Vnode has been moved left
    } else if (isUndef(oldEndVnode)) {
      oldEndVnode = oldCh[--oldEndIdx]
    } else if (sameVnode(oldStartVnode, newStartVnode)) {
      // 老开始节点和新开始节点是同一个节点，执行 patch
      patchVnode(oldStartVnode, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
      // patch 结束后老开始和新开始的索引分别加 1
      oldStartVnode = oldCh[++oldStartIdx]
      newStartVnode = newCh[++newStartIdx]
    } else if (sameVnode(oldEndVnode, newEndVnode)) {
      // 老结束和新结束是同一个节点，执行 patch
      patchVnode(oldEndVnode, newEndVnode, insertedVnodeQueue, newCh, newEndIdx)
      // patch 结束后老结束和新结束的索引分别减 1
      oldEndVnode = oldCh[--oldEndIdx]
      newEndVnode = newCh[--newEndIdx]
    } else if (sameVnode(oldStartVnode, newEndVnode)) { // Vnode moved right
      // 老开始和新结束是同一个节点，执行 patch
      patchVnode(oldStartVnode, newEndVnode, insertedVnodeQueue, newCh, newEndIdx)
      // 处理被 transtion-group 包裹的组件时使用
      canMove && nodeOps.insertBefore(parentElm, oldStartVnode.elm, nodeOps.nextSibling(oldEndVnode.elm))
      // patch 结束后老开始索引加 1，新结束索引减 1
      oldStartVnode = oldCh[++oldStartIdx]
      newEndVnode = newCh[--newEndIdx]
    } else if (sameVnode(oldEndVnode, newStartVnode)) { // Vnode moved left
      // 老结束和新开始是同一个节点，执行 patch
      patchVnode(oldEndVnode, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
      canMove && nodeOps.insertBefore(parentElm, oldEndVnode.elm, oldStartVnode.elm)
      // patch 结束后，老结束的索引减 1，新开始的索引加 1
      oldEndVnode = oldCh[--oldEndIdx]
      newStartVnode = newCh[++newStartIdx]
    } else {
      // 如果上面的四种假设都不成立，则通过遍历找到新开始节点在老节点中的位置索引

      // 找到老节点中每个节点 key 和 索引之间的关系映射 => oldKeyToIdx = { key1: idx1, ... }
      if (isUndef(oldKeyToIdx)) oldKeyToIdx = createKeyToOldIdx(oldCh, oldStartIdx, oldEndIdx)
      // 在映射中找到新开始节点在老节点中的位置索引
      idxInOld = isDef(newStartVnode.key)
        ? oldKeyToIdx[newStartVnode.key]
        : findIdxInOld(newStartVnode, oldCh, oldStartIdx, oldEndIdx)
      if (isUndef(idxInOld)) { // New element
        // 在老节点中没找到新开始节点，则说明是新创建的元素，执行创建
        createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx)
      } else {
        // 在老节点中找到新开始节点了
        vnodeToMove = oldCh[idxInOld]
        if (sameVnode(vnodeToMove, newStartVnode)) {
          // 如果这两个节点是同一个，则执行 patch
          patchVnode(vnodeToMove, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
          // patch 结束后将该老节点置为 undefined
          oldCh[idxInOld] = undefined
          canMove && nodeOps.insertBefore(parentElm, vnodeToMove.elm, oldStartVnode.elm)
        } else {
          // 最后这种情况是，找到节点了，但是发现两个节点不是同一个节点，则视为新元素，执行创建
          createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx)
        }
      }
      // 老节点向后移动一个
      newStartVnode = newCh[++newStartIdx]
    }
  }
  // 走到这里，说明老姐节点或者新节点被遍历完了
  if (oldStartIdx > oldEndIdx) {
    // 说明老节点被遍历完了，新节点有剩余，则说明这部分剩余的节点是新增的节点，然后添加这些节点
    refElm = isUndef(newCh[newEndIdx + 1]) ? null : newCh[newEndIdx + 1].elm
    addVnodes(parentElm, refElm, newCh, newStartIdx, newEndIdx, insertedVnodeQueue)
  } else if (newStartIdx > newEndIdx) {
    // 说明新节点被遍历完了，老节点有剩余，说明这部分的节点被删掉了，则移除这些节点
    removeVnodes(oldCh, oldStartIdx, oldEndIdx)
  }
}

```

#### checkDuplicateKeys

> src/core/vdom/patch.js

```javascript
/**
 * 检查一组元素的 key 是否重复 
 */
function checkDuplicateKeys(children) {
  const seenKeys = {}
  for (let i = 0; i < children.length; i++) {
    const vnode = children[i]
    const key = vnode.key
    if (isDef(key)) {
      if (seenKeys[key]) {
        warn(
          `Duplicate keys detected: '${key}'. This may cause an update error.`,
          vnode.context
        )
      } else {
        seenKeys[key] = true
      }
    }
  }
}

```

#### addVnodes

> src/core/vdom/patch.js

```javascript
/**
 * 在指定索引范围（startIdx —— endIdx）内添加节点
 */
function addVnodes(parentElm, refElm, vnodes, startIdx, endIdx, insertedVnodeQueue) {
  for (; startIdx <= endIdx; ++startIdx) {
    createElm(vnodes[startIdx], insertedVnodeQueue, parentElm, refElm, false, vnodes, startIdx)
  }
}

```

#### createKeyToOldIdx

> src/core/vdom/patch.js

```javascript
/**
 * 得到指定范围（beginIdx —— endIdx）内节点的 key 和 索引之间的关系映射 => { key1: idx1, ... }
 */
function createKeyToOldIdx(children, beginIdx, endIdx) {
  let i, key
  const map = {}
  for (i = beginIdx; i <= endIdx; ++i) {
    key = children[i].key
    if (isDef(key)) map[key] = i
  }
  return map
}

```

#### findIdxInOld

> src/core/vdom/patch.js

```javascript
/**
  * 找到新节点（vnode）在老节点（oldCh）中的位置索引 
  */
function findIdxInOld(node, oldCh, start, end) {
  for (let i = start; i < end; i++) {
    const c = oldCh[i]
    if (isDef(c) && sameVnode(node, c)) return i
  }
}

```

#### invokeCreateHooks

> src/core/vdom/patch.js

```javascript
/**
 * 调用 各个模块的 create 方法，比如创建属性的、创建样式的、指令的等等 ，然后执行组件的 mounted 生命周期方法
 */
function invokeCreateHooks(vnode, insertedVnodeQueue) {
  for (let i = 0; i < cbs.create.length; ++i) {
    cbs.create[i](emptyNode, vnode)
  }
  // 组件钩子
  i = vnode.data.hook // Reuse variable
  if (isDef(i)) {
    // 组件好像没有 create 钩子
    if (isDef(i.create)) i.create(emptyNode, vnode)
    // 调用组件的 insert 钩子，执行组件的 mounted 生命周期方法
    if (isDef(i.insert)) insertedVnodeQueue.push(vnode)
  }
}

```

#### createChildren

> src/core/vdom/patch.js

```javascript
/**
 * 创建所有子节点，并将子节点插入父节点，形成一棵 DOM 树
 */
function createChildren(vnode, children, insertedVnodeQueue) {
  if (Array.isArray(children)) {
    // children 是数组，表示是一组节点
    if (process.env.NODE_ENV !== 'production') {
      // 检测这组节点的 key 是否重复
      checkDuplicateKeys(children)
    }
    // 遍历这组节点，依次创建这些节点然后插入父节点，形成一棵 DOM 树
    for (let i = 0; i < children.length; ++i) {
      createElm(children[i], insertedVnodeQueue, vnode.elm, null, true, children, i)
    }
  } else if (isPrimitive(vnode.text)) {
    // 说明是文本节点，创建文本节点，并插入父节点
    nodeOps.appendChild(vnode.elm, nodeOps.createTextNode(String(vnode.text)))
  }
}

```

## 总结

* **面试官 问**：你能说一说 Vue 的 patch 算法吗？

  **答**：

  Vue 的 patch 算法有三个作用：负责首次渲染和后续更新或者销毁组件

  * 如果老的 VNode 是真实元素，则表示首次渲染，创建整棵 DOM 树，并插入 body，然后移除老的模版节点

  * 如果老的 VNode 不是真实元素，并且新的 VNode 也存在，则表示更新阶段，执行 patchVnode

    * 首先是全量更新所有的属性
  
    * 如果新老 VNode 都有孩子，则递归执行 updateChildren，进行 diff 过程
  
      > 针对前端操作 DOM 节点的特点进行如下优化：
    
      * 同层比较（降低时间复杂度）深度优先（递归）
      
      * 而且前端很少有完全打乱节点顺序的情况，所以做了四种假设，假设新老 VNode 的开头结尾存在相同节点，一旦命中假设，就避免了一次循环，降低了 diff 的时间复杂度，提高执行效率。如果不幸没有命中假设，则执行遍历，从老的 VNode 中找到新的 VNode 的开始节点
    
      * 找到相同节点，则执行 patchVnode，然后将老节点移动到正确的位置
      
      * 如果老的 VNode 先于新的 VNode 遍历结束，则剩余的新的 VNode 执行新增节点操作
    
      * 如果新的 VNode 先于老的 VNode 遍历结束，则剩余的老的 VNode 执行删除操纵，移除这些老节点
  
    * 如果新的 VNode 有孩子，老的 VNode 没孩子，则新增这些新孩子节点
  
    * 如果老的 VNode 有孩子，新的 VNode 没孩子，则删除这些老孩子节点
  
    * 剩下一种就是更新文本节点

  * 如果新的 VNode 不存在，老的 VNode 存在，则调用 destroy，销毁老节点
  
<hr />

好了，到这里，Vue 源码解读系列就结束了，如果你认认真真的读完整个系列的文章，相信你对 Vue 源码已经相当熟悉了，不论是从宏观层面理解，还是某些细节方面的详解，应该都没问题。即使有些细节现在不清楚，但是当遇到问题时，你也能一眼看出来该去源码的什么位置去找答案。

到这里你可以试着在自己的脑海中复述一下 Vue 的整个执行流程。过程很重要，但 **总结** 才是最后的升华时刻。如果在哪个环节卡住了，可再回去读相应的部分就可以了。

还记得系列的第一篇文章中提到的目标吗？相信阅读几遍下来，你一定可以在自己的简历中写到：**精通 Vue 框架的源码原理**。

接下来会开始 Vue 的手写系列。

## 链接

* [配套视频，微信公众号回复](https://gitee.com/liyongning/typora-image-bed/raw/master/202202171742614.jpg)："精通 Vue 技术栈源码原理视频版" 获取
* [精通 Vue 技术栈源码原理 专栏](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzA3NTk4NjQ1OQ==&action=getalbum&album_id=2273541436891693065#wechat_redirect)
* [github 仓库 liyongning/Vue](https://github.com/liyongning/Vue) 欢迎 Star



感谢各位的：**关注**、**点赞**、**收藏**和**评论**，我们下期见。

***

**当学习成为了习惯，知识也就变成了常识。** 感谢各位的 **关注**、 **点赞**、**收藏**和**评论**。

新视频和文章会第一时间在微信公众号发送，欢迎关注：[李永宁lyn](https://gitee.com/liyongning/typora-image-bed/raw/master/202202171742614.jpg)

文章已收录到 [github 仓库 liyongning/blog](https://github.com/liyongning/blog)，欢迎 Watch 和 Star。