# Vue 源码解读（10）—— 编译器 之 生成渲染函数

**当学习成为了习惯，知识也就变成了常识。** 感谢各位的 **关注**、**点赞**、**收藏**和**评论**。

新视频和文章会第一时间在微信公众号发送，欢迎关注：[李永宁lyn](https://gitee.com/liyongning/typora-image-bed/raw/master/202202171742614.jpg)

文章已收录到 [github 仓库 liyongning/blog](https://github.com/liyongning/blog)，欢迎 Watch 和 Star。

## 封面

![](https://gitee.com/liyongning/typora-image-bed/raw/master/202203031720542.png)

## 前言

这篇文章是 Vue 编译器的最后一部分，前两部分分别是：[Vue 源码解读（8）—— 编译器 之 解析](https://mp.weixin.qq.com/s?__biz=MzA3NTk4NjQ1OQ==&mid=2247485080&idx=1&sn=db5351d2f6d9a6f3d2a87c8f4ad70157&chksm=9f6965eca81eecfac96f7034651b37c8e9b35eca024075e0bf9e24ecda5993d478f0674183ee#rd)、[Vue 源码解读（9）—— 编译器 之 优化](https://mp.weixin.qq.com/s?__biz=MzA3NTk4NjQ1OQ==&mid=2247485109&idx=1&sn=e138fb0ba4d9791a11ad0caf8942a33b&chksm=9f6965c1a81eecd7181f09c9eafd2542612b9fc6b0b4efbe21f8708ba947220889ec432dbb83#rd)。

从 HTML 模版字符串开始，解析所有标签以及标签上的各个属性，得到 AST 语法树，然后基于 AST 语法树进行静态标记，首先标记每个节点是否为静态静态，然后进一步标记出静态根节点。这样在后续的更新中就可以跳过这些静态根节点的更新，从而提高性能。

这最后一部分讲的是如何从 AST 生成渲染函数。

## 目标

深入理解渲染函数的生成过程，理解编译器是如何将 AST 变成运行时的代码，也就是我们写的类 html 模版最终变成了什么？

## 源码解读

### 入口

> /src/compiler/index.js

```javascript
/**
 * 在这之前做的所有的事情，只有一个目的，就是为了构建平台特有的编译选项（options），比如 web 平台
 * 
 * 1、将 html 模版解析成 ast
 * 2、对 ast 树进行静态标记
 * 3、将 ast 生成渲染函数
 *    静态渲染函数放到  code.staticRenderFns 数组中
 *    code.render 为动态渲染函数
 *    在将来渲染时执行渲染函数得到 vnode
 */
export const createCompiler = createCompilerCreator(function baseCompile (
  template: string,
  options: CompilerOptions
): CompiledResult {
  // 将模版解析为 AST，每个节点的 ast 对象上都设置了元素的所有信息，比如，标签信息、属性信息、插槽信息、父节点、子节点等。
  // 具体有那些属性，查看 options.start 和 options.end 这两个处理开始和结束标签的方法
  const ast = parse(template.trim(), options)
  // 优化，遍历 AST，为每个节点做静态标记
  // 标记每个节点是否为静态节点，然后进一步标记出静态根节点
  // 这样在后续更新中就可以跳过这些静态节点了
  // 标记静态根，用于生成渲染函数阶段，生成静态根节点的渲染函数
  if (options.optimize !== false) {
    optimize(ast, options)
  }
  // 代码生成，将 ast 转换成可执行的 render 函数的字符串形式
  // code = {
  //   render: `with(this){return ${_c(tag, data, children, normalizationType)}}`,
  //   staticRenderFns: [_c(tag, data, children, normalizationType), ...]
  // }
  const code = generate(ast, options)
  return {
    ast,
    render: code.render,
    staticRenderFns: code.staticRenderFns
  }
})

```

### generate

> /src/compiler/codegen/index.js

```javascript
/**
 * 从 AST 生成渲染函数
 * @returns {
 *   render: `with(this){return _c(tag, data, children)}`,
 *   staticRenderFns: state.staticRenderFns
 * } 
 */
export function generate(
  ast: ASTElement | void,
  options: CompilerOptions
): CodegenResult {
  // 实例化 CodegenState 对象，生成代码的时候需要用到其中的一些东西
  const state = new CodegenState(options)
  // 生成字符串格式的代码，比如：'_c(tag, data, children, normalizationType)'
  // data 为节点上的属性组成 JSON 字符串，比如 '{ key: xx, ref: xx, ... }'
  // children 为所有子节点的字符串格式的代码组成的字符串数组，格式：
  //     `['_c(tag, data, children)', ...],normalizationType`，
  //     最后的 normalization 是 _c 的第四个参数，
  //     表示节点的规范化类型，不是重点，不需要关注
  // 当然 code 并不一定就是 _c，也有可能是其它的，比如整个组件都是静态的，则结果就为 _m(0)
  const code = ast ? genElement(ast, state) : '_c("div")'
  return {
    render: `with(this){return ${code}}`,
    staticRenderFns: state.staticRenderFns
  }
}

```

### genElement

> /src/compiler/codegen/index.js

> **阅读建议**：
>
> 先读最后的 else 模块生成 code 的语句部分，即处理自定义组件和原生标签的 else 分支，理解最终生成的数据格式是什么样的；然后再回头阅读 `genChildren` 和 `genData`，先读 `genChildren`，代码量少，彻底理解最终生成的数据结构，最后再从上到下去阅读其它的分支。
>
> 在阅读以下代码时，请把 [Vue 源码解读（8）—— 编译器 之 解析（下）](https://mp.weixin.qq.com/s?__biz=MzA3NTk4NjQ1OQ==&mid=2247485099&idx=1&sn=6f6e9f36e651d78834685706f7d9b143&chksm=9f6965dfa81eecc9249eec9cc8e7642787003653a7f872f7eeab15b903afeb2a9fb76ff8fc7f#rd) 最后得到的 AST 对象放旁边辅助阅读，因为生成渲染函数的过程就是在处理该对象上众多的属性的过程。

```javascript
export function genElement(el: ASTElement, state: CodegenState): string {
  if (el.parent) {
    el.pre = el.pre || el.parent.pre
  }

  if (el.staticRoot && !el.staticProcessed) {
    /**
     * 处理静态根节点，生成节点的渲染函数
     *   1、将当前静态节点的渲染函数放到 staticRenderFns 数组中
     *   2、返回一个可执行函数 _m(idx, true or '') 
     */
    return genStatic(el, state)
  } else if (el.once && !el.onceProcessed) {
    /**
     * 处理带有 v-once 指令的节点，结果会有三种：
     *   1、当前节点存在 v-if 指令，得到一个三元表达式，condition ? render1 : render2
     *   2、当前节点是一个包含在 v-for 指令内部的静态节点，得到 `_o(_c(tag, data, children), number, key)`
     *   3、当前节点就是一个单纯的 v-once 节点，得到 `_m(idx, true of '')`
     */
    return genOnce(el, state)
  } else if (el.for && !el.forProcessed) {
    /**
     * 处理节点上的 v-for 指令  
     * 得到 `_l(exp, function(alias, iterator1, iterator2){return _c(tag, data, children)})`
     */
    return genFor(el, state)
  } else if (el.if && !el.ifProcessed) {
    /**
     * 处理带有 v-if 指令的节点，最终得到一个三元表达式：condition ? render1 : render2
     */
    return genIf(el, state)
  } else if (el.tag === 'template' && !el.slotTarget && !state.pre) {
    /**
     * 当前节点不是 template 标签也不是插槽和带有 v-pre 指令的节点时走这里
     * 生成所有子节点的渲染函数，返回一个数组，格式如：
     * [_c(tag, data, children, normalizationType), ...] 
     */
    return genChildren(el, state) || 'void 0'
  } else if (el.tag === 'slot') {
    /**
     * 生成插槽的渲染函数，得到
     * _t(slotName, children, attrs, bind)
     */
    return genSlot(el, state)
  } else {
    // component or element
    // 处理动态组件和普通元素（自定义组件、原生标签）
    let code
    if (el.component) {
      /**
       * 处理动态组件，生成动态组件的渲染函数
       * 得到 `_c(compName, data, children)`
       */
      code = genComponent(el.component, el, state)
    } else {
      // 自定义组件和原生标签走这里
      let data
      if (!el.plain || (el.pre && state.maybeComponent(el))) {
        // 非普通元素或者带有 v-pre 指令的组件走这里，处理节点的所有属性，返回一个 JSON 字符串，
        // 比如 '{ key: xx, ref: xx, ... }'
        data = genData(el, state)
      }

      // 处理子节点，得到所有子节点字符串格式的代码组成的数组，格式：
      // `['_c(tag, data, children)', ...],normalizationType`，
      // 最后的 normalization 表示节点的规范化类型，不是重点，不需要关注
      const children = el.inlineTemplate ? null : genChildren(el, state, true)
      // 得到最终的字符串格式的代码，格式：
      // '_c(tag, data, children, normalizationType)'
      code = `_c('${el.tag}'${data ? `,${data}` : '' // data
        }${children ? `,${children}` : '' // children
        })`
    }
    // 如果提供了 transformCode 方法， 
    // 则最终的 code 会经过各个模块（module）的该方法处理，
    // 不过框架没提供这个方法，不过即使处理了，最终的格式也是 _c(tag, data, children)
    // module transforms
    for (let i = 0; i < state.transforms.length; i++) {
      code = state.transforms[i](el, code)
    }
    return code
  }
}

```

### genChildren

> /src/compiler/codegen/index.js

```javascript
/**
 * 生成所有子节点的渲染函数，返回一个数组，格式如：
 * [_c(tag, data, children, normalizationType), ...] 
 */
export function genChildren(
  el: ASTElement,
  state: CodegenState,
  checkSkip?: boolean,
  altGenElement?: Function,
  altGenNode?: Function
): string | void {
  // 所有子节点
  const children = el.children
  if (children.length) {
    // 第一个子节点
    const el: any = children[0]
    // optimize single v-for
    if (children.length === 1 &&
      el.for &&
      el.tag !== 'template' &&
      el.tag !== 'slot'
    ) {
      // 优化，只有一个子节点 && 子节点的上有 v-for 指令 && 子节点的标签不为 template 或者 slot
      // 优化的方式是直接调用 genElement 生成该节点的渲染函数，不需要走下面的循环然后调用 genCode 最后得到渲染函数
      const normalizationType = checkSkip
        ? state.maybeComponent(el) ? `,1` : `,0`
        : ``
      return `${(altGenElement || genElement)(el, state)}${normalizationType}`
    }
    // 获取节点规范化类型，返回一个 number 0、1、2，不是重点， 不重要
    const normalizationType = checkSkip
      ? getNormalizationType(children, state.maybeComponent)
      : 0
    // 函数，生成代码的一个函数
    const gen = altGenNode || genNode
    // 返回一个数组，数组的每个元素都是一个子节点的渲染函数，
    // 格式：['_c(tag, data, children, normalizationType)', ...]
    return `[${children.map(c => gen(c, state)).join(',')}]${normalizationType ? `,${normalizationType}` : ''
      }`
  }
}

```

#### genNode

> /src/compiler/codegen/index.js

```javascript
function genNode(node: ASTNode, state: CodegenState): string {
  if (node.type === 1) {
    return genElement(node, state)
  } else if (node.type === 3 && node.isComment) {
    return genComment(node)
  } else {
    return genText(node)
  }
}

```

#### genText

> /src/compiler/codegen/index.js

```javascript
export function genText(text: ASTText | ASTExpression): string {
  return `_v(${text.type === 2
    ? text.expression // no need for () because already wrapped in _s()
    : transformSpecialNewlines(JSON.stringify(text.text))
    })`
}

```

#### genComment

> /src/compiler/codegen/index.js

```javascript
export function genComment(comment: ASTText): string {
  return `_e(${JSON.stringify(comment.text)})`
}

```

### genData

> /src/compiler/codegen/index.js

```javascript
/**
 * 处理节点上的众多属性，最后生成这些属性组成的 JSON 字符串，比如 data = { key: xx, ref: xx, ... } 
 */
export function genData(el: ASTElement, state: CodegenState): string {
  // 节点的属性组成的 JSON 字符串
  let data = '{'

  // 首先先处理指令，因为指令可能在生成其它属性之前改变这些属性
  // 执行指令编译方法，比如 web 平台的 v-text、v-html、v-model，然后在 el 对象上添加相应的属性，
  // 比如 v-text： el.textContent = _s(value, dir)
  //     v-html：el.innerHTML = _s(value, dir)
  // 当指令在运行时还有任务时，比如 v-model，则返回 directives: [{ name, rawName, value, arg, modifiers }, ...}] 
  // directives first.
  // directives may mutate the el's other properties before they are generated.
  const dirs = genDirectives(el, state)
  if (dirs) data += dirs + ','

  // key，data = { key: xx }
  if (el.key) {
    data += `key:${el.key},`
  }
  // ref，data = { ref: xx }
  if (el.ref) {
    data += `ref:${el.ref},`
  }
  // 带有 ref 属性的节点在带有 v-for 指令的节点的内部， data = { refInFor: true }
  if (el.refInFor) {
    data += `refInFor:true,`
  }
  // pre，v-pre 指令，data = { pre: true }
  if (el.pre) {
    data += `pre:true,`
  }
  // 动态组件，data = { tag: 'component' }
  // record original tag name for components using "is" attribute
  if (el.component) {
    data += `tag:"${el.tag}",`
  }
  // 为节点执行模块(class、style)的 genData 方法，
  // 得到 data = { staticClass: xx, class: xx, staticStyle: xx, style: xx }
  // module data generation functions
  for (let i = 0; i < state.dataGenFns.length; i++) {
    data += state.dataGenFns[i](el)
  }
  // 其它属性，得到 data = { attrs: 静态属性字符串 } 或者 
  // data = { attrs: '_d(静态属性字符串, 动态属性字符串)' }
  // attributes
  if (el.attrs) {
    data += `attrs:${genProps(el.attrs)},`
  }
  // DOM props，结果同 el.attrs
  if (el.props) {
    data += `domProps:${genProps(el.props)},`
  }
  // 自定义事件，data = { `on${eventName}:handleCode` } 或者 { `on_d(${eventName}:handleCode`, `${eventName},handleCode`) }
  // event handlers
  if (el.events) {
    data += `${genHandlers(el.events, false)},`
  }
  // 带 .native 修饰符的事件，
  // data = { `nativeOn${eventName}:handleCode` } 或者 { `nativeOn_d(${eventName}:handleCode`, `${eventName},handleCode`) }
  if (el.nativeEvents) {
    data += `${genHandlers(el.nativeEvents, true)},`
  }
  // 非作用域插槽，得到 data = { slot: slotName }
  // slot target
  // only for non-scoped slots
  if (el.slotTarget && !el.slotScope) {
    data += `slot:${el.slotTarget},`
  }
  // scoped slots，作用域插槽，data = { scopedSlots: '_u(xxx)' }
  if (el.scopedSlots) {
    data += `${genScopedSlots(el, el.scopedSlots, state)},`
  }
  // 处理 v-model 属性，得到
  // data = { model: { value, callback, expression } }
  // component v-model
  if (el.model) {
    data += `model:{value:${el.model.value
      },callback:${el.model.callback
      },expression:${el.model.expression
      }},`
  }
  // inline-template，处理内联模版，得到
  // data = { inlineTemplate: { render: function() { render 函数 }, staticRenderFns: [ function() {}, ... ] } }
  if (el.inlineTemplate) {
    const inlineTemplate = genInlineTemplate(el, state)
    if (inlineTemplate) {
      data += `${inlineTemplate},`
    }
  }
  // 删掉 JSON 字符串最后的 逗号，然后加上闭合括号 }
  data = data.replace(/,$/, '') + '}'
  // v-bind dynamic argument wrap
  // v-bind with dynamic arguments must be applied using the same v-bind object
  // merge helper so that class/style/mustUseProp attrs are handled correctly.
  if (el.dynamicAttrs) {
    // 存在动态属性，data = `_b(data, tag, 静态属性字符串或者_d(静态属性字符串, 动态属性字符串))`
    data = `_b(${data},"${el.tag}",${genProps(el.dynamicAttrs)})`
  }
  // v-bind data wrap
  if (el.wrapData) {
    data = el.wrapData(data)
  }
  // v-on data wrap
  if (el.wrapListeners) {
    data = el.wrapListeners(data)
  }
  return data
}

```

#### genDirectives

> /src/compiler/codegen/index.js

> **阅读建议**：这部分内容也可以放到其它方法后面去读，比如你想深究 v-model 的实现原理

```javascript
/**
 * 运行指令的编译方法，如果指令存在运行时任务，则返回 directives: [{ name, rawName, value, arg, modifiers }, ...}] 
 */
function genDirectives(el: ASTElement, state: CodegenState): string | void {
  // 获取指令数组
  const dirs = el.directives
  // 没有指令则直接结束
  if (!dirs) return
  // 指令的处理结果
  let res = 'directives:['
  // 标记，用于标记指令是否需要在运行时完成的任务，比如 v-model 的 input 事件
  let hasRuntime = false
  let i, l, dir, needRuntime
  // 遍历指令数组
  for (i = 0, l = dirs.length; i < l; i++) {
    dir = dirs[i]
    needRuntime = true
    // 获取节点当前指令的处理方法，比如 web 平台的 v-html、v-text、v-model
    const gen: DirectiveFunction = state.directives[dir.name]
    if (gen) {
      // 执行指令的编译方法，如果指令还需要运行时完成一部分任务，则返回 true，比如 v-model
      // compile-time directive that manipulates AST.
      // returns true if it also needs a runtime counterpart.
      needRuntime = !!gen(el, dir, state.warn)
    }
    if (needRuntime) {
      // 表示该指令在运行时还有任务
      hasRuntime = true
      // res = directives:[{ name, rawName, value, arg, modifiers }, ...]
      res += `{name:"${dir.name}",rawName:"${dir.rawName}"${dir.value ? `,value:(${dir.value}),expression:${JSON.stringify(dir.value)}` : ''
        }${dir.arg ? `,arg:${dir.isDynamicArg ? dir.arg : `"${dir.arg}"`}` : ''
        }${dir.modifiers ? `,modifiers:${JSON.stringify(dir.modifiers)}` : ''
        }},`
    }
  }
  if (hasRuntime) {
    // 也就是说，只有指令存在运行时任务时，才会返回 res
    return res.slice(0, -1) + ']'
  }
}

```

#### genProps

> /src/compiler/codegen/index.js

```javascript
/**
 * 遍历属性数组 props，得到所有属性组成的字符串
 * 如果不存在动态属性，则返回：
 *   'attrName,attrVal,...'
 * 如果存在动态属性，则返回：
 *   '_d(静态属性字符串, 动态属性字符串)' 
 */
function genProps(props: Array<ASTAttr>): string {
  // 静态属性
  let staticProps = ``
  // 动态属性
  let dynamicProps = ``
  // 遍历属性数组
  for (let i = 0; i < props.length; i++) {
    // 属性
    const prop = props[i]
    // 属性值
    const value = __WEEX__
      ? generateValue(prop.value)
      : transformSpecialNewlines(prop.value)
    if (prop.dynamic) {
      // 动态属性，`dAttrName,dAttrVal,...`
      dynamicProps += `${prop.name},${value},`
    } else {
      // 静态属性，'attrName,attrVal,...'
      staticProps += `"${prop.name}":${value},`
    }
  }
  // 去掉静态属性最后的逗号
  staticProps = `{${staticProps.slice(0, -1)}}`
  if (dynamicProps) {
    // 如果存在动态属性则返回：
    // _d(静态属性字符串，动态属性字符串)
    return `_d(${staticProps},[${dynamicProps.slice(0, -1)}])`
  } else {
    // 说明属性数组中不存在动态属性，直接返回静态属性字符串
    return staticProps
  }
}

```

#### genHandlers

> /src/compiler/codegen/events.js

```javascript
/**
 * 生成自定义事件的代码
 * 动态：'nativeOn|on_d(staticHandlers, [dynamicHandlers])'
 * 静态：`nativeOn|on${staticHandlers}`
 */
 export function genHandlers (
  events: ASTElementHandlers,
  isNative: boolean
): string {
  // 原生：nativeOn，否则为 on
  const prefix = isNative ? 'nativeOn:' : 'on:'
  // 静态
  let staticHandlers = ``
  // 动态
  let dynamicHandlers = ``
  // 遍历 events 数组
  // events = [{ name: { value: 回调函数名, ... } }]
  for (const name in events) {
    // 获取指定事件的回调函数名，即 this.methodName 或者 [this.methodName1, ...]
    const handlerCode = genHandler(events[name])
    if (events[name] && events[name].dynamic) {
      // 动态，dynamicHandles = `eventName,handleCode,...,`
      dynamicHandlers += `${name},${handlerCode},`
    } else {
      // 静态，staticHandles = `"eventName":handleCode,`
      staticHandlers += `"${name}":${handlerCode},`
    }
  }
  // 去掉末尾的逗号
  staticHandlers = `{${staticHandlers.slice(0, -1)}}`
  if (dynamicHandlers) {
    // 动态，on_d(statickHandles, [dynamicHandlers])
    return prefix + `_d(${staticHandlers},[${dynamicHandlers.slice(0, -1)}])`
  } else {
    // 静态，`on${staticHandlers}`
    return prefix + staticHandlers
  }
}

```

### genStatic

> /src/compiler/codegen/index.js

```javascript
/**
 * 生成静态节点的渲染函数
 *   1、将当前静态节点的渲染函数放到 staticRenderFns 数组中
 *   2、返回一个可执行函数 _m(idx, true or '') 
 */
// hoist static sub-trees out
function genStatic(el: ASTElement, state: CodegenState): string {
  // 标记当前静态节点已经被处理过了
  el.staticProcessed = true
  // Some elements (templates) need to behave differently inside of a v-pre
  // node.  All pre nodes are static roots, so we can use this as a location to
  // wrap a state change and reset it upon exiting the pre node.
  const originalPreState = state.pre
  if (el.pre) {
    state.pre = el.pre
  }
  // 将静态根节点的渲染函数 push 到 staticRenderFns 数组中，比如：
  // [`with(this){return _c(tag, data, children)}`]
  state.staticRenderFns.push(`with(this){return ${genElement(el, state)}}`)
  state.pre = originalPreState
  // 返回一个可执行函数：_m(idx, true or '')
  // idx = 当前静态节点的渲染函数在 staticRenderFns 数组中下标
  return `_m(${state.staticRenderFns.length - 1
    }${el.staticInFor ? ',true' : ''
    })`
}

```

### genOnce

> /src/compiler/codegen/index.js

```javascript
/**
 * 处理带有 v-once 指令的节点，结果会有三种：
 *   1、当前节点存在 v-if 指令，得到一个三元表达式，condition ? render1 : render2
 *   2、当前节点是一个包含在 v-for 指令内部的静态节点，得到 `_o(_c(tag, data, children), number, key)`
 *   3、当前节点就是一个单纯的 v-once 节点，得到 `_m(idx, true of '')`
 */
function genOnce(el: ASTElement, state: CodegenState): string {
  // 标记当前节点的 v-once 指令已经被处理过了
  el.onceProcessed = true
  if (el.if && !el.ifProcessed) {
    // 如果含有 v-if 指令 && if 指令没有被处理过，则走这里
    // 处理带有 v-if 指令的节点，最终得到一个三元表达式，condition ? render1 : render2 
    return genIf(el, state)
  } else if (el.staticInFor) {
    // 说明当前节点是被包裹在还有 v-for 指令节点内部的静态节点
    // 获取 v-for 指令的 key
    let key = ''
    let parent = el.parent
    while (parent) {
      if (parent.for) {
        key = parent.key
        break
      }
      parent = parent.parent
    }
    // key 不存在则给出提示，v-once 节点只能用于带有 key 的 v-for 节点内部
    if (!key) {
      process.env.NODE_ENV !== 'production' && state.warn(
        `v-once can only be used inside v-for that is keyed. `,
        el.rawAttrsMap['v-once']
      )
      return genElement(el, state)
    }
    // 生成 `_o(_c(tag, data, children), number, key)`
    return `_o(${genElement(el, state)},${state.onceId++},${key})`
  } else {
    // 上面几种情况都不符合，说明就是一个简单的静态节点，和处理静态根节点时的操作一样,
    // 得到 _m(idx, true or '')
    return genStatic(el, state)
  }
}

```

### genFor

> /src/compiler/codegen/index.js

```javascript
/**
 * 处理节点上的 v-for 指令  
 * 得到 `_l(exp, function(alias, iterator1, iterator2){return _c(tag, data, children)})`
 */
export function genFor(
  el: any,
  state: CodegenState,
  altGen?: Function,
  altHelper?: string
): string {
  // v-for 的迭代器，比如 一个数组
  const exp = el.for
  // 迭代时的别名
  const alias = el.alias
  // iterator 为 v-for = "(item ,idx) in obj" 时会有，比如 iterator1 = idx
  const iterator1 = el.iterator1 ? `,${el.iterator1}` : ''
  const iterator2 = el.iterator2 ? `,${el.iterator2}` : ''

  // 提示，v-for 指令在组件上时必须使用 key
  if (process.env.NODE_ENV !== 'production' &&
    state.maybeComponent(el) &&
    el.tag !== 'slot' &&
    el.tag !== 'template' &&
    !el.key
  ) {
    state.warn(
      `<${el.tag} v-for="${alias} in ${exp}">: component lists rendered with ` +
      `v-for should have explicit keys. ` +
      `See https://vuejs.org/guide/list.html#key for more info.`,
      el.rawAttrsMap['v-for'],
      true /* tip */
    )
  }

  // 标记当前节点上的 v-for 指令已经被处理过了
  el.forProcessed = true // avoid r
  // 得到 `_l(exp, function(alias, iterator1, iterator2){return _c(tag, data, children)})`
  return `${altHelper || '_l'}((${exp}),` +
    `function(${alias}${iterator1}${iterator2}){` +
    `return ${(altGen || genElement)(el, state)}` +
    '})'
}

```

### genIf

> /src/compiler/codegen/index.js

```javascript
/**
 * 处理带有 v-if 指令的节点，最终得到一个三元表达式，condition ? render1 : render2 
 */
export function genIf(
  el: any,
  state: CodegenState,
  altGen?: Function,
  altEmpty?: string
): string {
  // 标记当前节点的 v-if 指令已经被处理过了，避免无效的递归
  el.ifProcessed = true // avoid recursion
  // 得到三元表达式，condition ? render1 : render2
  return genIfConditions(el.ifConditions.slice(), state, altGen, altEmpty)
}

function genIfConditions(
  conditions: ASTIfConditions,
  state: CodegenState,
  altGen?: Function,
  altEmpty?: string
): string {
  // 长度若为空，则直接返回一个空节点渲染函数
  if (!conditions.length) {
    return altEmpty || '_e()'
  }

  // 从 conditions 数组中拿出第一个条件对象 { exp, block }
  const condition = conditions.shift()
  // 返回结果是一个三元表达式字符串，condition ? 渲染函数1 : 渲染函数2
  if (condition.exp) {
    // 如果 condition.exp 条件成立，则得到一个三元表达式，
    // 如果条件不成立，则通过递归的方式找 conditions 数组中下一个元素，
    // 直到找到条件成立的元素，然后返回一个三元表达式
    return `(${condition.exp})?${genTernaryExp(condition.block)
      }:${genIfConditions(conditions, state, altGen, altEmpty)
      }`
  } else {
    return `${genTernaryExp(condition.block)}`
  }

  // v-if with v-once should generate code like (a)?_m(0):_m(1)
  function genTernaryExp(el) {
    return altGen
      ? altGen(el, state)
      : el.once
        ? genOnce(el, state)
        : genElement(el, state)
  }
}

```

### genSlot

> /src/compiler/codegen/index.js

```javascript
/**
 * 生成插槽的渲染函数，得到
 * _t(slotName, children, attrs, bind)
 */
function genSlot(el: ASTElement, state: CodegenState): string {
  // 插槽名称
  const slotName = el.slotName || '"default"'
  // 生成所有的子节点
  const children = genChildren(el, state)
  // 结果字符串，_t(slotName, children, attrs, bind)
  let res = `_t(${slotName}${children ? `,${children}` : ''}`
  const attrs = el.attrs || el.dynamicAttrs
    ? genProps((el.attrs || []).concat(el.dynamicAttrs || []).map(attr => ({
      // slot props are camelized
      name: camelize(attr.name),
      value: attr.value,
      dynamic: attr.dynamic
    })))
    : null
  const bind = el.attrsMap['v-bind']
  if ((attrs || bind) && !children) {
    res += `,null`
  }
  if (attrs) {
    res += `,${attrs}`
  }
  if (bind) {
    res += `${attrs ? '' : ',null'},${bind}`
  }
  return res + ')'
}

```

### genComponent

> /src/compiler/codegen/index.js

```javascript
// componentName is el.component, take it as argument to shun flow's pessimistic refinement
/**
 * 生成动态组件的渲染函数
 * 返回 `_c(compName, data, children)`
 */
function genComponent(
  componentName: string,
  el: ASTElement,
  state: CodegenState
): string {
  // 所有的子节点
  const children = el.inlineTemplate ? null : genChildren(el, state, true)
  // 返回 `_c(compName, data, children)`
  // compName 是 is 属性的值
  return `_c(${componentName},${genData(el, state)}${children ? `,${children}` : ''
    })`
}

```

## 总结

* **面试官 问**：简单说一下 Vue 的编译器都做了什么？

  **答**：
  
  Vue 的编译器做了三件事情：
  
  * 将组件的 html 模版解析成 AST 对象
    
  * 优化，遍历 AST，为每个节点做静态标记，标记其是否为静态节点，然后进一步标记出静态根节点，这样在后续更新的过程中就可以跳过这些静态节点了；标记静态根用于生成渲染函数阶段，生成静态根节点的渲染函数
    
  * 从 AST 生成运行渲染函数，即大家说的 render，其实还有一个，就是 staticRenderFns 数组，里面存放了所有的静态节点的渲染函数
  
<hr />

* **面试官**：详细说一下渲染函数的生成过程

  **答**：

  大家一说到渲染函数，基本上说的就是 render 函数，其实编译器生成的渲染有两类：
  
  * 第一类就是一个 render 函数，负责生成动态节点的 vnode
  
  * 第二类是放在一个叫 staticRenderFns 数组中的静态渲染函数，这些函数负责生成静态节点的 vnode
  
  渲染函数生成的过程，其实就是在遍历 AST 节点，通过递归的方式，处理每个节点，最后生成形如：`_c(tag, attr, children, normalizationType)` 的结果。tag 是标签名，attr 是属性对象，children 是子节点组成的数组，其中每个元素的格式都是 `_c(tag, attr, children, normalizationTYpe)` 的形式，normalization 表示节点的规范化类型，是一个数字 0、1、2，不重要。
  
  在处理 AST 节点过程中需要大家重点关注也是面试中常见的问题有：
  
  * 静态节点是怎么处理的
  
    静态节点的处理分为两步：
    
    * 将生成静态节点 vnode 函数放到 staticRenderFns 数组中
    
    * 返回一个 _m(idx) 的可执行函数，意思是执行 staticRenderFns 数组中下标为 idx 的函数，生成静态节点的 vnode
  
  * v-once、v-if、v-for、组件 等都是怎么处理的
  
    * 单纯的 v-once 节点处理方式和静态节点一致
    
    * v-if 节点的处理结果是一个三元表达式
    
    * v-for 节点的处理结果是可执行的 _l 函数，该函数负责生成 v-for 节点的 vnode
  
    * 组件的处理结果和普通元素一样，得到的是形如 `_c(compName)` 的可执行代码，生成组件的 vnode
    
<hr />

到这里，Vue 编译器 的源码解读就结束了。相信大家在阅读的过程中不免会产生云里雾里的感觉。这个没什么，编译器这块儿确实是比较复杂，可以说是整个框架最难理解也是代码量最大的一部分了。一定要静下心来多读几遍，遇到无法理解的地方，一定要勤动手，通过示例代码加断点调试的方式帮助自己理解。

当你读完几遍以后，这时候情况可能就会好一些，但是有些地方可能还会有些晕，这没事，正常。毕竟这是一个框架的编译器，要处理的东西太多太多了，你只需要理解其核心思想（模版解析、静态标记、代码生成）就可以了。后面会有 **手写 Vue 系列**，编译器这部分会有一个简版的实现，帮助加深对这部分知识的理解。

编译器读完以后，会发现有个不明白的地方：编译器最后生成的代码都是经过 `with` 包裹的，比如:

```html
<div id="app">
  <div v-for="item in arr" :key="item">{{ item }}</div>
</div>
```
经过编译后生成：

```javascript
with (this) {
  return _c(
    'div',
    {
      attrs:
      {
        "id": "app"
      }
    },
    _l(
      (arr),
      function (item) {
        return _c(
          'div',
          {
            key: item
          },
          [_v(_s(item))]
        )
      }
    ),
    0
  )
}

```

都知道，`with` 语句可以扩展作用域链，所以生成的代码中的 `_c、_l、_v、_s` 都是 this 上一些方法，也就是说在运行时执行这些方法可以生成各个节点的 vnode。

所以联系前面的知识，响应式数据更新的整个执行过程就是：

* 响应式拦截到数据的更新

* dep 通知 watcher 进行异步更新

* watcher 更新时执行组件更新函数 updateComponent

* 首先执行 vm._render 生成组件的 vnode，这时就会执行编译器生成的函数

* **问题**：

  * 渲染函数中的 `_c、_l、、_v、_s` 等方法是什么？
  
  * 它们是如何生成 vnode 的？
  

下一篇文章 **Vue 源码解读（11）—— render helper** 将会带来这部分知识的详细解读，也是面试经常被问题的：比如：`v-for` 的原理是什么？

## 链接

* [配套视频，微信公众号回复](https://gitee.com/liyongning/typora-image-bed/raw/master/202202171742614.jpg)："精通 Vue 技术栈源码原理视频版" 获取
* [精通 Vue 技术栈源码原理 专栏](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzA3NTk4NjQ1OQ==&action=getalbum&album_id=2273541436891693065#wechat_redirect)
* [github 仓库 liyongning/Vue](https://github.com/liyongning/Vue) 欢迎 Star



感谢各位的：**关注**、**点赞**、**收藏**和**评论**，我们下期见。

***

**当学习成为了习惯，知识也就变成了常识。** 感谢各位的 **关注**、 **点赞**、**收藏**和**评论**。

新视频和文章会第一时间在微信公众号发送，欢迎关注：[李永宁lyn](https://gitee.com/liyongning/typora-image-bed/raw/master/202202171742614.jpg)

文章已收录到 [github 仓库 liyongning/blog](https://github.com/liyongning/blog)，欢迎 Watch 和 Star。