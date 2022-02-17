# 手写 Vue2 系列 之 初始渲染

**当学习成为了习惯，知识也就变成了常识。** 感谢各位的 **关注**、**点赞**、**收藏**和**评论**。

新视频和文章会第一时间在微信公众号发送，欢迎关注：[李永宁lyn](https://gitee.com/liyongning/typora-image-bed/raw/master/202202171742614.jpg)

文章已收录到 [github 仓库 liyongning/blog](https://github.com/liyongning/blog)，欢迎 Watch 和 Star。

## 封面

![](https://gitee.com/liyongning/typora-image-bed/raw/master/202203141833875.png)

## 前言

上一篇文章 [手写 Vue2 系列 之 编译器](https://mp.weixin.qq.com/s?__biz=MzA3NTk4NjQ1OQ==&mid=2247485292&idx=1&sn=8702868bcbf2ca91980ab6493284188b&chksm=9f696418a81eed0e0ee08d630613e4905a9e9fe77b27fba413265e340ad09188c15e1c0bdfa5#rd) 中完成了从模版字符串到 render 函数的工作。当我们得到 render 函数之后，接下来就该进入到真正的挂载阶段了：

挂载 -> 实例化渲染 Watcher -> 执行 updateComponent 方法 -> 执行 render 函数生成 VNode -> 执行 patch 进行首次渲染 -> 递归遍历 VNode 创建各个节点并处理节点上的普通属性和指令 -> 如果节点是自定义组件则创建组件实例 -> 进行组件的初始化、挂载 -> 最终所有 VNode 变成真实的 DOM 节点并替换掉页面上的模版内容 -> 完成初始渲染

## 目标

所以，本篇文章目标就是实现上面描述的整个过成，完成初始渲染。整个过程中涉及如下知识点：

* render helper

* VNode

* patch 初始渲染

* 指令(v-model、v-bind、v-on)的处理

* 实例化子组件

* 插槽的处理

## 实现

接下来就正式进入代码实现过程，一步步实现上述所有内容，完成页面的初始渲染。

### mount

> /src/compiler/index.js

```javascript
/**
 * 编译器
 */
export default function mount(vm) {
  if (!vm.$options.render) { // 没有提供 render 选项，则编译生成 render 函数
    // ...
  }
  mountComponent(vm)
}

```

### mountComponent

> /src/compiler/mountComponent.js

```javascript
/**
 * @param {*} vm Vue 实例
 */
export default function mountComponent(vm) {
  // 更新组件的的函数
  const updateComponent = () => {
    vm._update(vm._render())
  }

  // 实例化一个渲染 Watcher，当响应式数据更新时，这个更新函数会被执行
  new Watcher(updateComponent)
}

```

### vm._render

> /src/compiler/mountComponent.js

```javascript
/**
 * 负责执行 vm.$options.render 函数
 */
Vue.prototype._render = function () {
  // 给 render 函数绑定 this 上下文为 Vue 实例
  return this.$options.render.apply(this)
}

```

#### render helper

> /src/compiler/renderHelper.js

```javascript
/**
 * 在 Vue 实例上安装运行时的渲染帮助函数，比如 _c、_v，这些函数会生成 Vnode
 * @param {VueContructor} target Vue 实例
 */
export default function renderHelper(target) {
  target._c = createElement
  target._v = createTextNode
}

```

#### createElement

> /src/compiler/renderHelper.js

```javascript
/**
 * 根据标签信息创建 Vnode
 * @param {string} tag 标签名 
 * @param {Map} attr 标签的属性 Map 对象
 * @param {Array<Render>} children 所有的子节点的渲染函数
 */
function createElement(tag, attr, children) {
  return VNode(tag, attr, children, this)
}

```

#### createTextNode

> /src/compiler/renderHelper.js

```javascript
/**
 * 生成文本节点的 VNode
 * @param {*} textAst 文本节点的 AST 对象
 */
function createTextNode(textAst) {
  return VNode(null, null, null, this, textAst)
}

```

#### VNode

> /src/compiler/vnode.js

```javascript
/**
 * VNode
 * @param {*} tag 标签名
 * @param {*} attr 属性 Map 对象
 * @param {*} children 子节点组成的 VNode
 * @param {*} text 文本节点的 ast 对象
 * @param {*} context Vue 实例
 * @returns VNode
 */
export default function VNode(tag, attr, children, context, text = null) {
  return {
    // 标签
    tag,
    // 属性 Map 对象
    attr,
    // 父节点
    parent: null,
    // 子节点组成的 Vnode 数组
    children,
    // 文本节点的 Ast 对象
    text,
    // Vnode 的真实节点
    elm: null,
    // Vue 实例
    context
  }
}

```

### vm._update

> /src/compiler/mountComponent.js

```javascript
Vue.prototype._update = function (vnode) {
  // 老的 VNode
  const prevVNode = this._vnode
  // 新的 VNode
  this._vnode = vnode
  if (!prevVNode) {
    // 老的 VNode 不存在，则说明时首次渲染根组件
    this.$el = this.__patch__(this.$el, vnode)
  } else {
    // 后续更新组件或者首次渲染子组件，都会走这里
    this.$el = this.__patch__(prevVNode, vnode)
  }
}

```

### 安装 \_\_patch\_\_、render helper

> /src/index.js

```javascript
/**
 * 初始化配置对象
 * @param {*} options 
 */
Vue.prototype._init = function (options) {
  // ...
  initData(this)
  // 安装运行时的渲染工具函数
  renderHelper(this)
  // 在实例上安装 patch 函数
  this.__patch__ = patch
  // 如果存在 el 配置项，则调用 $mount 方法编译模版
  if (this.$options.el) {
    this.$mount()
  }
}

```

### patch

> /src/compiler/patch.js

```javascript
/**
 * 初始渲染和后续更新的入口
 * @param {VNode} oldVnode 老的 VNode
 * @param {VNode} vnode 新的 VNode
 * @returns VNode 的真实 DOM 节点
 */
export default function patch(oldVnode, vnode) {
  if (oldVnode && !vnode) {
    // 老节点存在，新节点不存在，则销毁组件
    return
  }

  if (!oldVnode) { // oldVnode 不存在，说明是子组件首次渲染
    createElm(vnode)
  } else {
    if (oldVnode.nodeType) { // 真实节点，则表示首次渲染根组件
      // 父节点，即 body
      const parent = oldVnode.parentNode
      // 参考节点，即老的 vnode 的下一个节点 —— script，新节点要插在 script 的前面
      const referNode = oldVnode.nextSibling
      // 创建元素
      createElm(vnode, parent, referNode)
      // 移除老的 vnode
      parent.removeChild(oldVnode)
    } else {
      console.log('update')
    }
  }
  return vnode.elm
}

```

### createElm

> /src/compiler/patch.js

```javascript
/**
 * 创建元素
 * @param {*} vnode VNode
 * @param {*} parent VNode 的父节点，真实节点
 * @returns 
 */
function createElm(vnode, parent, referNode) {
  // 记录节点的父节点
  vnode.parent = parent
  // 创建自定义组件，如果是非组件，则会继续后面的流程
  if (createComponent(vnode)) return

  const { attr, children, text } = vnode
  if (text) { // 文本节点
    // 创建文本节点，并插入到父节点内
    vnode.elm = createTextNode(vnode)
  } else { // 元素节点
    // 创建元素，在 vnode 上记录对应的 dom 节点
    vnode.elm = document.createElement(vnode.tag)
    // 给元素设置属性
    setAttribute(attr, vnode)
    // 递归创建子节点
    for (let i = 0, len = children.length; i < len; i++) {
      createElm(children[i], vnode.elm)
    }
  }
  // 如果存在 parent，则将创建的节点插入到父节点内
  if (parent) {
    const elm = vnode.elm
    if (referNode) {
      parent.insertBefore(elm, referNode)
    } else {
      parent.appendChild(elm)
    }
  }
}

```

#### createTextNode

> /src/compiler/patch.js

```javascript
/**
 * 创建文本节点
 * @param {*} textVNode 文本节点的 VNode
 */
function createTextNode(textVNode) {
  let { text } = textVNode, textNode = null
  if (text.expression) {
    // 存在表达式，这个表达式的值是一个响应式数据
    const value = textVNode.context[text.expression]
    textNode = document.createTextNode(typeof value === 'object' ? JSON.stringify(value) : String(value))
  } else {
    // 纯文本
    textNode = document.createTextNode(text.text)
  }
  return textNode
}

```

#### setAttribute

> /src/compiler/patch.js

```javascript
/**
 * 给节点设置属性
 * @param {*} attr 属性 Map 对象
 * @param {*} vnode
 */
function setAttribute(attr, vnode) {
  // 遍历属性，如果是普通属性，直接设置，如果是指令，则特殊处理
  for (let name in attr) {
    if (name === 'vModel') {
      // v-model 指令
      const { tag, value } = attr.vModel
      setVModel(tag, value, vnode)
    } else if (name === 'vBind') {
      // v-bind 指令
      setVBind(vnode)
    } else if (name === 'vOn') {
      // v-on 指令
      setVOn(vnode)
    } else {
      // 普通属性
      vnode.elm.setAttribute(name, attr[name])
    }
  }
}

```

##### setVModel

> /src/compiler/patch.js

```javascript
/**
 * v-model 的原理
 * @param {*} tag 节点的标签名
 * @param {*} value 属性值
 * @param {*} node 节点
 */
function setVModel(tag, value, vnode) {
  const { context: vm, elm } = vnode
  if (tag === 'select') {
    // 下拉框，<select></select>
    Promise.resolve().then(() => {
      // 利用 promise 延迟设置，直接设置不行，
      // 因为这会儿 option 元素还没创建
      elm.value = vm[value]
    })
    elm.addEventListener('change', function () {
      vm[value] = elm.value
    })
  } else if (tag === 'input' && vnode.elm.type === 'text') {
    // 文本框，<input type="text" />
    elm.value = vm[value]
    elm.addEventListener('input', function () {
      vm[value] = elm.value
    })
  } else if (tag === 'input' && vnode.elm.type === 'checkbox') {
    // 选择框，<input type="checkbox" />
    elm.checked = vm[value]
    elm.addEventListener('change', function () {
      vm[value] = elm.checked
    })
  }
}

```

##### setVBind

> /src/compiler/patch.js

```javascript
/**
 * v-bind 原理
 * @param {*} vnode
 */
function setVBind(vnode) {
  const { attr: { vBind }, elm, context: vm } = vnode
  for (let attrName in vBind) {
    elm.setAttribute(attrName, vm[vBind[attrName]])
    elm.removeAttribute(`v-bind:${attrName}`)
  }
}

```

##### setVOn

> /src/compiler/patch.js

```javascript
/**
 * v-on 原理
 * @param {*} vnode 
 */
function setVOn(vnode) {
  const { attr: { vOn }, elm, context: vm } = vnode
  for (let eventName in vOn) {
    elm.addEventListener(eventName, function (...args) {
      vm.$options.methods[vOn[eventName]].apply(vm, args)
    })
  }
}

```

#### createComponent

> /src/compiler/patch.js

```javascript
/**
 * 创建自定义组件
 * @param {*} vnode
 */
function createComponent(vnode) {
  if (vnode.tag && !isReserveTag(vnode.tag)) { // 非保留节点，则说明是组件
    // 获取组件配置信息
    const { tag, context: { $options: { components } } } = vnode
    const compOptions = components[tag]
    const compIns = new Vue(compOptions)
    // 将父组件的 VNode 放到子组件的实例上
    compIns._parentVnode = vnode
    // 挂载子组件
    compIns.$mount()
    // 记录子组件 vnode 的父节点信息
    compIns._vnode.parent = vnode.parent
    // 将子组件添加到父节点内
    vnode.parent.appendChild(compIns._vnode.elm)
    return true
  }
}

```

### isReserveTag

> /src/utils.js

```javascript
/**
 * 是否为平台保留节点
 */
export function isReserveTag(tagName) {
  const reserveTag = ['div', 'h3', 'span', 'input', 'select', 'option', 'p', 'button', 'template']
  return reserveTag.includes(tagName)
}

```

### 插槽原理

以下示例是插槽的常用方式。插槽的原理其实很简单，只是实现起来稍微有些麻烦罢了。

* 解析

  如果组件标签有子节点，在解析的时候将这些子节点，解析成一个特定的数据结构，该结构中包含了插槽的全部信息，然后将该数据结构放到父节点的属性上，其实就是找个地方存放这些信息，然后在 renderSlot 中使用时取出来。当然这个解析过程是发生在父组件的解析过程中的。

* 生成渲染函数

  在生成子组件的渲染函数阶段，如果碰到 slot 标签，则返回一个 `_t` 的渲染函数，函数接收两个参数：属性的 JSON 字符串形式，slot 标签的所有子节点的渲染函数组成的 children 数组。
  
* render helper

  在执行子组件的渲染函数时，如果执行到 `vm._t`，就会调用 `renderSlot` 方法，该方法会返回插槽的 VNode，然后进入子组件的 patch 阶段，将这些 VNode 变成真实的 DOM 并渲染到页面上。
  

以上就是插槽的原理，然后接下来实现的时候，在某些地方可能会稍微有点绕，多多少少是因为整体架构存在一些问题，所以里面会有一些修补性质的代码，这些代码你可以理解为为了实现插槽功能，而写的一点业务代码。你只需要把住插槽的本质即可。

#### 示例

```html
<!-- comp -->
<template>
  <div>
    <div>
      <slot name="slot1">
        <span>插槽默认内容</span>
      </slot>
    </div>
      <slot name="slot2" v-bind:test="xx">
        <span>插槽默认内容</span>
      </slot>
    <div>
    </div>
  </div>
</template>

```

```html
<comp></comp>
```

```html
<comp>
  <template v-slot:slot2="xx">
    <div>作用域插槽，通过插槽从父组件给子组件传递内容</div>
  </template>
<comp>
```

#### parse

> /src/compiler/parse.js

```javascript
function processElement() {
    // ...

    // 处理插槽内容
    processSlotContent(curEle)

    // 节点处理完以后让其和父节点产生关系
    if (stackLen) {
      stack[stackLen - 1].children.push(curEle)
      curEle.parent = stack[stackLen - 1]
      // 如果节点存在 slotName，则说明该节点是组件传递给插槽的内容
      // 将插槽信息放到组件节点的 rawAttr.scopedSlots 对象上
      // 而这些信息在生成组件插槽的 VNode 时（renderSlot）会用到
      if (curEle.slotName) {
        const { parent, slotName, scopeSlot, children } = curEle
        // 这里关于 children 的操作，只是单纯为了避开 JSON.stringify 的循环引用问题
        // 因为生成渲染函数时需要对 attr 执行 JSON.stringify 方法
        const slotInfo = {
          slotName, scopeSlot, children: children.map(item => {
            delete item.parent
            return item
          })
        }
        if (parent.rawAttr.scopedSlots) {
          parent.rawAttr.scopedSlots[curEle.slotName] = slotInfo
        } else {
          parent.rawAttr.scopedSlots = { [curEle.slotName]: slotInfo }
        }
      }
    }
  }

```

#### processSlotContent

> /src/compiler/parse.js

```javascript
/**
 * 处理插槽
 * <scope-slot>
 *   <template v-slot:default="scopeSlot">
 *     <div>{{ scopeSlot }}</div>
 *   </template>
 * </scope-slot>
 * @param { AST } el 节点的 AST 对象
 */
function processSlotContent(el) {
  // 注意，具有 v-slot:xx 属性的 template 只能是组件的根元素，这里不做判断
  if (el.tag === 'template') { // 获取插槽信息
    // 属性 map 对象
    const attrMap = el.rawAttr
    // 遍历属性 map 对象，找出其中的 v-slot 指令信息
    for (let key in attrMap) {
      if (key.match(/v-slot:(.*)/)) { // 说明 template 标签上 v-slot 指令
        // 获取指令后的插槽名称和值，比如: v-slot:default=xx
        // default
        const slotName = el.slotName = RegExp.$1
        // xx
        el.scopeSlot = attrMap[`v-slot:${slotName}`]
        // 直接 return，因为该标签上只可能有一个 v-slot 指令
        return
      }
    }
  }
}

```

#### generate

> /src/compiler/generate.js

```javascript
/**
 * 解析 ast 生成 渲染函数
 * @param {*} ast 语法树 
 * @returns {string} 渲染函数的字符串形式
 */
function genElement(ast) {
  // ...

  // 处理子节点，得到一个所有子节点渲染函数组成的数组
  const children = genChildren(ast)

  if (tag === 'slot') {
    // 生成插槽的处理函数
    return `_t(${JSON.stringify(attrs)}, [${children}])`
  }

  // 生成 VNode 的可执行方法
  return `_c('${tag}', ${JSON.stringify(attrs)}, [${children}])`
}

```

##### renderHelper

> /src/compiler/renderHelper.js

```javascript
/**
 * 在 Vue 实例上安装运行时的渲染帮助函数，比如 _c、_v，这些函数会生成 Vnode
 * @param {VueContructor} target Vue 实例
 */
export default function renderHelper(target) {
  // ...
  target._t = renderSlot
}

```

##### renderSlot

> /src/compiler/renderHelper.js

```javascript
/**
 * 插槽的原理其实很简单，难点在于实现
 * 其原理就是生成 VNode，难点在于生成 VNode 之前的各种解析，也就是数据准备阶段
 * 生成插槽的的 VNode
 * @param {*} attrs 插槽的属性
 * @param {*} children 插槽所有子节点的 ast 组成的数组
 */
function renderSlot(attrs, children) {
  // 父组件 VNode 的 attr 信息
  const parentAttr = this._parentVnode.attr
  let vnode = null
  if (parentAttr.scopedSlots) { // 说明给当前组件的插槽传递了内容
    // 获取插槽信息
    const slotName = attrs.name
    const slotInfo = parentAttr.scopedSlots[slotName]
    // 这里的逻辑稍微有点绕，建议打开调试，查看一下数据结构，理清对应的思路
    // 这里比较绕的逻辑完全是为了实现插槽这个功能，和插槽本身的原理没关系
    this[slotInfo.scopeSlot] = this[Object.keys(attrs.vBind)[0]]
    vnode = genVNode(slotInfo.children, this)
  } else { // 插槽默认内容
    // 将 children 变成 vnode 数组
    vnode = genVNode(children, this)
  }

  // 如果 children 长度为 1，则说明插槽只有一个子节点
  if (children.length === 1) return vnode[0]
  return createElement.call(this, 'div', {}, vnode)
}

```

##### genVNode

> /src/compiler/renderHelper.js

```javascript
/**
 * 将一批 ast 节点(数组)转换成 vnode 数组
 * @param {Array<Ast>} childs 节点数组
 * @param {*} vm 组件实例
 * @returns vnode 数组
 */
function genVNode(childs, vm) {
  const vnode = []
  for (let i = 0, len = childs.length; i < len; i++) {
    const { tag, attr, children, text } = childs[i]
    if (text) { // 文本节点
      if (typeof text === 'string') { // text 为字符串
        // 构造文本节点的 AST 对象
        const textAst = {
          type: 3,
          text,
        }
        if (text.match(/{{(.*)}}/)) {
          // 说明是表达式
          textAst.expression = RegExp.$1.trim()
        }
        vnode.push(createTextNode.call(vm, textAst))
      } else { // text 为文本节点的 ast 对象
        vnode.push(createTextNode.call(vm, text))
      }
    } else { // 元素节点
      vnode.push(createElement.call(vm, tag, attr, genVNode(children, vm)))
    }
  }
  return vnode
}

```

## 结果

好了，到这里，模版的初始渲染就已经完成了，如果你能看到如下效果图，则说明一切正常。因为整个过程涉及的内容还是比较多的，如果觉得某些地方不太清楚，建议再看看，仔细梳理下整个流程。

动图链接：https://gitee.com/liyongning/typora-image-bed/raw/master/202203141833484.image

![Jun-18-2021 07-35-17.gif](https://gitee.com/liyongning/typora-image-bed/raw/master/202203141833484.image)

可以看到，原始标签、自定义组件、插槽都已经完整的渲染到了页面上，完成了初始渲染之后，接下来就该去实现后续的更新过程了，也就是下一篇 **手写 Vue2 系列 之 patch —— diff**。

## 链接

* [配套视频，微信公众号回复](https://gitee.com/liyongning/typora-image-bed/raw/master/202202171742614.jpg)："精通 Vue 技术栈源码原理视频版" 获取
* [精通 Vue 技术栈源码原理 专栏](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzA3NTk4NjQ1OQ==&action=getalbum&album_id=2273541436891693065#wechat_redirect)
* [github 仓库 liyongning/Vue](https://github.com/liyongning/Vue) 欢迎 Star
* [github 仓库 liyongning/Lyn-Vue-DOM](https://github.com/liyongning/Lyn-Vue-DOM) 欢迎 Star
* [github 仓库 liyongning/Lyn-Vue-Template](https://github.com/liyongning/Lyn-Vue-Template) 欢迎 Star



感谢各位的：**关注**、**点赞**、**收藏**和**评论**，我们下期见。

***

**当学习成为了习惯，知识也就变成了常识。** 感谢各位的 **关注**、 **点赞**、**收藏**和**评论**。

新视频和文章会第一时间在微信公众号发送，欢迎关注：[李永宁lyn](https://gitee.com/liyongning/typora-image-bed/raw/master/202202171742614.jpg)

文章已收录到 [github 仓库 liyongning/blog](https://github.com/liyongning/blog)，欢迎 Watch 和 Star。