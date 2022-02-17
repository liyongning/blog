# Vue 源码解读（9）—— 编译器 之 优化

**当学习成为了习惯，知识也就变成了常识。** 感谢各位的 **关注**、**点赞**、**收藏**和**评论**。

新视频和文章会第一时间在微信公众号发送，欢迎关注：[李永宁lyn](https://gitee.com/liyongning/typora-image-bed/raw/master/202202171742614.jpg)

文章已收录到 [github 仓库 liyongning/blog](https://github.com/liyongning/blog)，欢迎 Watch 和 Star。

## 封面

![](https://gitee.com/liyongning/typora-image-bed/raw/master/202202271641368.png)

## 前言

上一篇文章 [Vue 源码解读（8）—— 编译器 之 解析](https://mp.weixin.qq.com/s?__biz=MzA3NTk4NjQ1OQ==&mid=2247485080&idx=1&sn=db5351d2f6d9a6f3d2a87c8f4ad70157&chksm=9f6965eca81eecfac96f7034651b37c8e9b35eca024075e0bf9e24ecda5993d478f0674183ee#rd) 详细详解了编译器的第一部分，如何将 html 模版字符串编译成 AST。今天带来编译器的第二部分，优化 AST，也是大家常说的静态标记。

## 目标

深入理解编译器的静态标记过程

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
  // 具体有那些属性，查看 start 和 end 这两个处理开始和结束标签的方法
  const ast = parse(template.trim(), options)
  // 优化，遍历 AST，为每个节点做静态标记
  // 标记每个节点是否为静态节点，然后进一步标记出静态根节点
  // 这样在后续更新的过程中就可以跳过这些静态节点了
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
})

```

### optimize

> /src/compiler/optimizer.js

```javascript
/**
 * 优化：
 *   遍历 AST，标记每个节点是静态节点还是动态节点，然后标记静态根节点
 *   这样在后续更新的过程中就不需要再关注这些节点
 */
export function optimize(root: ?ASTElement, options: CompilerOptions) {
  if (!root) return
  /**
   * options.staticKeys = 'staticClass,staticStyle'
   * isStaticKey = function(val) { return map[val] }
   */
  isStaticKey = genStaticKeysCached(options.staticKeys || '')
  // 平台保留标签
  isPlatformReservedTag = options.isReservedTag || no
  // 遍历所有节点，给每个节点设置 static 属性，标识其是否为静态节点
  markStatic(root)
  // 进一步标记静态根，一个节点要成为静态根节点，需要具体以下条件：
  // 节点本身是静态节点，而且有子节点，而且子节点不只是一个文本节点，则标记为静态根
  // 静态根节点不能只有静态文本的子节点，因为这样收益太低，这种情况下始终更新它就好了
  markStaticRoots(root, false)
}

```

### markStatic

> /src/compiler/optimizer.js

```javascript
/**
 * 在所有节点上设置 static 属性，用来标识是否为静态节点
 * 注意：如果有子节点为动态节点，则父节点也被认为是动态节点
 * @param {*} node 
 * @returns 
 */
function markStatic(node: ASTNode) {
  // 通过 node.static 来标识节点是否为 静态节点
  node.static = isStatic(node)
  if (node.type === 1) {
    /**
     * 不要将组件的插槽内容设置为静态节点，这样可以避免：
     *   1、组件不能改变插槽节点
     *   2、静态插槽内容在热重载时失败
     */
    if (
      !isPlatformReservedTag(node.tag) &&
      node.tag !== 'slot' &&
      node.attrsMap['inline-template'] == null
    ) {
      // 递归终止条件，如果节点不是平台保留标签  && 也不是 slot 标签 && 也不是内联模版，则直接结束
      return
    }
    // 遍历子节点，递归调用 markStatic 来标记这些子节点的 static 属性
    for (let i = 0, l = node.children.length; i < l; i++) {
      const child = node.children[i]
      markStatic(child)
      // 如果子节点是非静态节点，则将父节点更新为非静态节点
      if (!child.static) {
        node.static = false
      }
    }
    // 如果节点存在 v-if、v-else-if、v-else 这些指令，则依次标记 block 中节点的 static
    if (node.ifConditions) {
      for (let i = 1, l = node.ifConditions.length; i < l; i++) {
        const block = node.ifConditions[i].block
        markStatic(block)
        if (!block.static) {
          node.static = false
        }
      }
    }
  }
}

```

### isStatic

> /src/compiler/optimizer.js

```javascript
/**
 * 判断节点是否为静态节点：
 *  通过自定义的 node.type 来判断，2: 表达式 => 动态，3: 文本 => 静态
 *  凡是有 v-bind、v-if、v-for 等指令的都属于动态节点
 *  组件为动态节点
 *  父节点为含有 v-for 指令的 template 标签，则为动态节点
 * @param {*} node 
 * @returns boolean
 */
function isStatic(node: ASTNode): boolean {
  if (node.type === 2) { // expression
    // 比如：{{ msg }}
    return false
  }
  if (node.type === 3) { // text
    return true
  }
  return !!(node.pre || (
    !node.hasBindings && // no dynamic bindings
    !node.if && !node.for && // not v-if or v-for or v-else
    !isBuiltInTag(node.tag) && // not a built-in
    isPlatformReservedTag(node.tag) && // not a component
    !isDirectChildOfTemplateFor(node) &&
    Object.keys(node).every(isStaticKey)
  ))
}

```

### markStaticRoots

> /src/compiler/optimizer.js

```javascript
/**
 * 进一步标记静态根，一个节点要成为静态根节点，需要具体以下条件：
 * 节点本身是静态节点，而且有子节点，而且子节点不只是一个文本节点，则标记为静态根
 * 静态根节点不能只有静态文本的子节点，因为这样收益太低，这种情况下始终更新它就好了
 * 
 * @param { ASTElement } node 当前节点
 * @param { boolean } isInFor 当前节点是否被包裹在 v-for 指令所在的节点内
 */
function markStaticRoots(node: ASTNode, isInFor: boolean) {
  if (node.type === 1) {
    if (node.static || node.once) {
      // 节点是静态的 或者 节点上有 v-once 指令，标记 node.staticInFor = true or false
      node.staticInFor = isInFor
    }

    if (node.static && node.children.length && !(
      node.children.length === 1 &&
      node.children[0].type === 3
    )) {
      // 节点本身是静态节点，而且有子节点，而且子节点不只是一个文本节点，则标记为静态根 => node.staticRoot = true，否则为非静态根
      node.staticRoot = true
      return
    } else {
      node.staticRoot = false
    }
    // 当前节点不是静态根节点的时候，递归遍历其子节点，标记静态根
    if (node.children) {
      for (let i = 0, l = node.children.length; i < l; i++) {
        markStaticRoots(node.children[i], isInFor || !!node.for)
      }
    }
    // 如果节点存在 v-if、v-else-if、v-else 指令，则为 block 节点标记静态根
    if (node.ifConditions) {
      for (let i = 1, l = node.ifConditions.length; i < l; i++) {
        markStaticRoots(node.ifConditions[i].block, isInFor)
      }
    }
  }
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

* **面试官**：详细说一下静态标记的过程

  **答**：
  
  * 标记静态节点
  
    * 通过递归的方式标记所有的元素节点
    
    * 如果节点本身是静态节点，但是存在非静态的子节点，则将节点修改为非静态节点
  
  * 标记静态根节点，基于静态节点，进一步标记静态根节点
  
    * 如果节点本身是静态节点 && 而且有子节点 && 子节点不全是文本节点，则标记为静态根节点
    
    * 如果节点本身不是静态根节点，则递归的遍历所有子节点，在子节点中标记静态根
    
<hr />

* **面试官**：什么样的节点才可以被标记为静态节点？

  **答**：
  
  * 文本节点
  
  * 节点上没有 v-bind、v-for、v-if 等指令
  
  * 非组件
  
## 链接

* [配套视频，微信公众号回复](https://gitee.com/liyongning/typora-image-bed/raw/master/202202171742614.jpg)："精通 Vue 技术栈源码原理视频版" 获取
* [精通 Vue 技术栈源码原理 专栏](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzA3NTk4NjQ1OQ==&action=getalbum&album_id=2273541436891693065#wechat_redirect)
* [github 仓库 liyongning/Vue](https://github.com/liyongning/Vue) 欢迎 Star



感谢各位的：**关注**、**点赞**、**收藏**和**评论**，我们下期见。

***

**当学习成为了习惯，知识也就变成了常识。** 感谢各位的 **关注**、 **点赞**、**收藏**和**评论**。

新视频和文章会第一时间在微信公众号发送，欢迎关注：[李永宁lyn](https://gitee.com/liyongning/typora-image-bed/raw/master/202202171742614.jpg)

文章已收录到 [github 仓库 liyongning/blog](https://github.com/liyongning/blog)，欢迎 Watch 和 Star。