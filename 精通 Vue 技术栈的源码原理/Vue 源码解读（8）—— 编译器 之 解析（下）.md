# Vue 源码解读（8）—— 编译器 之 解析（下）

**当学习成为了习惯，知识也就变成了常识。** 感谢各位的 **关注**、**点赞**、**收藏**和**评论**。

新视频和文章会第一时间在微信公众号发送，欢迎关注：[李永宁lyn](https://gitee.com/liyongning/typora-image-bed/raw/master/202202171742614.jpg)

文章已收录到 [github 仓库 liyongning/blog](https://github.com/liyongning/blog)，欢迎 Watch 和 Star。

## 封面

![](https://gitee.com/liyongning/typora-image-bed/raw/master/202202271636639.png)

## 特殊说明

由于文章篇幅限制，所以将 **Vue 源码解读（8）—— 编译器 之 解析** 拆成了两篇文章，本篇是对 [Vue 源码解读（8）—— 编译器 之 解析（上）](https://mp.weixin.qq.com/s?__biz=MzA3NTk4NjQ1OQ==&mid=2247485080&idx=1&sn=db5351d2f6d9a6f3d2a87c8f4ad70157&chksm=9f6965eca81eecfac96f7034651b37c8e9b35eca024075e0bf9e24ecda5993d478f0674183ee#rd) 的一个补充，所以在阅读时请同时打开 [Vue 源码解读（8）—— 编译器 之 解析（上）](https://mp.weixin.qq.com/s?__biz=MzA3NTk4NjQ1OQ==&mid=2247485080&idx=1&sn=db5351d2f6d9a6f3d2a87c8f4ad70157&chksm=9f6965eca81eecfac96f7034651b37c8e9b35eca024075e0bf9e24ecda5993d478f0674183ee#rd) 一起阅读。

### processAttrs

> /src/compiler/parser/index.js

```javascript
/**
 * 处理元素上的所有属性：
 * v-bind 指令变成：el.attrs 或 el.dynamicAttrs = [{ name, value, start, end, dynamic }, ...]，
 *                或者是必须使用 props 的属性，变成了 el.props = [{ name, value, start, end, dynamic }, ...]
 * v-on 指令变成：el.events 或 el.nativeEvents = { name: [{ value, start, end, modifiers, dynamic }, ...] }
 * 其它指令：el.directives = [{name, rawName, value, arg, isDynamicArg, modifier, start, end }, ...]
 * 原生属性：el.attrs = [{ name, value, start, end }]，或者一些必须使用 props 的属性，变成了：
 *         el.props = [{ name, value: true, start, end, dynamic }]
 */
function processAttrs(el) {
  // list = [{ name, value, start, end }, ...]
  const list = el.attrsList
  let i, l, name, rawName, value, modifiers, syncGen, isDynamic
  for (i = 0, l = list.length; i < l; i++) {
    // 属性名
    name = rawName = list[i].name
    // 属性值
    value = list[i].value
    if (dirRE.test(name)) {
      // 说明该属性是一个指令

      // 元素上存在指令，将元素标记动态元素
      // mark element as dynamic
      el.hasBindings = true
      // modifiers，在属性名上解析修饰符，比如 xx.lazy
      modifiers = parseModifiers(name.replace(dirRE, ''))
      // support .foo shorthand syntax for the .prop modifier
      if (process.env.VBIND_PROP_SHORTHAND && propBindRE.test(name)) {
        // 为 .props 修饰符支持 .foo 速记写法
        (modifiers || (modifiers = {})).prop = true
        name = `.` + name.slice(1).replace(modifierRE, '')
      } else if (modifiers) {
        // 属性中的修饰符去掉，得到一个干净的属性名
        name = name.replace(modifierRE, '')
      }
      if (bindRE.test(name)) { // v-bind, <div :id="test"></div>
        // 处理 v-bind 指令属性，最后得到 el.attrs 或者 el.dynamicAttrs = [{ name, value, start, end, dynamic }, ...]

        // 属性名，比如：id
        name = name.replace(bindRE, '')
        // 属性值，比如：test
        value = parseFilters(value)
        // 是否为动态属性 <div :[id]="test"></div>
        isDynamic = dynamicArgRE.test(name)
        if (isDynamic) {
          // 如果是动态属性，则去掉属性两侧的方括号 []
          name = name.slice(1, -1)
        }
        // 提示，动态属性值不能为空字符串
        if (
          process.env.NODE_ENV !== 'production' &&
          value.trim().length === 0
        ) {
          warn(
            `The value for a v-bind expression cannot be empty. Found in "v-bind:${name}"`
          )
        }
        // 存在修饰符
        if (modifiers) {
          if (modifiers.prop && !isDynamic) {
            name = camelize(name)
            if (name === 'innerHtml') name = 'innerHTML'
          }
          if (modifiers.camel && !isDynamic) {
            name = camelize(name)
          }
          // 处理 sync 修饰符
          if (modifiers.sync) {
            syncGen = genAssignmentCode(value, `$event`)
            if (!isDynamic) {
              addHandler(
                el,
                `update:${camelize(name)}`,
                syncGen,
                null,
                false,
                warn,
                list[i]
              )
              if (hyphenate(name) !== camelize(name)) {
                addHandler(
                  el,
                  `update:${hyphenate(name)}`,
                  syncGen,
                  null,
                  false,
                  warn,
                  list[i]
                )
              }
            } else {
              // handler w/ dynamic event name
              addHandler(
                el,
                `"update:"+(${name})`,
                syncGen,
                null,
                false,
                warn,
                list[i],
                true // dynamic
              )
            }
          }
        }
        if ((modifiers && modifiers.prop) || (
          !el.component && platformMustUseProp(el.tag, el.attrsMap.type, name)
        )) {
          // 将属性对象添加到 el.props 数组中，表示这些属性必须通过 props 设置
          // el.props = [{ name, value, start, end, dynamic }, ...]
          addProp(el, name, value, list[i], isDynamic)
        } else {
          // 将属性添加到 el.attrs 数组或者 el.dynamicAttrs 数组
          addAttr(el, name, value, list[i], isDynamic)
        }
      } else if (onRE.test(name)) { // v-on, 处理事件，<div @click="test"></div>
        // 属性名，即事件名
        name = name.replace(onRE, '')
        // 是否为动态属性
        isDynamic = dynamicArgRE.test(name)
        if (isDynamic) {
          // 动态属性，则获取 [] 中的属性名
          name = name.slice(1, -1)
        }
        // 处理事件属性，将属性的信息添加到 el.events 或者 el.nativeEvents 对象上，格式：
        // el.events = [{ value, start, end, modifiers, dynamic }, ...]
        addHandler(el, name, value, modifiers, false, warn, list[i], isDynamic)
      } else { // normal directives，其它的普通指令
        // 得到 el.directives = [{name, rawName, value, arg, isDynamicArg, modifier, start, end }, ...]
        name = name.replace(dirRE, '')
        // parse arg
        const argMatch = name.match(argRE)
        let arg = argMatch && argMatch[1]
        isDynamic = false
        if (arg) {
          name = name.slice(0, -(arg.length + 1))
          if (dynamicArgRE.test(arg)) {
            arg = arg.slice(1, -1)
            isDynamic = true
          }
        }
        addDirective(el, name, rawName, value, arg, isDynamic, modifiers, list[i])
        if (process.env.NODE_ENV !== 'production' && name === 'model') {
          checkForAliasModel(el, value)
        }
      }
    } else {
      // 当前属性不是指令
      // literal attribute
      if (process.env.NODE_ENV !== 'production') {
        const res = parseText(value, delimiters)
        if (res) {
          warn(
            `${name}="${value}": ` +
            'Interpolation inside attributes has been removed. ' +
            'Use v-bind or the colon shorthand instead. For example, ' +
            'instead of <div id="{{ val }}">, use <div :id="val">.',
            list[i]
          )
        }
      }
      // 将属性对象放到 el.attrs 数组中，el.attrs = [{ name, value, start, end }]
      addAttr(el, name, JSON.stringify(value), list[i])
      // #6887 firefox doesn't update muted state if set via attribute
      // even immediately after element creation
      if (!el.component &&
        name === 'muted' &&
        platformMustUseProp(el.tag, el.attrsMap.type, name)) {
        addProp(el, name, 'true', list[i])
      }
    }
  }
}

```

### addHandler

> /src/compiler/helpers.js

```javascript
/**
 * 处理事件属性，将事件属性添加到 el.events 对象或者 el.nativeEvents 对象中，格式：
 * el.events[name] = [{ value, start, end, modifiers, dynamic }, ...]
 * 其中用了大量的篇幅在处理 name 属性带修饰符 (modifier) 的情况
 * @param {*} el ast 对象
 * @param {*} name 属性名，即事件名
 * @param {*} value 属性值，即事件回调函数名
 * @param {*} modifiers 修饰符
 * @param {*} important 
 * @param {*} warn 日志
 * @param {*} range 
 * @param {*} dynamic 属性名是否为动态属性
 */
export function addHandler (
  el: ASTElement,
  name: string,
  value: string,
  modifiers: ?ASTModifiers,
  important?: boolean,
  warn?: ?Function,
  range?: Range,
  dynamic?: boolean
) {
  // modifiers 是一个对象，如果传递的参数为空，则给一个冻结的空对象
  modifiers = modifiers || emptyObject
  // 提示：prevent 和 passive 修饰符不能一起使用
  // warn prevent and passive modifier
  /* istanbul ignore if */
  if (
    process.env.NODE_ENV !== 'production' && warn &&
    modifiers.prevent && modifiers.passive
  ) {
    warn(
      'passive and prevent can\'t be used together. ' +
      'Passive handler can\'t prevent default event.',
      range
    )
  }

  // 标准化 click.right 和 click.middle，它们实际上不会被真正的触发，从技术讲他们是它们
  // 是特定于浏览器的，但至少目前位置只有浏览器才具有右键和中间键的点击
  // normalize click.right and click.middle since they don't actually fire
  // this is technically browser-specific, but at least for now browsers are
  // the only target envs that have right/middle clicks.
  if (modifiers.right) {
    // 右键
    if (dynamic) {
      // 动态属性
      name = `(${name})==='click'?'contextmenu':(${name})`
    } else if (name === 'click') {
      // 非动态属性，name = contextmenu
      name = 'contextmenu'
      // 删除修饰符中的 right 属性
      delete modifiers.right
    }
  } else if (modifiers.middle) {
    // 中间键
    if (dynamic) {
      // 动态属性，name => mouseup 或者 ${name}
      name = `(${name})==='click'?'mouseup':(${name})`
    } else if (name === 'click') {
      // 非动态属性，mouseup
      name = 'mouseup'
    }
  }

  /**
   * 处理 capture、once、passive 这三个修饰符，通过给 name 添加不同的标记来标记这些修饰符
   */
  // check capture modifier
  if (modifiers.capture) {
    delete modifiers.capture
    // 给带有 capture 修饰符的属性，加上 ! 标记
    name = prependModifierMarker('!', name, dynamic)
  }
  if (modifiers.once) {
    delete modifiers.once
    // once 修饰符加 ~ 标记
    name = prependModifierMarker('~', name, dynamic)
  }
  /* istanbul ignore if */
  if (modifiers.passive) {
    delete modifiers.passive
    // passive 修饰符加 & 标记
    name = prependModifierMarker('&', name, dynamic)
  }

  let events
  if (modifiers.native) {
    // native 修饰符， 监听组件根元素的原生事件，将事件信息存放到 el.nativeEvents 对象中
    delete modifiers.native
    events = el.nativeEvents || (el.nativeEvents = {})
  } else {
    events = el.events || (el.events = {})
  }

  const newHandler: any = rangeSetItem({ value: value.trim(), dynamic }, range)
  if (modifiers !== emptyObject) {
    // 说明有修饰符，将修饰符对象放到 newHandler 对象上
    // { value, dynamic, start, end, modifiers }
    newHandler.modifiers = modifiers
  }

  // 将配置对象放到 events[name] = [newHander, handler, ...]
  const handlers = events[name]
  /* istanbul ignore if */
  if (Array.isArray(handlers)) {
    important ? handlers.unshift(newHandler) : handlers.push(newHandler)
  } else if (handlers) {
    events[name] = important ? [newHandler, handlers] : [handlers, newHandler]
  } else {
    events[name] = newHandler
  }

  el.plain = false
}

```

### addIfCondition

> /src/compiler/parser/index.js

```javascript
/**
 * 将传递进来的条件对象放进 el.ifConditions 数组中
 */
export function addIfCondition(el: ASTElement, condition: ASTIfCondition) {
  if (!el.ifConditions) {
    el.ifConditions = []
  }
  el.ifConditions.push(condition)
}

```

### processPre

> /src/compiler/parser/index.js

```javascript
/**
 * 如果元素上存在 v-pre 指令，则设置 el.pre = true 
 */
function processPre(el) {
  if (getAndRemoveAttr(el, 'v-pre') != null) {
    el.pre = true
  }
}

```

### processRawAttrs

> /src/compiler/parser/index.js

```javascript
/**
 * 设置 el.attrs 数组对象，每个元素都是一个属性对象 { name: attrName, value: attrVal, start, end }
 */
function processRawAttrs(el) {
  const list = el.attrsList
  const len = list.length
  if (len) {
    const attrs: Array<ASTAttr> = el.attrs = new Array(len)
    for (let i = 0; i < len; i++) {
      attrs[i] = {
        name: list[i].name,
        value: JSON.stringify(list[i].value)
      }
      if (list[i].start != null) {
        attrs[i].start = list[i].start
        attrs[i].end = list[i].end
      }
    }
  } else if (!el.pre) {
    // non root node in pre blocks with no attributes
    el.plain = true
  }
}

```

### processIf

> /src/compiler/parser/index.js

```javascript
/**
 * 处理 v-if、v-else-if、v-else
 * 得到 el.if = "exp"，el.elseif = exp, el.else = true
 * v-if 属性会额外在 el.ifConditions 数组中添加 { exp, block } 对象
 */
function processIf(el) {
  // 获取 v-if 属性的值，比如 <div v-if="test"></div>
  const exp = getAndRemoveAttr(el, 'v-if')
  if (exp) {
    // el.if = "test"
    el.if = exp
    // 在 el.ifConditions 数组中添加 { exp, block }
    addIfCondition(el, {
      exp: exp,
      block: el
    })
  } else {
    // 处理 v-else，得到 el.else = true
    if (getAndRemoveAttr(el, 'v-else') != null) {
      el.else = true
    }
    // 处理 v-else-if，得到 el.elseif = exp
    const elseif = getAndRemoveAttr(el, 'v-else-if')
    if (elseif) {
      el.elseif = elseif
    }
  }
}

```

### processOnce

> /src/compiler/parser/index.js

```javascript
/**
 * 处理 v-once 指令，得到 el.once = true
 * @param {*} el 
 */
function processOnce(el) {
  const once = getAndRemoveAttr(el, 'v-once')
  if (once != null) {
    el.once = true
  }
}

```

### checkRootConstraints

> /src/compiler/parser/index.js

```javascript
/**
 * 检查根元素：
 *   不能使用 slot 和 template 标签作为组件的根元素
 *   不能在有状态组件的 根元素 上使用 v-for 指令，因为它会渲染出多个元素
 * @param {*} el 
 */
function checkRootConstraints(el) {
  // 不能使用 slot 和 template 标签作为组件的根元素
  if (el.tag === 'slot' || el.tag === 'template') {
    warnOnce(
      `Cannot use <${el.tag}> as component root element because it may ` +
      'contain multiple nodes.',
      { start: el.start }
    )
  }
  // 不能在有状态组件的 根元素 上使用 v-for，因为它会渲染出多个元素
  if (el.attrsMap.hasOwnProperty('v-for')) {
    warnOnce(
      'Cannot use v-for on stateful component root element because ' +
      'it renders multiple elements.',
      el.rawAttrsMap['v-for']
    )
  }
}

```

### closeElement

> /src/compiler/parser/index.js

```javascript
/**
 * 主要做了 3 件事：
 *   1、如果元素没有被处理过，即 el.processed 为 false，则调用 processElement 方法处理节点上的众多属性
 *   2、让自己和父元素产生关系，将自己放到父元素的 children 数组中，并设置自己的 parent 属性为 currentParent
 *   3、设置自己的子元素，将自己所有非插槽的子元素放到自己的 children 数组中
 */
function closeElement(element) {
  // 移除节点末尾的空格，当前 pre 标签内的元素除外
  trimEndingWhitespace(element)
  // 当前元素不再 pre 节点内，并且也没有被处理过
  if (!inVPre && !element.processed) {
    // 分别处理元素节点的 key、ref、插槽、自闭合的 slot 标签、动态组件、class、style、v-bind、v-on、其它指令和一些原生属性 
    element = processElement(element, options)
  }
  // 处理根节点上存在 v-if、v-else-if、v-else 指令的情况
  // 如果根节点存在 v-if 指令，则必须还提供一个具有 v-else-if 或者 v-else 的同级别节点，防止根元素不存在
  // tree management
  if (!stack.length && element !== root) {
    // allow root elements with v-if, v-else-if and v-else
    if (root.if && (element.elseif || element.else)) {
      if (process.env.NODE_ENV !== 'production') {
        // 检查根元素
        checkRootConstraints(element)
      }
      // 给根元素设置 ifConditions 属性，root.ifConditions = [{ exp: element.elseif, block: element }, ...]
      addIfCondition(root, {
        exp: element.elseif,
        block: element
      })
    } else if (process.env.NODE_ENV !== 'production') {
      // 提示，表示不应该在 根元素 上只使用 v-if，应该将 v-if、v-else-if 一起使用，保证组件只有一个根元素
      warnOnce(
        `Component template should contain exactly one root element. ` +
        `If you are using v-if on multiple elements, ` +
        `use v-else-if to chain them instead.`,
        { start: element.start }
      )
    }
  }
  // 让自己和父元素产生关系
  // 将自己放到父元素的 children 数组中，然后设置自己的 parent 属性为 currentParent
  if (currentParent && !element.forbidden) {
    if (element.elseif || element.else) {
      processIfConditions(element, currentParent)
    } else {
      if (element.slotScope) {
        // scoped slot
        // keep it in the children list so that v-else(-if) conditions can
        // find it as the prev node.
        const name = element.slotTarget || '"default"'
          ; (currentParent.scopedSlots || (currentParent.scopedSlots = {}))[name] = element
      }
      currentParent.children.push(element)
      element.parent = currentParent
    }
  }

  // 设置自己的子元素
  // 将自己的所有非插槽的子元素设置到 element.children 数组中
  // final children cleanup
  // filter out scoped slots
  element.children = element.children.filter(c => !(c: any).slotScope)
  // remove trailing whitespace node again
  trimEndingWhitespace(element)

  // check pre state
  if (element.pre) {
    inVPre = false
  }
  if (platformIsPreTag(element.tag)) {
    inPre = false
  }
  // 分别为 element 执行 model、class、style 三个模块的 postTransform 方法
  // 但是 web 平台没有提供该方法
  // apply post-transforms
  for (let i = 0; i < postTransforms.length; i++) {
    postTransforms[i](element, options)
  }
}

```

### trimEndingWhitespace

> /src/compiler/parser/index.js

```javascript
/**
 * 删除元素中空白的文本节点，比如：<div> </div>，删除 div 元素中的空白节点，将其从元素的 children 属性中移出去
 */
function trimEndingWhitespace(el) {
  if (!inPre) {
    let lastNode
    while (
      (lastNode = el.children[el.children.length - 1]) &&
      lastNode.type === 3 &&
      lastNode.text === ' '
    ) {
      el.children.pop()
    }
  }
}

```

### processIfConditions

> /src/compiler/parser/index.js

```javascript
function processIfConditions(el, parent) {
  // 找到 parent.children 中的最后一个元素节点
  const prev = findPrevElement(parent.children)
  if (prev && prev.if) {
    addIfCondition(prev, {
      exp: el.elseif,
      block: el
    })
  } else if (process.env.NODE_ENV !== 'production') {
    warn(
      `v-${el.elseif ? ('else-if="' + el.elseif + '"') : 'else'} ` +
      `used on element <${el.tag}> without corresponding v-if.`,
      el.rawAttrsMap[el.elseif ? 'v-else-if' : 'v-else']
    )
  }
}

```

### findPrevElement

> /src/compiler/parser/index.js

```javascript
/**
 * 找到 children 中的最后一个元素节点 
 */
function findPrevElement(children: Array<any>): ASTElement | void {
  let i = children.length
  while (i--) {
    if (children[i].type === 1) {
      return children[i]
    } else {
      if (process.env.NODE_ENV !== 'production' && children[i].text !== ' ') {
        warn(
          `text "${children[i].text.trim()}" between v-if and v-else(-if) ` +
          `will be ignored.`,
          children[i]
        )
      }
      children.pop()
    }
  }
}

```

## 帮助

到这里编译器的解析部分就结束了，相信很多人看的是云里雾里的，即使多看几遍可能也没有那么清晰。

不要着急，这个很正常，编译器这块儿的代码量确实是比较大。但是内容本身其实不复杂，复杂的是它要处理东西实在是太多了，这才导致这部分的代码量巨大，相对应的，就会产生比较难的感觉。确实不简单，至少我觉得它是整个框架最复杂最难的地方了。

对照着视频和文章大家可以多看几遍，不明白的地方写一些示例代码辅助调试，编写详细的注释。还是那句话，书读百遍，其义自现。

阅读的过程中，大家需要抓住编译器解析部分的本质：将类 HTML 字符串模版解析成 AST 对象。

所以这么多代码都在做一件事情，就是解析字符串模版，将整个模版用 AST 对象来表示和记录。所以，大家阅读的时候，可以将解析过程中生成的 AST 对象记录下来，帮助阅读和理解，这样在读完以后不至于那么迷茫，也有助于大家理解。

这是我在阅读的时候的一个简单记录：

```javascript
const element = {
  type: 1,
  tag,
  attrsList: [{ name: attrName, value: attrVal, start, end }],
  attrsMap: { attrName: attrVal, },
  rawAttrsMap: { attrName: attrVal, type: checkbox },
  // v-if
  ifConditions: [{ exp, block }],
  // v-for
  for: iterator,
  alias: 别名,
  // :key
  key: xx,
  // ref
  ref: xx,
  refInFor: boolean,
  // 插槽
  slotTarget: slotName,
  slotTargetDynamic: boolean,
  slotScope: 作用域插槽的表达式,
  scopeSlot: {
    name: {
      slotTarget: slotName,
      slotTargetDynamic: boolean,
      children: {
        parent: container,
        otherProperty,
      }
    },
    slotScope: 作用域插槽的表达式,
  },
  slotName: xx,
  // 动态组件
  component: compName,
  inlineTemplate: boolean,
  // class
  staticClass: className,
  classBinding: xx,
  // style
  staticStyle: xx,
  styleBinding: xx,
  // attr
  hasBindings: boolean,
  nativeEvents: {同 evetns},
  events: {
    name: [{ value, dynamic, start, end, modifiers }]
  },
  props: [{ name, value, dynamic, start, end }],
  dynamicAttrs: [同 attrs],
  attrs: [{ name, value, dynamic, start, end }],
  directives: [{ name, rawName, value, arg, isDynamicArg, modifiers, start, end }],
  // v-pre
  pre: true,
  // v-once
  once: true,
  parent,
  children: [],
  plain: boolean,
}

```

## 总结

* **面试官 问**：简单说一下 Vue 的编译器都做了什么？

  **答**：
  
  Vue 的编译器做了三件事情：
  
  * 将组件的 html 模版解析成 AST 对象
    
  * 优化，遍历 AST，为每个节点做静态标记，标记其是否为静态节点，然后进一步标记出静态根节点，这样在后续更新的过程中就可以跳过这些静态节点了；标记静态根用于生成渲染函数阶段，生成静态根节点的渲染函数
    
  * 从 AST 生成运行时的渲染函数，即大家说的 render，其实还有一个，就是 staticRenderFns 数组，里面存放了所有的静态节点的渲染函数
  
<hr />

* **面试官 问**：详细说一说编译器的解析过程，它是怎么将 html 字符串模版变成 AST 对象的？

  **答**：
  
  * 遍历 HTML 模版字符串，通过正则表达式匹配 "<"
  
  * 跳过某些不需要处理的标签，比如：注释标签、条件注释标签、Doctype。
  
    > 备注：整个解析过程的核心是处理开始标签和结束标签
  
  * 解析开始标签
  
    * 得到一个对象，包括 标签名（tagName）、所有的属性（attrs）、标签在 html 模版字符串中的索引位置
    
    * 进一步处理上一步得到的 attrs 属性，将其变成 [{ name: attrName, value: attrVal, start: xx, end: xx }, ...] 的形式
    
    * 通过标签名、属性对象和当前元素的父元素生成 AST 对象，其实就是一个 普通的 JS 对象，通过 key、value 的形式记录了该元素的一些信息
    
    * 接下来进一步处理开始标签上的一些指令，比如 v-pre、v-for、v-if、v-once，并将处理结果放到 AST 对象上
    
    * 处理结束将 ast 对象存放到 stack 数组
    
    * 处理完成后会截断 html 字符串，将已经处理掉的字符串截掉
    
  * 解析闭合标签
  
    * 如果匹配到结束标签，就从 stack 数组中拿出最后一个元素，它和当前匹配到的结束标签是一对。
    
    * 再次处理开始标签上的属性，这些属性和前面处理的不一样，比如：key、ref、scopedSlot、样式等，并将处理结果放到元素的 AST 对象上
    
      > **备注** 视频中说这块儿有误，回头看了下，没有问题，不需要改，确实是这样
    
    * 然后将当前元素和父元素产生联系，给当前元素的 ast 对象设置 parent 属性，然后将自己放到父元素的 ast 对象的 children 数组中
    
  * 最后遍历完整个 html 模版字符串以后，返回 ast 对象
  
## 链接

* [配套视频，微信公众号回复](https://gitee.com/liyongning/typora-image-bed/raw/master/202202171742614.jpg)："精通 Vue 技术栈源码原理视频版" 获取
* [精通 Vue 技术栈源码原理 专栏](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzA3NTk4NjQ1OQ==&action=getalbum&album_id=2273541436891693065#wechat_redirect)
* [github 仓库 liyongning/Vue](https://github.com/liyongning/Vue) 欢迎 Star



感谢各位的：**关注**、**点赞**、**收藏**和**评论**，我们下期见。

***

**当学习成为了习惯，知识也就变成了常识。** 感谢各位的 **关注**、 **点赞**、**收藏**和**评论**。

新视频和文章会第一时间在微信公众号发送，欢迎关注：[李永宁lyn](https://gitee.com/liyongning/typora-image-bed/raw/master/202202171742614.jpg)

文章已收录到 [github 仓库 liyongning/blog](https://github.com/liyongning/blog)，欢迎 Watch 和 Star。