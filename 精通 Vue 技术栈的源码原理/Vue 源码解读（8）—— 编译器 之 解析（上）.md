# Vue 源码解读（8）—— 编译器 之 解析（上）

**当学习成为了习惯，知识也就变成了常识。** 感谢各位的 **关注**、**点赞**、**收藏**和**评论**。

新视频和文章会第一时间在微信公众号发送，欢迎关注：[李永宁lyn](https://gitee.com/liyongning/typora-image-bed/raw/master/202202171742614.jpg)

文章已收录到 [github 仓库 liyongning/blog](https://github.com/liyongning/blog)，欢迎 Watch 和 Star。

## 封面

![](https://gitee.com/liyongning/typora-image-bed/raw/master/202202271623961.png)

## 特殊说明

由于文章篇幅限制，所以将 **Vue 源码解读（8）—— 编译器 之 解析** 拆成了上下两篇，所以在阅读本篇文章时请同时打开 [Vue 源码解读（8）—— 编译器 之 解析（下）](https://mp.weixin.qq.com/s?__biz=MzA3NTk4NjQ1OQ==&mid=2247485099&idx=1&sn=6f6e9f36e651d78834685706f7d9b143&chksm=9f6965dfa81eecc9249eec9cc8e7642787003653a7f872f7eeab15b903afeb2a9fb76ff8fc7f#rd)一起阅读。

## 前言

[Vue 源码解读（4）—— 异步更新](https://mp.weixin.qq.com/s?__biz=MzA3NTk4NjQ1OQ==&mid=2247484856&idx=1&sn=ecfceefe3b843ae4487953876c3be6e0&chksm=9f6966cca81eefdac9e45c56bdcab39c510a47be15ed97595ded029a72e0fc839f1f67a694c2#rd) 最后说到刷新 watcher 队列，执行每个 `watcher.run` 方法，由 `watcher.run` 调用 `watcher.get`，从而执行 `watcher.getter` 方法，进入实际的更新阶段。这个流程如果不熟悉，建议大家再去读一下这篇文章。

当更新一个渲染 watcher 时，执行的是 `updateComponent` 方法：

```javascript
// /src/core/instance/lifecycle.js
const updateComponent = () => {
  // 执行 vm._render() 函数，得到 虚拟 DOM，并将 vnode 传递给 _update 方法，接下来就该到 patch 阶段了
  vm._update(vm._render(), hydrating)
}

```

可以看到每次更新前都需要先执行一下 `vm._render()` 方法，vm._render 就是大家经常听到的 render 函数，由两种方式得到：

* 用户自己提供，在编写组件时，用 render 选项代替模版

* 由编译器编译组件模版生成 render 选项

今天我们就来深入研究编译器，看看它是怎么将我们平时编写的类 html 模版编译成 render 函数的。

编译器的核心由三部分组成：

* **解析**，将类 html 模版转换为 AST 对象

* **优化**，也叫静态标记，遍历 AST 对象，标记每个节点是否为静态节点，以及标记出静态根节点

* **生成渲染函数**，将 AST 对象生成渲染函数

由于编译器这块儿的代码量太大，所以，将这部分知识拆成三部分来讲，第一部分就是：解析。

## 目标

深入理解 Vue 编译器的解析过程，理解如何将类 html 模版字符串转换成 AST 对象。

## 源码解读

接下来我们去源码中找答案。

> **阅读建议**
>
> 由于解析过程代码量巨大，所以建议大家抓住主线：“解析类 HTML 字符串模版，生成 AST 对象”，而这个 AST 对象就是我们最终要得到的结果，所以大家在阅读的过程中，要动手记录这个 AST 对象，这样有助于理解，也让你不那么容易迷失。

> 也可以先阅读 **下篇** 的 **帮助** 部分，有个提前的准备和心理预期。

### 入口 - $mount

编译器的入口位置在 `/src/platforms/web/entry-runtime-with-compiler.js`，有两种方式找到这个入口

* 断点调试，[Vue 源码解读（2）—— Vue 初始化过程](https://mp.weixin.qq.com/s?__biz=MzA3NTk4NjQ1OQ==&mid=2247484817&idx=1&sn=d6357aed6fd2f7e7cc8e47f8ad79ea96&chksm=9f6966e5a81eeff3eb2ac169a41fba0960e9cc8863c193e8608501a873bead743021d37d71f7#rd) 中讲到，初始化的最后一步是执行 `$mount` 进行挂载，在全量的 Vue 包中这一步就会进入编译阶段。

* 通过 rollup 的配置文件一步步的去找

> /src/platforms/web/entry-runtime-with-compiler.js

```javascript
/**
 * 编译器的入口
 * 运行时的 Vue.js 包就没有这部分的代码，通过 打包器 结合 vue-loader + vue-compiler-utils 进行预编译，将模版编译成 render 函数
 * 
 * 就做了一件事情，得到组件的渲染函数，将其设置到 this.$options 上
 */
const mount = Vue.prototype.$mount
Vue.prototype.$mount = function (
  el?: string | Element,
  hydrating?: boolean
): Component {
  // 挂载点
  el = el && query(el)

  // 挂载点不能是 body 或者 html
  /* istanbul ignore if */
  if (el === document.body || el === document.documentElement) {
    process.env.NODE_ENV !== 'production' && warn(
      `Do not mount Vue to <html> or <body> - mount to normal elements instead.`
    )
    return this
  }

  // 配置项
  const options = this.$options
  // resolve template/el and convert to render function
  /**
   * 如果用户提供了 render 配置项，则直接跳过编译阶段，否则进入编译阶段
   *   解析 template 和 el，并转换为 render 函数
   *   优先级：render > template > el
   */
  if (!options.render) {
    let template = options.template
    if (template) {
      // 处理 template 选项
      if (typeof template === 'string') {
        if (template.charAt(0) === '#') {
          // { template: '#app' }，template 是一个 id 选择器，则获取该元素的 innerHtml 作为模版
          template = idToTemplate(template)
          /* istanbul ignore if */
          if (process.env.NODE_ENV !== 'production' && !template) {
            warn(
              `Template element not found or is empty: ${options.template}`,
              this
            )
          }
        }
      } else if (template.nodeType) {
        // template 是一个正常的元素，获取其 innerHtml 作为模版
        template = template.innerHTML
      } else {
        if (process.env.NODE_ENV !== 'production') {
          warn('invalid template option:' + template, this)
        }
        return this
      }
    } else if (el) {
      // 设置了 el 选项，获取 el 选择器的 outerHtml 作为模版
      template = getOuterHTML(el)
    }
    if (template) {
      // 模版就绪，进入编译阶段
      /* istanbul ignore if */
      if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
        mark('compile')
      }

      // 编译模版，得到 动态渲染函数和静态渲染函数
      const { render, staticRenderFns } = compileToFunctions(template, {
        // 在非生产环境下，编译时记录标签属性在模版字符串中开始和结束的位置索引
        outputSourceRange: process.env.NODE_ENV !== 'production',
        shouldDecodeNewlines,
        shouldDecodeNewlinesForHref,
        // 界定符，默认 {{}}
        delimiters: options.delimiters,
        // 是否保留注释
        comments: options.comments
      }, this)
      // 将两个渲染函数放到 this.$options 上
      options.render = render
      options.staticRenderFns = staticRenderFns

      /* istanbul ignore if */
      if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
        mark('compile end')
        measure(`vue ${this._name} compile`, 'compile', 'compile end')
      }
    }
  }
  // 执行挂载
  return mount.call(this, el, hydrating)
}

```

### compileToFunctions

> /src/compiler/to-function.js

```javascript
/**
 * 1、执行编译函数，得到编译结果 -> compiled
 * 2、处理编译期间产生的 error 和 tip，分别输出到控制台
 * 3、将编译得到的字符串代码通过 new Function(codeStr) 转换成可执行的函数
 * 4、缓存编译结果
 * @param { string } template 字符串模版
 * @param { CompilerOptions } options 编译选项
 * @param { Component } vm 组件实例
 * @return { render, staticRenderFns }
 */
return function compileToFunctions(
  template: string,
  options?: CompilerOptions,
  vm?: Component
): CompiledFunctionResult {
  // 传递进来的编译选项
  options = extend({}, options)
  // 日志
  const warn = options.warn || baseWarn
  delete options.warn

  /* istanbul ignore if */
  if (process.env.NODE_ENV !== 'production') {
    // 检测可能的 CSP 限制
    try {
      new Function('return 1')
    } catch (e) {
      if (e.toString().match(/unsafe-eval|CSP/)) {
        // 看起来你在一个 CSP 不安全的环境中使用完整版的 Vue.js，模版编译器不能工作在这样的环境中。
        // 考虑放宽策略限制或者预编译你的 template 为 render 函数
        warn(
          'It seems you are using the standalone build of Vue.js in an ' +
          'environment with Content Security Policy that prohibits unsafe-eval. ' +
          'The template compiler cannot work in this environment. Consider ' +
          'relaxing the policy to allow unsafe-eval or pre-compiling your ' +
          'templates into render functions.'
        )
      }
    }
  }

  // 如果有缓存，则跳过编译，直接从缓存中获取上次编译的结果
  const key = options.delimiters
    ? String(options.delimiters) + template
    : template
  if (cache[key]) {
    return cache[key]
  }

  // 执行编译函数，得到编译结果
  const compiled = compile(template, options)

  // 检查编译期间产生的 error 和 tip，分别输出到控制台
  if (process.env.NODE_ENV !== 'production') {
    if (compiled.errors && compiled.errors.length) {
      if (options.outputSourceRange) {
        compiled.errors.forEach(e => {
          warn(
            `Error compiling template:\n\n${e.msg}\n\n` +
            generateCodeFrame(template, e.start, e.end),
            vm
          )
        })
      } else {
        warn(
          `Error compiling template:\n\n${template}\n\n` +
          compiled.errors.map(e => `- ${e}`).join('\n') + '\n',
          vm
        )
      }
    }
    if (compiled.tips && compiled.tips.length) {
      if (options.outputSourceRange) {
        compiled.tips.forEach(e => tip(e.msg, vm))
      } else {
        compiled.tips.forEach(msg => tip(msg, vm))
      }
    }
  }

  // 转换编译得到的字符串代码为函数，通过 new Function(code) 实现
  // turn code into functions
  const res = {}
  const fnGenErrors = []
  res.render = createFunction(compiled.render, fnGenErrors)
  res.staticRenderFns = compiled.staticRenderFns.map(code => {
    return createFunction(code, fnGenErrors)
  })

  // 处理上面代码转换过程中出现的错误，这一步一般不会报错，除非编译器本身出错了
  /* istanbul ignore if */
  if (process.env.NODE_ENV !== 'production') {
    if ((!compiled.errors || !compiled.errors.length) && fnGenErrors.length) {
      warn(
        `Failed to generate render function:\n\n` +
        fnGenErrors.map(({ err, code }) => `${err.toString()} in\n\n${code}\n`).join('\n'),
        vm
      )
    }
  }

  // 缓存编译结果
  return (cache[key] = res)
}

```

### compile

> /src/compiler/create-compiler.js

```javascript
/**
 * 编译函数，做了两件事：
 *   1、选项合并，将 options 配置项 合并到 finalOptions(baseOptions) 中，得到最终的编译配置对象
 *   2、调用核心编译器 baseCompile 得到编译结果
 *   3、将编译期间产生的 error 和 tip 挂载到编译结果上，返回编译结果
 * @param {*} template 模版
 * @param {*} options 配置项
 * @returns 
 */
function compile(
  template: string,
  options?: CompilerOptions
): CompiledResult {
  // 以平台特有的编译配置为原型创建编译选项对象
  const finalOptions = Object.create(baseOptions)
  const errors = []
  const tips = []

  // 日志，负责记录将 error 和 tip
  let warn = (msg, range, tip) => {
    (tip ? tips : errors).push(msg)
  }

  // 如果存在编译选项，合并 options 和 baseOptions
  if (options) {
    // 开发环境走
    if (process.env.NODE_ENV !== 'production' && options.outputSourceRange) {
      // $flow-disable-line
      const leadingSpaceLength = template.match(/^\s*/)[0].length

      // 增强 日志 方法
      warn = (msg, range, tip) => {
        const data: WarningMessage = { msg }
        if (range) {
          if (range.start != null) {
            data.start = range.start + leadingSpaceLength
          }
          if (range.end != null) {
            data.end = range.end + leadingSpaceLength
          }
        }
        (tip ? tips : errors).push(data)
      }
    }

    /**
     * 将 options 中的配置项合并到 finalOptions
     */

    // 合并自定义 module
    if (options.modules) {
      finalOptions.modules =
        (baseOptions.modules || []).concat(options.modules)
    }
    // 合并自定义指令
    if (options.directives) {
      finalOptions.directives = extend(
        Object.create(baseOptions.directives || null),
        options.directives
      )
    }
    // 拷贝其它配置项
    for (const key in options) {
      if (key !== 'modules' && key !== 'directives') {
        finalOptions[key] = options[key]
      }
    }
  }

  // 日志
  finalOptions.warn = warn

  // 到这里为止终于到重点了，调用核心编译函数，传递模版字符串和最终的编译选项，得到编译结果
  // 前面做的所有事情都是为了构建平台最终的编译选项
  const compiled = baseCompile(template.trim(), finalOptions)
  if (process.env.NODE_ENV !== 'production') {
    detectErrors(compiled.ast, warn)
  }
  // 将编译期间产生的错误和提示挂载到编译结果上
  compiled.errors = errors
  compiled.tips = tips
  return compiled
}

```

#### baseOptions

> /src/platforms/web/compiler/options.js

```javascript
export const baseOptions: CompilerOptions = {
  expectHTML: true,
  // 处理 class、style、v-model
  modules,
  // 处理指令
  // 是否是 pre 标签
  isPreTag,
  // 是否是自闭合标签
  isUnaryTag,
  // 规定了一些应该使用 props 进行绑定的属性
  mustUseProp,
  // 可以只写开始标签的标签，结束标签浏览器会自动补全
  canBeLeftOpenTag,
  // 是否是保留标签（html + svg）
  isReservedTag,
  // 获取标签的命名空间
  getTagNamespace,
  staticKeys: genStaticKeys(modules)
}

```

### baseCompile

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
function baseCompile (
  template: string,
  options: CompilerOptions
): CompiledResult {
  // 将模版解析为 AST，每个节点的 ast 对象上都设置了元素的所有信息，比如，标签信息、属性信息、插槽信息、父节点、子节点等。
  // 具体有那些属性，查看 start 和 end 这两个处理开始和结束标签的方法
  const ast = parse(template.trim(), options)
  // 优化，遍历 AST，为每个节点做静态标记
  // 标记每个节点是否为静态节点，然后进一步标记出静态根节点
  // 这样在后续更新中就可以跳过这些静态节点了
  // 标记静态根，用于生成渲染函数阶段，生成静态根节点的渲染函数
  if (options.optimize !== false) {
    optimize(ast, options)
  }
  // 从 AST 生成渲染函数，生成像这样的代码，比如：code.render = "_c('div',{attrs:{"id":"app"}},_l((arr),function(item){return _c('div',{key:item},[_v(_s(item))])}),0)"
  const code = generate(ast, options)
  return {
    ast,
    render: code.render,
    staticRenderFns: code.staticRenderFns
  }
}

```

### parse

**注意**：由于这部分的代码量太大，于是将代码在结构上做了一些调整，方便大家阅读和理解。

> /src/compiler/parser/index.js

```javascript
/**
 * 
 * 将 HTML 字符串转换为 AST
 * @param {*} template HTML 模版
 * @param {*} options 平台特有的编译选项
 * @returns root
 */
export function parse(
  template: s tring,
  options: CompilerOptions
): ASTElement | void {
  // 日志
  warn = options.warn || baseWarn

  // 是否为 pre 标签
  platformIsPreTag = options.isPreTag || no
  // 必须使用 props 进行绑定的属性
  platformMustUseProp = options.mustUseProp || no
  // 获取标签的命名空间
  platformGetTagNamespace = options.getTagNamespace || no
  // 是否是保留标签（html + svg)
  const isReservedTag = options.isReservedTag || no
  // 判断一个元素是否为一个组件
  maybeComponent = (el: ASTElement) => !!el.component || !isReservedTag(el.tag)

  // 分别获取 options.modules 下的 class、model、style 三个模块中的 transformNode、preTransformNode、postTransformNode 方法
  // 负责处理元素节点上的 class、style、v-model
  transforms = pluckModuleFunction(options.modules, 'transformNode')
  preTransforms = pluckModuleFunction(options.modules, 'preTransformNode')
  postTransforms = pluckModuleFunction(options.modules, 'postTransformNode')

  // 界定符，比如: {{}}
  delimiters = options.delimiters

  const stack = []
  // 空格选项
  const preserveWhitespace = options.preserveWhitespace !== false
  const whitespaceOption = options.whitespace
  // 根节点，以 root 为根，处理后的节点都会按照层级挂载到 root 下，最后 return 的就是 root，一个 ast 语法树
  let root
  // 当前元素的父元素
  let currentParent
  let inVPre = false
  let inPre = false
  let warned = false
  
  // 解析 html 模版字符串，处理所有标签以及标签上的属性
  // 这里的 parseHTMLOptions 在后面处理过程中用到，再进一步解析
  // 提前解析的话容易让大家岔开思路
  parseHTML(template, parseHtmlOptions)
  
  // 返回生成的 ast 对象
  return root

```

### parseHTML

> /src/compiler/parser/html-parser.js

```javascript
/**
 * 通过循环遍历 html 模版字符串，依次处理其中的各个标签，以及标签上的属性
 * @param {*} html html 模版
 * @param {*} options 配置项
 */
export function parseHTML(html, options) {
  const stack = []
  const expectHTML = options.expectHTML
  // 是否是自闭合标签
  const isUnaryTag = options.isUnaryTag || no
  // 是否可以只有开始标签
  const canBeLeftOpenTag = options.canBeLeftOpenTag || no
  // 记录当前在原始 html 字符串中的开始位置
  let index = 0
  let last, lastTag
  while (html) {
    last = html
    // 确保不是在 script、style、textarea 这样的纯文本元素中
    if (!lastTag || !isPlainTextElement(lastTag)) {
      // 找第一个 < 字符
      let textEnd = html.indexOf('<')
      // textEnd === 0 说明在开头找到了
      // 分别处理可能找到的注释标签、条件注释标签、Doctype、开始标签、结束标签
      // 每处理完一种情况，就会截断（continue）循环，并且重置 html 字符串，将处理过的标签截掉，下一次循环处理剩余的 html 字符串模版
      if (textEnd === 0) {
        // 处理注释标签 <!-- xx -->
        if (comment.test(html)) {
          // 注释标签的结束索引
          const commentEnd = html.indexOf('-->')

          if (commentEnd >= 0) {
            // 是否应该保留 注释
            if (options.shouldKeepComment) {
              // 得到：注释内容、注释的开始索引、结束索引
              options.comment(html.substring(4, commentEnd), index, index + commentEnd + 3)
            }
            // 调整 html 和 index 变量
            advance(commentEnd + 3)
            continue
          }
        }

        // 处理条件注释标签：<!--[if IE]>
        // http://en.wikipedia.org/wiki/Conditional_comment#Downlevel-revealed_conditional_comment
        if (conditionalComment.test(html)) {
          // 找到结束位置
          const conditionalEnd = html.indexOf(']>')

          if (conditionalEnd >= 0) {
            // 调整 html 和 index 变量
            advance(conditionalEnd + 2)
            continue
          }
        }

        // 处理 Doctype，<!DOCTYPE html>
        const doctypeMatch = html.match(doctype)
        if (doctypeMatch) {
          advance(doctypeMatch[0].length)
          continue
        }

        /**
         * 处理开始标签和结束标签是这整个函数中的核型部分，其它的不用管
         * 这两部分就是在构造 element ast
         */

        // 处理结束标签，比如 </div>
        const endTagMatch = html.match(endTag)
        if (endTagMatch) {
          const curIndex = index
          advance(endTagMatch[0].length)
          // 处理结束标签
          parseEndTag(endTagMatch[1], curIndex, index)
          continue
        }

        // 处理开始标签，比如 <div id="app">，startTagMatch = { tagName: 'div', attrs: [[xx], ...], start: index }
        const startTagMatch = parseStartTag()
        if (startTagMatch) {
          // 进一步处理上一步得到结果，并最后调用 options.start 方法
          // 真正的解析工作都是在这个 start 方法中做的
          handleStartTag(startTagMatch)
          if (shouldIgnoreFirstNewline(startTagMatch.tagName, html)) {
            advance(1)
          }
          continue
        }
      }

      let text, rest, next
      if (textEnd >= 0) {
        // 能走到这儿，说明虽然在 html 中匹配到到了 <xx，但是这不属于上述几种情况，
        // 它就只是一个普通的一段文本：<我是文本
        // 于是从 html 中找到下一个 <，直到 <xx 是上述几种情况的标签，则结束，
        // 在这整个过程中一直在调整 textEnd 的值，作为 html 中下一个有效标签的开始位置

        // 截取 html 模版字符串中 textEnd 之后的内容，rest = <xx
        rest = html.slice(textEnd)
        // 这个 while 循环就是处理 <xx 之后的纯文本情况
        // 截取文本内容，并找到有效标签的开始位置（textEnd）
        while (
          !endTag.test(rest) &&
          !startTagOpen.test(rest) &&
          !comment.test(rest) &&
          !conditionalComment.test(rest)
        ) {
          // 则认为 < 后面的内容为纯文本，然后在这些纯文本中再次找 <
          next = rest.indexOf('<', 1)
          // 如果没找到 <，则直接结束循环
          if (next < 0) break
          // 走到这儿说明在后续的字符串中找到了 <，索引位置为 textEnd
          textEnd += next
          // 截取 html 字符串模版 textEnd 之后的内容赋值给 rest，继续判断之后的字符串是否存在标签
          rest = html.slice(textEnd)
        }
        // 走到这里，说明遍历结束，有两种情况，一种是 < 之后就是一段纯文本，要不就是在后面找到了有效标签，截取文本
        text = html.substring(0, textEnd)
      }

      // 如果 textEnd < 0，说明 html 中就没找到 <，那说明 html 就是一段文本
      if (textEnd < 0) {
        text = html
      }

      // 将文本内容从 html 模版字符串上截取掉
      if (text) {
        advance(text.length)
      }

      // 处理文本
      // 基于文本生成 ast 对象，然后将该 ast 放到它的父元素的肚子里，
      // 即 currentParent.children 数组中
      if (options.chars && text) {
        options.chars(text, index - text.length, index)
      }
    } else {
      // 处理 script、style、textarea 标签的闭合标签
      let endTagLength = 0
      // 开始标签的小写形式
      const stackedTag = lastTag.toLowerCase()
      const reStackedTag = reCache[stackedTag] || (reCache[stackedTag] = new RegExp('([\\s\\S]*?)(</' + stackedTag + '[^>]*>)', 'i'))
      // 匹配并处理开始标签和结束标签之间的所有文本，比如 <script>xx</script>
      const rest = html.replace(reStackedTag, function (all, text, endTag) {
        endTagLength = endTag.length
        if (!isPlainTextElement(stackedTag) && stackedTag !== 'noscript') {
          text = text
            .replace(/<!\--([\s\S]*?)-->/g, '$1') // #7298
            .replace(/<!\[CDATA\[([\s\S]*?)]]>/g, '$1')
        }
        if (shouldIgnoreFirstNewline(stackedTag, text)) {
          text = text.slice(1)
        }
        if (options.chars) {
          options.chars(text)
        }
        return ''
      })
      index += html.length - rest.length
      html = rest
      parseEndTag(stackedTag, index - endTagLength, index)
    }

    // 到这里就处理结束，如果 stack 数组中还有内容，则说明有标签没有被闭合，给出提示信息
    if (html === last) {
      options.chars && options.chars(html)
      if (process.env.NODE_ENV !== 'production' && !stack.length && options.warn) {
        options.warn(`Mal-formatted tag at end of template: "${html}"`, { start: index + html.length })
      }
      break
    }
  }

  // Clean up any remaining tags
  parseEndTag()
}

```

#### advance

> /src/compiler/parser/html-parser.js

```javascript
/**
 * 重置 html，html = 从索引 n 位置开始的向后的所有字符
 * index 为 html 在 原始的 模版字符串 中的的开始索引，也是下一次该处理的字符的开始位置
 * @param {*} n 索引
 */
function advance(n) {
  index += n
  html = html.substring(n)
}

```

#### parseStartTag

> /src/compiler/parser/html-parser.js

```javascript
/**
 * 解析开始标签，比如：<div id="app">
 * @returns { tagName: 'div', attrs: [[xx], ...], start: index }
 */
function parseStartTag() {
  const start = html.match(startTagOpen)
  if (start) {
    // 处理结果
    const match = {
      // 标签名
      tagName: start[1],
      // 属性，占位符
      attrs: [],
      // 标签的开始位置
      start: index
    }
    /**
     * 调整 html 和 index，比如：
     *   html = ' id="app">'
     *   index = 此时的索引
     *   start[0] = '<div'
     */
    advance(start[0].length)
    let end, attr
    // 处理 开始标签 内的各个属性，并将这些属性放到 match.attrs 数组中
    while (!(end = html.match(startTagClose)) && (attr = html.match(dynamicArgAttribute) || html.match(attribute))) {
      attr.start = index
      advance(attr[0].length)
      attr.end = index
      match.attrs.push(attr)
    }
    // 开始标签的结束，end = '>' 或 end = ' />'
    if (end) {
      match.unarySlash = end[1]
      advance(end[0].length)
      match.end = index
      return match
    }
  }
}

```

#### handleStartTag

> /src/compiler/parser/html-parser.js

```javascript
/**
 * 进一步处理开始标签的解析结果 ——— match 对象
 *  处理属性 match.attrs，如果不是自闭合标签，则将标签信息放到 stack 数组，待将来处理到它的闭合标签时再将其弹出 stack，表示该标签处理完毕，这时标签的所有信息都在 element ast 对象上了
 *  接下来调用 options.start 方法处理标签，并根据标签信息生成 element ast，
 *  以及处理开始标签上的属性和指令，最后将 element ast 放入 stack 数组
 * 
 * @param {*} match { tagName: 'div', attrs: [[xx], ...], start: index }
 */
function handleStartTag(match) {
  const tagName = match.tagName
  // />
  const unarySlash = match.unarySlash

  if (expectHTML) {
    if (lastTag === 'p' && isNonPhrasingTag(tagName)) {
      parseEndTag(lastTag)
    }
    if (canBeLeftOpenTag(tagName) && lastTag === tagName) {
      parseEndTag(tagName)
    }
  }

  // 一元标签，比如 <hr />
  const unary = isUnaryTag(tagName) || !!unarySlash

  // 处理 match.attrs，得到 attrs = [{ name: attrName, value: attrVal, start: xx, end: xx }, ...]
  // 比如 attrs = [{ name: 'id', value: 'app', start: xx, end: xx }, ...]
  const l = match.attrs.length
  const attrs = new Array(l)
  for (let i = 0; i < l; i++) {
    const args = match.attrs[i]
    // 比如：args[3] => 'id'，args[4] => '='，args[5] => 'app'
    const value = args[3] || args[4] || args[5] || ''
    const shouldDecodeNewlines = tagName === 'a' && args[1] === 'href'
      ? options.shouldDecodeNewlinesForHref
      : options.shouldDecodeNewlines
    // attrs[i] = { id: 'app' }
    attrs[i] = {
      name: args[1],
      value: decodeAttr(value, shouldDecodeNewlines)
    }
    // 非生产环境，记录属性的开始和结束索引
    if (process.env.NODE_ENV !== 'production' && options.outputSourceRange) {
      attrs[i].start = args.start + args[0].match(/^\s*/).length
      attrs[i].end = args.end
    }
  }

  // 如果不是自闭合标签，则将标签信息放到 stack 数组中，待将来处理到它的闭合标签时再将其弹出 stack
  // 如果是自闭合标签，则标签信息就没必要进入 stack 了，直接处理众多属性，将他们都设置到 element ast 对象上，就没有处理 结束标签的那一步了，这一步在处理开始标签的过程中就进行了
  if (!unary) {
    // 将标签信息放到 stack 数组中，{ tag, lowerCasedTag, attrs, start, end }
    stack.push({ tag: tagName, lowerCasedTag: tagName.toLowerCase(), attrs: attrs, start: match.start, end: match.end })
    // 标识当前标签的结束标签为 tagName
    lastTag = tagName
  }

  /**
   * 调用 start 方法，主要做了以下 6 件事情:
   *   1、创建 AST 对象
   *   2、处理存在 v-model 指令的 input 标签，分别处理 input 为 checkbox、radio、其它的情况
   *   3、处理标签上的众多指令，比如 v-pre、v-for、v-if、v-once
   *   4、如果根节点 root 不存在则设置当前元素为根节点
   *   5、如果当前元素为非自闭合标签则将自己 push 到 stack 数组，并记录 currentParent，在接下来处理子元素时用来告诉子元素自己的父节点是谁
   *   6、如果当前元素为自闭合标签，则表示该标签要处理结束了，让自己和父元素产生关系，以及设置自己的子元素
   */
  if (options.start) {
    options.start(tagName, attrs, unary, match.start, match.end)
  }
}

```

#### parseEndTag

> /src/compiler/parser/html-parser.js

```javascript
/**
 * 解析结束标签，比如：</div>
 * 最主要的事就是：
 *   1、处理 stack 数组，从 stack 数组中找到当前结束标签对应的开始标签，然后调用 options.end 方法
 *   2、处理完结束标签之后调整 stack 数组，保证在正常情况下 stack 数组中的最后一个元素就是下一个结束标签对应的开始标签
 *   3、处理一些异常情况，比如 stack 数组最后一个元素不是当前结束标签对应的开始标签，还有就是
 *      br 和 p 标签单独处理
 * @param {*} tagName 标签名，比如 div
 * @param {*} start 结束标签的开始索引
 * @param {*} end 结束标签的结束索引
 */
function parseEndTag(tagName, start, end) {
  let pos, lowerCasedTagName
  if (start == null) start = index
  if (end == null) end = index

  // 倒序遍历 stack 数组，找到第一个和当前结束标签相同的标签，该标签就是结束标签对应的开始标签的描述对象
  // 理论上，不出异常，stack 数组中的最后一个元素就是当前结束标签的开始标签的描述对象
  // Find the closest opened tag of the same type
  if (tagName) {
    lowerCasedTagName = tagName.toLowerCase()
    for (pos = stack.length - 1; pos >= 0; pos--) {
      if (stack[pos].lowerCasedTag === lowerCasedTagName) {
        break
      }
    }
  } else {
    // If no tag name is provided, clean shop
    pos = 0
  }

  // 如果在 stack 中一直没有找到相同的标签名，则 pos 就会 < 0，进行后面的 else 分支

  if (pos >= 0) {
    // 这个 for 循环负责关闭 stack 数组中索引 >= pos 的所有标签
    // 为什么要用一个循环，上面说到正常情况下 stack 数组的最后一个元素就是我们要找的开始标签，
    // 但是有些异常情况，就是有些元素没有给提供结束标签，比如：
    // stack = ['span', 'div', 'span', 'h1']，当前处理的结束标签 tagName = div
    // 匹配到 div，pos = 1，那索引为 2 和 3 的两个标签（span、h1）说明就没提供结束标签
    // 这个 for 循环就负责关闭 div、span 和 h1 这三个标签，
    // 并在开发环境为 span 和 h1 这两个标签给出 ”未匹配到结束标签的提示”
    // Close all the open elements, up the stack
    for (let i = stack.length - 1; i >= pos; i--) {
      if (process.env.NODE_ENV !== 'production' &&
        (i > pos || !tagName) &&
        options.warn
      ) {
        options.warn(
          `tag <${stack[i].tag}> has no matching end tag.`,
          { start: stack[i].start, end: stack[i].end }
        )
      }
      if (options.end) {
        // 走到这里，说明上面的异常情况都处理完了，调用 options.end 处理正常的结束标签
        options.end(stack[i].tag, start, end)
      }
    }

    // 将刚才处理的那些标签从数组中移除，保证数组的最后一个元素就是下一个结束标签对应的开始标签
    // Remove the open elements from the stack
    stack.length = pos
    // lastTag 记录 stack 数组中未处理的最后一个开始标签
    lastTag = pos && stack[pos - 1].tag
  } else if (lowerCasedTagName === 'br') {
    // 当前处理的标签为 <br /> 标签
    if (options.start) {
      options.start(tagName, [], true, start, end)
    }
  } else if (lowerCasedTagName === 'p') {
    // p 标签
    if (options.start) {
      // 处理 <p> 标签
      options.start(tagName, [], false, start, end)
    }
    if (options.end) {
      // 处理 </p> 标签
      options.end(tagName, start, end)
    }
  }
}

```

### parseHtmlOptions

> src/compiler/parser/index.js

定义如何处理开始标签、结束标签、文本节点和注释节点。

#### start

```javascript
/**
 * 主要做了以下 6 件事情:
 *   1、创建 AST 对象
 *   2、处理存在 v-model 指令的 input 标签，分别处理 input 为 checkbox、radio、其它的情况
 *   3、处理标签上的众多指令，比如 v-pre、v-for、v-if、v-once
 *   4、如果根节点 root 不存在则设置当前元素为根节点
 *   5、如果当前元素为非自闭合标签则将自己 push 到 stack 数组，并记录 currentParent，在接下来处理子元素时用来告诉子元素自己的父节点是谁
 *   6、如果当前元素为自闭合标签，则表示该标签要处理结束了，让自己和父元素产生关系，以及设置自己的子元素
 * @param {*} tag 标签名
 * @param {*} attrs [{ name: attrName, value: attrVal, start, end }, ...] 形式的属性数组
 * @param {*} unary 自闭合标签
 * @param {*} start 标签在 html 字符串中的开始索引
 * @param {*} end 标签在 html 字符串中的结束索引
 */
function start(tag, attrs, unary, start, end) {
  // 检查命名空间，如果存在，则继承父命名空间
  // check namespace.
  // inherit parent ns if there is one
  const ns = (currentParent && currentParent.ns) || platformGetTagNamespace(tag)

  // handle IE svg bug
  /* istanbul ignore if */
  if (isIE && ns === 'svg') {
    attrs = guardIESVGBug(attrs)
  }

  // 创建当前标签的 AST 对象
  let element: ASTElement = createASTElement(tag, attrs, currentParent)
  // 设置命名空间
  if (ns) {
    element.ns = ns
  }

  // 这段在非生产环境下会走，在 ast 对象上添加 一些 属性，比如 start、end
  if (process.env.NODE_ENV !== 'production') {
    if (options.outputSourceRange) {
      element.start = start
      element.end = end
      // 将属性数组解析成 { attrName: { name: attrName, value: attrVal, start, end }, ... } 形式的对象
      element.rawAttrsMap = element.attrsList.reduce((cumulated, attr) => {
        cumulated[attr.name] = attr
        return cumulated
      }, {})
    }
    // 验证属性是否有效，比如属性名不能包含: spaces, quotes, <, >, / or =.
    attrs.forEach(attr => {
      if (invalidAttributeRE.test(attr.name)) {
        warn(
          `Invalid dynamic argument expression: attribute names cannot contain ` +
          `spaces, quotes, <, >, / or =.`,
          {
            start: attr.start + attr.name.indexOf(`[`),
            end: attr.start + attr.name.length
          }
        )
      }
    })
  }

  // 非服务端渲染的情况下，模版中不应该出现 style、script 标签
  if (isForbiddenTag(element) && !isServerRendering()) {
    element.forbidden = true
    process.env.NODE_ENV !== 'production' && warn(
      'Templates should only be responsible for mapping the state to the ' +
      'UI. Avoid placing tags with side-effects in your templates, such as ' +
      `<${tag}>` + ', as they will not be parsed.',
      { start: element.start }
    )
  }

  /**
   * 为 element 对象分别执行 class、style、model 模块中的 preTransforms 方法
   * 不过 web 平台只有 model 模块有 preTransforms 方法
   * 用来处理存在 v-model 的 input 标签，但没处理 v-model 属性
   * 分别处理了 input 为 checkbox、radio 和 其它的情况
   * input 具体是哪种情况由 el.ifConditions 中的条件来判断
   * <input v-mode="test" :type="checkbox or radio or other(比如 text)" />
   */
  // apply pre-transforms
  for (let i = 0; i < preTransforms.length; i++) {
    element = preTransforms[i](element, options) || element
  }

  if (!inVPre) {
    // 表示 element 是否存在 v-pre 指令，存在则设置 element.pre = true
    processPre(element)
    if (element.pre) {
      // 存在 v-pre 指令，则设置 inVPre 为 true
      inVPre = true
    }
  }
  // 如果 pre 标签，则设置 inPre 为 true
  if (platformIsPreTag(element.tag)) {
    inPre = true
  }

  if (inVPre) {
    // 说明标签上存在 v-pre 指令，这样的节点只会渲染一次，将节点上的属性都设置到 el.attrs 数组对象中，作为静态属性，数据更新时不会渲染这部分内容
    // 设置 el.attrs 数组对象，每个元素都是一个属性对象 { name: attrName, value: attrVal, start, end }
    processRawAttrs(element)
  } else if (!element.processed) {
    // structural directives
    // 处理 v-for 属性，得到 element.for = 可迭代对象 element.alias = 别名
    processFor(element)
    /**
     * 处理 v-if、v-else-if、v-else
     * 得到 element.if = "exp"，element.elseif = exp, element.else = true
     * v-if 属性会额外在 element.ifConditions 数组中添加 { exp, block } 对象
     */
    processIf(element)
    // 处理 v-once 指令，得到 element.once = true 
    processOnce(element)
  }

  // 如果 root 不存在，则表示当前处理的元素为第一个元素，即组件的 根 元素
  if (!root) {
    root = element
    if (process.env.NODE_ENV !== 'production') {
      // 检查根元素，对根元素有一些限制，比如：不能使用 slot 和 template 作为根元素，也不能在有状态组件的根元素上使用 v-for 指令
      checkRootConstraints(root)
    }
  }

  if (!unary) {
    // 非自闭合标签，通过 currentParent 记录当前元素，下一个元素在处理的时候，就知道自己的父元素是谁
    currentParent = element
    // 然后将 element push 到 stack 数组，将来处理到当前元素的闭合标签时再拿出来
    // 将当前标签的 ast 对象 push 到 stack 数组中，这里需要注意，在调用 options.start 方法
    // 之前也发生过一次 push 操作，那个 push 进来的是当前标签的一个基本配置信息
    stack.push(element)
  } else {
    /**
     * 说明当前元素为自闭合标签，主要做了 3 件事：
     *   1、如果元素没有被处理过，即 el.processed 为 false，则调用 processElement 方法处理节点上的众多属性
     *   2、让自己和父元素产生关系，将自己放到父元素的 children 数组中，并设置自己的 parent 属性为 currentParent
     *   3、设置自己的子元素，将自己所有非插槽的子元素放到自己的 children 数组中
     */
    closeElement(element)
  }
}
```

#### end

```javascript
/**
 * 处理结束标签
 * @param {*} tag 结束标签的名称
 * @param {*} start 结束标签的开始索引
 * @param {*} end 结束标签的结束索引
 */
function end(tag, start, end) {
  // 结束标签对应的开始标签的 ast 对象
  const element = stack[stack.length - 1]
  // pop stack
  stack.length -= 1
  // 这块儿有点不太理解，因为上一个元素有可能是当前元素的兄弟节点
  currentParent = stack[stack.length - 1]
  if (process.env.NODE_ENV !== 'production' && options.outputSourceRange) {
    element.end = end
  }
  /**
   * 主要做了 3 件事：
   *   1、如果元素没有被处理过，即 el.processed 为 false，则调用 processElement 方法处理节点上的众多属性
   *   2、让自己和父元素产生关系，将自己放到父元素的 children 数组中，并设置自己的 parent 属性为 currentParent
   *   3、设置自己的子元素，将自己所有非插槽的子元素放到自己的 children 数组中
   */
  closeElement(element)
}

```

#### chars

```javascript
/**
 * 处理文本，基于文本生成 ast 对象，然后将该 ast 放到它的父元素的肚子里，即 currentParent.children 数组中 
 */
function chars(text: string, start: number, end: number) {
  // 异常处理，currentParent 不存在说明这段文本没有父元素
  if (!currentParent) {
    if (process.env.NODE_ENV !== 'production') {
      if (text === template) { // 文本不能作为组件的根元素
        warnOnce(
          'Component template requires a root element, rather than just text.',
          { start }
        )
      } else if ((text = text.trim())) { // 放在根元素之外的文本会被忽略
        warnOnce(
          `text "${text}" outside root element will be ignored.`,
          { start }
        )
      }
    }
    return
  }
  // IE textarea placeholder bug
  /* istanbul ignore if */
  if (isIE &&
    currentParent.tag === 'textarea' &&
    currentParent.attrsMap.placeholder === text
  ) {
    return
  }
  // 当前父元素的所有孩子节点
  const children = currentParent.children
  // 对 text 进行一系列的处理，比如删除空白字符，或者存在 whitespaceOptions 选项，则 text 直接置为空或者空格
  if (inPre || text.trim()) {
    // 文本在 pre 标签内 或者 text.trim() 不为空
    text = isTextTag(currentParent) ? text : decodeHTMLCached(text)
  } else if (!children.length) {
    // 说明文本不在 pre 标签内而且 text.trim() 为空，而且当前父元素也没有孩子节点，
    // 则将 text 置为空
    // remove the whitespace-only node right after an opening tag
    text = ''
  } else if (whitespaceOption) {
    // 压缩处理
    if (whitespaceOption === 'condense') {
      // in condense mode, remove the whitespace node if it contains
      // line break, otherwise condense to a single space
      text = lineBreakRE.test(text) ? '' : ' '
    } else {
      text = ' '
    }
  } else {
    text = preserveWhitespace ? ' ' : ''
  }
  // 如果经过处理后 text 还存在
  if (text) {
    if (!inPre && whitespaceOption === 'condense') {
      // 不在 pre 节点中，并且配置选项中存在压缩选项，则将多个连续空格压缩为单个
      // condense consecutive whitespaces into single space
      text = text.replace(whitespaceRE, ' ')
    }
    let res
    // 基于 text 生成 AST 对象
    let child: ?ASTNode
    if (!inVPre && text !== ' ' && (res = parseText(text, delimiters))) {
      // 文本中存在表达式（即有界定符）
      child = {
        type: 2,
        // 表达式
        expression: res.expression,
        tokens: res.tokens,
        // 文本
        text
      }
    } else if (text !== ' ' || !children.length || children[children.length - 1].text !== ' ') {
      // 纯文本节点
      child = {
        type: 3,
        text
      }
    }
    // child 存在，则将 child 放到父元素的肚子里，即 currentParent.children 数组中
    if (child) {
      if (process.env.NODE_ENV !== 'production' && options.outputSourceRange) {
        child.start = start
        child.end = end
      }
      children.push(child)
    }
  }
},

```

#### comment

```javascript
/**
 * 处理注释节点
 */
function comment(text: string, start, end) {
  // adding anything as a sibling to the root node is forbidden
  // comments should still be allowed, but ignored
  // 禁止将任何内容作为 root 的节点的同级进行添加，注释应该被允许，但是会被忽略
  // 如果 currentParent 不存在，说明注释和 root 为同级，忽略
  if (currentParent) {
    // 注释节点的 ast
    const child: ASTText = {
      // 节点类型
      type: 3,
      // 注释内容
      text,
      // 是否为注释
      isComment: true
    }
    if (process.env.NODE_ENV !== 'production' && options.outputSourceRange) {
      // 记录节点的开始索引和结束索引
      child.start = start
      child.end = end
    }
    // 将当前注释节点放到父元素的 children 属性中
    currentParent.children.push(child)
  }
}

```

### createASTElement

> /src/compiler/parser/index.js

```javascript
/**
 * 为指定元素创建 AST 对象
 * @param {*} tag 标签名
 * @param {*} attrs 属性数组，[{ name: attrName, value: attrVal, start, end }, ...]
 * @param {*} parent 父元素
 * @returns { type: 1, tag, attrsList, attrsMap: makeAttrsMap(attrs), rawAttrsMap: {}, parent, children: []}
 */
export function createASTElement(
  tag: string,
  attrs: Array<ASTAttr>,
  parent: ASTElement | void
): ASTElement {
  return {
    // 节点类型
    type: 1,
    // 标签名
    tag,
    // 标签的属性数组
    attrsList: attrs,
    // 标签的属性对象 { attrName: attrVal, ... }
    attrsMap: makeAttrsMap(attrs),
    // 原始属性对象
    rawAttrsMap: {},
    // 父节点
    parent,
    // 孩子节点
    children: []
  }
}

```

### preTransformNode

> /src/platforms/web/compiler/modules/model.js

```javascript
/**
 * 处理存在 v-model 的 input 标签，但没处理 v-model 属性
 * 分别处理了 input 为 checkbox、radio 和 其它的情况
 * input 具体是哪种情况由 el.ifConditions 中的条件来判断
 * <input v-mode="test" :type="checkbox or radio or other(比如 text)" />
 * @param {*} el 
 * @param {*} options 
 * @returns branch0
 */
function preTransformNode (el: ASTElement, options: CompilerOptions) {
  if (el.tag === 'input') {
    const map = el.attrsMap
    // 不存在 v-model 属性，直接结束
    if (!map['v-model']) {
      return
    }

    // 获取 :type 的值
    let typeBinding
    if (map[':type'] || map['v-bind:type']) {
      typeBinding = getBindingAttr(el, 'type')
    }
    if (!map.type && !typeBinding && map['v-bind']) {
      typeBinding = `(${map['v-bind']}).type`
    }

    // 如果存在 type 属性
    if (typeBinding) {
      // 获取 v-if 的值，比如： <input v-model="test" :type="checkbox" v-if="test" />
      const ifCondition = getAndRemoveAttr(el, 'v-if', true)
      // &&test
      const ifConditionExtra = ifCondition ? `&&(${ifCondition})` : ``
      // 是否存在 v-else 属性，<input v-else />
      const hasElse = getAndRemoveAttr(el, 'v-else', true) != null
      // 获取 v-else-if 属性的值 <inpu v-else-if="test" />
      const elseIfCondition = getAndRemoveAttr(el, 'v-else-if', true)
      // 克隆一个新的 el 对象，分别处理 input 为 chekbox、radio 或 其它的情况
      // 具体是哪种情况，通过 el.ifConditins 条件来判断
      // 1. checkbox
      const branch0 = cloneASTElement(el)
      // process for on the main node
      // <input v-for="item in arr" :key="item" />
      // 处理 v-for 表达式，得到 branch0.for = arr, branch0.alias = item
      processFor(branch0)
      // 在 branch0.attrsMap 和 branch0.attrsList 对象中添加 type 属性
      addRawAttr(branch0, 'type', 'checkbox')
      // 分别处理元素节点的 key、ref、插槽、自闭合的 slot 标签、动态组件、class、style、v-bind、v-on、其它指令和一些原生属性 
      processElement(branch0, options)
      // 标记当前对象已经被处理过了
      branch0.processed = true // prevent it from double-processed
      // 得到 true&&test or false&&test，标记当前 input 是否为 checkbox
      branch0.if = `(${typeBinding})==='checkbox'` + ifConditionExtra
      // 在 branch0.ifConfitions 数组中放入 { exp, block } 对象
      addIfCondition(branch0, {
        exp: branch0.if,
        block: branch0
      })
      // 克隆一个新的 ast 对象
      // 2. add radio else-if condition
      const branch1 = cloneASTElement(el)
      // 获取 v-for 属性值
      getAndRemoveAttr(branch1, 'v-for', true)
      // 在 branch1.attrsMap 和 branch1.attrsList 对象中添加 type 属性
      addRawAttr(branch1, 'type', 'radio')
      // 分别处理元素节点的 key、ref、插槽、自闭合的 slot 标签、动态组件、class、style、v-bind、v-on、其它指令和一些原生属性 
      processElement(branch1, options)
      // 在 branch0.ifConfitions 数组中放入 { exp, block } 对象
      addIfCondition(branch0, {
        // 标记当前 input 是否为 radio
        exp: `(${typeBinding})==='radio'` + ifConditionExtra,
        block: branch1
      })
      // 3. other，input 为其它的情况
      const branch2 = cloneASTElement(el)
      getAndRemoveAttr(branch2, 'v-for', true)
      addRawAttr(branch2, ':type', typeBinding)
      processElement(branch2, options)
      addIfCondition(branch0, {
        exp: ifCondition,
        block: branch2
      })

      // 给 branch0 设置 else 或 elseif 条件
      if (hasElse) {
        branch0.else = true
      } else if (elseIfCondition) {
        branch0.elseif = elseIfCondition
      }

      return branch0
    }
  }
}

```

### getBindingAttr

> /src/compiler/helpers.js

```javascript
/**
 * 获取 el 对象上执行属性 name 的值 
 */
export function getBindingAttr (
  el: ASTElement,
  name: string,
  getStatic?: boolean
): ?string {
  // 获取指定属性的值
  const dynamicValue =
    getAndRemoveAttr(el, ':' + name) ||
    getAndRemoveAttr(el, 'v-bind:' + name)
  if (dynamicValue != null) {
    return parseFilters(dynamicValue)
  } else if (getStatic !== false) {
    const staticValue = getAndRemoveAttr(el, name)
    if (staticValue != null) {
      return JSON.stringify(staticValue)
    }
  }
}

```

### getAndRemoveAttr

> /src/compiler/helpers.js

```javascript
/**
 * 从 el.attrsList 中删除指定的属性 name
 * 如果 removeFromMap 为 true，则同样删除 el.attrsMap 对象中的该属性，
 *   比如 v-if、v-else-if、v-else 等属性就会被移除,
 *   不过一般不会删除该对象上的属性，因为从 ast 生成 代码 期间还需要使用该对象
 * 返回指定属性的值
 */
// note: this only removes the attr from the Array (attrsList) so that it
// doesn't get processed by processAttrs.
// By default it does NOT remove it from the map (attrsMap) because the map is
// needed during codegen.
export function getAndRemoveAttr (
  el: ASTElement,
  name: string,
  removeFromMap?: boolean
): ?string {
  let val
  // 将执行属性 name 从 el.attrsList 中移除
  if ((val = el.attrsMap[name]) != null) {
    const list = el.attrsList
    for (let i = 0, l = list.length; i < l; i++) {
      if (list[i].name === name) {
        list.splice(i, 1)
        break
      }
    }
  }
  // 如果 removeFromMap 为 true，则从 el.attrsMap 中移除指定的属性 name
  // 不过一般不会移除 el.attsMap 中的数据，因为从 ast 生成 代码 期间还需要使用该对象
  if (removeFromMap) {
    delete el.attrsMap[name]
  }
  // 返回执行属性的值
  return val
}

```

### processFor

> /src/compiler/parser/index.js

```javascript
/**
 * 处理 v-for，将结果设置到 el 对象上，得到:
 *   el.for = 可迭代对象，比如 arr
 *   el.alias = 别名，比如 item
 * @param {*} el 元素的 ast 对象
 */
export function processFor(el: ASTElement) {
  let exp
  // 获取 el 上的 v-for 属性的值
  if ((exp = getAndRemoveAttr(el, 'v-for'))) {
    // 解析 v-for 的表达式，得到 { for: 可迭代对象， alias: 别名 }，比如 { for: arr, alias: item }
    const res = parseFor(exp)
    if (res) {
      // 将 res 对象上的属性拷贝到 el 对象上
      extend(el, res)
    } else if (process.env.NODE_ENV !== 'production') {
      warn(
        `Invalid v-for expression: ${exp}`,
        el.rawAttrsMap['v-for']
      )
    }
  }
}

```

### addRawAttr

> /src/compiler/helpers.js

```javascript
// 在 el.attrsMap 和 el.attrsList 中添加指定属性 name
// add a raw attr (use this in preTransforms)
export function addRawAttr (el: ASTElement, name: string, value: any, range?: Range) {
  el.attrsMap[name] = value
  el.attrsList.push(rangeSetItem({ name, value }, range))
}

```

### processElement

> /src/compiler/parser/index.js

```javascript
/**
 * 分别处理元素节点的 key、ref、插槽、自闭合的 slot 标签、动态组件、class、style、v-bind、v-on、其它指令和一些原生属性 
 * 然后在 el 对象上添加如下属性：
 * el.key、ref、refInFor、scopedSlot、slotName、component、inlineTemplate、staticClass
 * el.bindingClass、staticStyle、bindingStyle、attrs
 * @param {*} element 被处理元素的 ast 对象
 * @param {*} options 配置项
 * @returns 
 */
export function processElement(
  element: ASTElement,
  options: CompilerOptions
) {
  // el.key = val
  processKey(element)

  // 确定 element 是否为一个普通元素
  // determine whether this is a plain element after
  // removing structural attributes
  element.plain = (
    !element.key &&
    !element.scopedSlots &&
    !element.attrsList.length
  )

  // el.ref = val, el.refInFor = boolean
  processRef(element)
  // 处理作为插槽传递给组件的内容，得到  插槽名称、是否为动态插槽、作用域插槽的值，以及插槽中的所有子元素，子元素放到插槽对象的 children 属性中
  processSlotContent(element)
  // 处理自闭合的 slot 标签，得到插槽名称 => el.slotName = xx
  processSlotOutlet(element)
  // 处理动态组件，<component :is="compoName"></component>得到 el.component = compName，
  // 以及标记是否存在内联模版，el.inlineTemplate = true of false
  processComponent(element)
  // 为 element 对象分别执行 class、style、model 模块中的 transformNode 方法
  // 不过 web 平台只有 class、style 模块有 transformNode 方法，分别用来处理 class 属性和 style 属性
  // 得到 el.staticStyle、 el.styleBinding、el.staticClass、el.classBinding
  // 分别存放静态 style 属性的值、动态 style 属性的值，以及静态 class 属性的值和动态 class 属性的值
  for (let i = 0; i < transforms.length; i++) {
    element = transforms[i](element, options) || element
  }
  /**
   * 处理元素上的所有属性：
   * v-bind 指令变成：el.attrs 或 el.dynamicAttrs = [{ name, value, start, end, dynamic }, ...]，
   *                或者是必须使用 props 的属性，变成了 el.props = [{ name, value, start, end, dynamic }, ...]
   * v-on 指令变成：el.events 或 el.nativeEvents = { name: [{ value, start, end, modifiers, dynamic }, ...] }
   * 其它指令：el.directives = [{name, rawName, value, arg, isDynamicArg, modifier, start, end }, ...]
   * 原生属性：el.attrs = [{ name, value, start, end }]，或者一些必须使用 props 的属性，变成了：
   *         el.props = [{ name, value: true, start, end, dynamic }]
   */
  processAttrs(element)
  return element
}

```

### processKey

> /src/compiler/parser/index.js

```javascript
/**
 * 处理元素上的 key 属性，设置 el.key = val
 * @param {*} el 
 */
function processKey(el) {
  // 拿到 key 的属性值
  const exp = getBindingAttr(el, 'key')
  if (exp) {
    // 关于 key 使用上的异常处理
    if (process.env.NODE_ENV !== 'production') {
      // template 标签不允许设置 key
      if (el.tag === 'template') {
        warn(
          `<template> cannot be keyed. Place the key on real elements instead.`,
          getRawBindingAttr(el, 'key')
        )
      }
      // 不要在 <transition=group> 的子元素上使用 v-for 的 index 作为 key，这和没用 key 没什么区别
      if (el.for) {
        const iterator = el.iterator2 || el.iterator1
        const parent = el.parent
        if (iterator && iterator === exp && parent && parent.tag === 'transition-group') {
          warn(
            `Do not use v-for index as key on <transition-group> children, ` +
            `this is the same as not using keys.`,
            getRawBindingAttr(el, 'key'),
            true /* tip */
          )
        }
      }
    }
    // 设置 el.key = exp
    el.key = exp
  }
}

```

### processRef

> /src/compiler/parser/index.js

```javascript
/**
 * 处理元素上的 ref 属性
 *  el.ref = refVal
 *  el.refInFor = boolean
 * @param {*} el 
 */
function processRef(el) {
  const ref = getBindingAttr(el, 'ref')
  if (ref) {
    el.ref = ref
    // 判断包含 ref 属性的元素是否包含在具有 v-for 指令的元素内或后代元素中
    // 如果是，则 ref 指向的则是包含 DOM 节点或组件实例的数组
    el.refInFor = checkInFor(el)
  }
}

```

### processSlotContent

> /src/compiler/parser/index.js

```javascript
/**
 * 处理作为插槽传递给组件的内容，得到：
 *  slotTarget => 插槽名
 *  slotTargetDynamic => 是否为动态插槽
 *  slotScope => 作用域插槽的值
 *  直接在 <comp> 标签上使用 v-slot 语法时，将上述属性放到 el.scopedSlots 对象上，其它情况直接放到 el 对象上
 * handle content being passed to a component as slot,
 * e.g. <template slot="xxx">, <div slot-scope="xxx">
 */
function processSlotContent(el) {
  let slotScope
  if (el.tag === 'template') {
    // template 标签上使用 scope 属性的提示
    // scope 已经弃用，并在 2.5 之后使用 slot-scope 代替
    // slot-scope 即可以用在 template 标签也可以用在普通标签上
    slotScope = getAndRemoveAttr(el, 'scope')
    /* istanbul ignore if */
    if (process.env.NODE_ENV !== 'production' && slotScope) {
      warn(
        `the "scope" attribute for scoped slots have been deprecated and ` +
        `replaced by "slot-scope" since 2.5. The new "slot-scope" attribute ` +
        `can also be used on plain elements in addition to <template> to ` +
        `denote scoped slots.`,
        el.rawAttrsMap['scope'],
        true
      )
    }
    // el.slotScope = val
    el.slotScope = slotScope || getAndRemoveAttr(el, 'slot-scope')
  } else if ((slotScope = getAndRemoveAttr(el, 'slot-scope'))) {
    /* istanbul ignore if */
    if (process.env.NODE_ENV !== 'production' && el.attrsMap['v-for']) {
      // 元素不能同时使用 slot-scope 和 v-for，v-for 具有更高的优先级
      // 应该用 template 标签作为容器，将 slot-scope 放到 template 标签上 
      warn(
        `Ambiguous combined usage of slot-scope and v-for on <${el.tag}> ` +
        `(v-for takes higher priority). Use a wrapper <template> for the ` +
        `scoped slot to make it clearer.`,
        el.rawAttrsMap['slot-scope'],
        true
      )
    }
    el.slotScope = val
    el.slotScope = slotScope
  }

  // 获取 slot 属性的值
  // slot="xxx"，老旧的具名插槽的写法
  const slotTarget = getBindingAttr(el, 'slot')
  if (slotTarget) {
    // el.slotTarget = 插槽名（具名插槽）
    el.slotTarget = slotTarget === '""' ? '"default"' : slotTarget
    // 动态插槽名
    el.slotTargetDynamic = !!(el.attrsMap[':slot'] || el.attrsMap['v-bind:slot'])
    // preserve slot as an attribute for native shadow DOM compat
    // only for non-scoped slots.
    if (el.tag !== 'template' && !el.slotScope) {
      addAttr(el, 'slot', slotTarget, getRawBindingAttr(el, 'slot'))
    }
  }

  // 2.6 v-slot syntax
  if (process.env.NEW_SLOT_SYNTAX) {
    if (el.tag === 'template') {
      // v-slot 在 tempalte 标签上，得到 v-slot 的值
      // v-slot on <template>
      const slotBinding = getAndRemoveAttrByRegex(el, slotRE)
      if (slotBinding) {
        // 异常提示
        if (process.env.NODE_ENV !== 'production') {
          if (el.slotTarget || el.slotScope) {
            // 不同插槽语法禁止混合使用
            warn(
              `Unexpected mixed usage of different slot syntaxes.`,
              el
            )
          }
          if (el.parent && !maybeComponent(el.parent)) {
            // <template v-slot> 只能出现在组件的根位置，比如：
            // <comp>
            //   <template v-slot>xx</template>
            // </comp>
            // 而不能是
            // <comp>
            //   <div>
            //     <template v-slot>xxx</template>
            //   </div>
            // </comp>
            warn(
              `<template v-slot> can only appear at the root level inside ` +
              `the receiving component`,
              el
            )
          }
        }
        // 得到插槽名称
        const { name, dynamic } = getSlotName(slotBinding)
        // 插槽名
        el.slotTarget = name
        // 是否为动态插槽
        el.slotTargetDynamic = dynamic
        // 作用域插槽的值
        el.slotScope = slotBinding.value || emptySlotScopeToken // force it into a scoped slot for perf
      }
    } else {
      // 处理组件上的 v-slot，<comp v-slot:header />
      // slotBinding = { name: "v-slot:header", value: "", start, end}
      // v-slot on component, denotes default slot
      const slotBinding = getAndRemoveAttrByRegex(el, slotRE)
      if (slotBinding) {
        // 异常提示
        if (process.env.NODE_ENV !== 'production') {
          // el 不是组件的话，提示，v-slot 只能出现在组件上或 template 标签上
          if (!maybeComponent(el)) {
            warn(
              `v-slot can only be used on components or <template>.`,
              slotBinding
            )
          }
          // 语法混用
          if (el.slotScope || el.slotTarget) {
            warn(
              `Unexpected mixed usage of different slot syntaxes.`,
              el
            )
          }
          // 为了避免作用域歧义，当存在其他命名槽时，默认槽也应该使用<template>语法
          if (el.scopedSlots) {
            warn(
              `To avoid scope ambiguity, the default slot should also use ` +
              `<template> syntax when there are other named slots.`,
              slotBinding
            )
          }
        }
        // 将组件的孩子添加到它的默认插槽内
        // add the component's children to its default slot
        const slots = el.scopedSlots || (el.scopedSlots = {})
        // 获取插槽名称以及是否为动态插槽
        const { name, dynamic } = getSlotName(slotBinding)
        // 创建一个 template 标签的 ast 对象，用于容纳插槽内容，父级是 el
        const slotContainer = slots[name] = createASTElement('template', [], el)
        // 插槽名
        slotContainer.slotTarget = name
        // 是否为动态插槽
        slotContainer.slotTargetDynamic = dynamic
        // 所有的孩子，将每一个孩子的 parent 属性都设置为 slotContainer
        slotContainer.children = el.children.filter((c: any) => {
          if (!c.slotScope) {
            // 给插槽内元素设置 parent 属性为 slotContainer，也就是 template 元素
            c.parent = slotContainer
            return true
          }
        })
        slotContainer.slotScope = slotBinding.value || emptySlotScopeToken
        // remove children as they are returned from scopedSlots now
        el.children = []
        // mark el non-plain so data gets generated
        el.plain = false
      }
    }
  }
}

```

### getSlotName

> /src/compiler/parser/index.js

```javascript
/**
 * 解析 binding，得到插槽名称以及是否为动态插槽
 * @returns { name: 插槽名称, dynamic: 是否为动态插槽 }
 */
function getSlotName(binding) {
  let name = binding.name.replace(slotRE, '')
  if (!name) {
    if (binding.name[0] !== '#') {
      name = 'default'
    } else if (process.env.NODE_ENV !== 'production') {
      warn(
        `v-slot shorthand syntax requires a slot name.`,
        binding
      )
    }
  }
  return dynamicArgRE.test(name)
    // dynamic [name]
    ? { name: name.slice(1, -1), dynamic: true }
    // static name
    : { name: `"${name}"`, dynamic: false }
}

```

### processSlotOutlet

> /src/compiler/parser/index.js

```javascript
// handle <slot/> outlets，处理自闭合 slot 标签
// 得到插槽名称，el.slotName
function processSlotOutlet(el) {
  if (el.tag === 'slot') {
    // 得到插槽名称
    el.slotName = getBindingAttr(el, 'name')
    // 提示信息，不要在 slot 标签上使用 key 属性
    if (process.env.NODE_ENV !== 'production' && el.key) {
      warn(
        `\`key\` does not work on <slot> because slots are abstract outlets ` +
        `and can possibly expand into multiple elements. ` +
        `Use the key on a wrapping element instead.`,
        getRawBindingAttr(el, 'key')
      )
    }
  }
}

```

### processComponent

> /src/compiler/parser/index.js

```javascript
/**
 * 处理动态组件，<component :is="compName"></component>
 * 得到 el.component = compName
 */
function processComponent(el) {
  let binding
  // 解析 is 属性，得到属性值，即组件名称，el.component = compName
  if ((binding = getBindingAttr(el, 'is'))) {
    el.component = binding
  }
  // <component :is="compName" inline-template>xx</component>
  // 组件上存在 inline-template 属性，进行标记：el.inlineTemplate = true
  // 表示组件开始和结束标签内的内容作为组件模版出现，而不是作为插槽别分发，方便定义组件模版
  if (getAndRemoveAttr(el, 'inline-template') != null) {
    el.inlineTemplate = true
  }
}

```

### transformNode

> /src/platforms/web/compiler/modules/class.js

```javascript
/**
 * 处理元素上的 class 属性
 * 静态的 class 属性值赋值给 el.staticClass 属性
 * 动态的 class 属性值赋值给 el.classBinding 属性
 */
function transformNode (el: ASTElement, options: CompilerOptions) {
  // 日志
  const warn = options.warn || baseWarn
  // 获取元素上静态 class 属性的值 xx，<div class="xx"></div>
  const staticClass = getAndRemoveAttr(el, 'class')
  if (process.env.NODE_ENV !== 'production' && staticClass) {
    const res = parseText(staticClass, options.delimiters)
    // 提示，同 style 的提示一样，不能使用 <div class="{{ val}}"></div>，请用
    // <div :class="val"></div> 代替
    if (res) {
      warn(
        `class="${staticClass}": ` +
        'Interpolation inside attributes has been removed. ' +
        'Use v-bind or the colon shorthand instead. For example, ' +
        'instead of <div class="{{ val }}">, use <div :class="val">.',
        el.rawAttrsMap['class']
      )
    }
  }
  // 静态 class 属性值赋值给 el.staticClass
  if (staticClass) {
    el.staticClass = JSON.stringify(staticClass)
  }
  // 获取动态绑定的 class 属性值，并赋值给 el.classBinding
  const classBinding = getBindingAttr(el, 'class', false /* getStatic */)
  if (classBinding) {
    el.classBinding = classBinding
  }
}

```

### transformNode

> /src/platforms/web/compiler/modules/style.js

```javascript
/**
 * 从 el 上解析出静态的 style 属性和动态绑定的 style 属性，分别赋值给：
 * el.staticStyle 和 el.styleBinding
 * @param {*} el 
 * @param {*} options 
 */
function transformNode(el: ASTElement, options: CompilerOptions) {
  // 日志
  const warn = options.warn || baseWarn
  // <div style="xx"></div>
  // 获取 style 属性
  const staticStyle = getAndRemoveAttr(el, 'style')
  if (staticStyle) {
    // 提示，如果从 xx 中解析到了界定符，说明是一个动态的 style，
    // 比如 <div style="{{ val }}"></div>则给出提示:
    // 动态的 style 请使用 <div :style="val"></div>
    /* istanbul ignore if */
    if (process.env.NODE_ENV !== 'production') {
      const res = parseText(staticStyle, options.delimiters)
      if (res) {
        warn(
          `style="${staticStyle}": ` +
          'Interpolation inside attributes has been removed. ' +
          'Use v-bind or the colon shorthand instead. For example, ' +
          'instead of <div style="{{ val }}">, use <div :style="val">.',
          el.rawAttrsMap['style']
        )
      }
    }
    // 将静态的 style 样式赋值给 el.staticStyle
    el.staticStyle = JSON.stringify(parseStyleText(staticStyle))
  }

  // 获取动态绑定的 style 属性，比如 <div :style="{{ val }}"></div>
  const styleBinding = getBindingAttr(el, 'style', false /* getStatic */)
  if (styleBinding) {
    // 赋值给 el.styleBinding
    el.styleBinding = styleBinding
  }
}

```

## 链接

* [配套视频，微信公众号回复](https://gitee.com/liyongning/typora-image-bed/raw/master/202202171742614.jpg)："精通 Vue 技术栈源码原理视频版" 获取
* [精通 Vue 技术栈源码原理 专栏](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzA3NTk4NjQ1OQ==&action=getalbum&album_id=2273541436891693065#wechat_redirect)
* [github 仓库 liyongning/Vue](https://github.com/liyongning/Vue) 欢迎 Star



感谢各位的：**关注**、**点赞**、**收藏**和**评论**，我们下期见。

***

**当学习成为了习惯，知识也就变成了常识。** 感谢各位的 **关注**、 **点赞**、**收藏**和**评论**。

新视频和文章会第一时间在微信公众号发送，欢迎关注：[李永宁lyn](https://gitee.com/liyongning/typora-image-bed/raw/master/202202171742614.jpg)

文章已收录到 [github 仓库 liyongning/blog](https://github.com/liyongning/blog)，欢迎 Watch 和 Star。