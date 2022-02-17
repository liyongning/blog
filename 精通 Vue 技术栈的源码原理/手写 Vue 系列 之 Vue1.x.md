# 手写 Vue 系列 之 Vue1.x

**当学习成为了习惯，知识也就变成了常识。** 感谢各位的 **关注**、**点赞**、**收藏**和**评论**。

新视频和文章会第一时间在微信公众号发送，欢迎关注：[李永宁lyn](https://gitee.com/liyongning/typora-image-bed/raw/master/202202171742614.jpg)

文章已收录到 [github 仓库 liyongning/blog](https://github.com/liyongning/blog)，欢迎 Watch 和 Star。

## 封面

![](https://gitee.com/liyongning/typora-image-bed/raw/master/202203092004749.png)

## 前言

前面我们用 12 篇文章详细讲解了 Vue2 的框架源码。接下来我们就开始手写 Vue 系列，写一个自己的 Vue 框架，用最简单的代码实现 Vue 的核心功能，进一步理解 Vue 核心原理。

## 为什么要手写框架

有人会有疑问：我已经详细阅读过框架源码了，甚至不止两三遍，这难道还不够吗？我自认为对框架的源码已经很熟悉了，我觉得没必要再手写。

**有没有必要手写框架** 这个事情，和 **有没有必要阅读框架源码** 的答案一样。看你的出发点是什么。

### 读源码

如果你是抱以学习的态度，那不用说，阅读框架源码肯定是有必要的。

大家都明白，平时的业务开发中，你身边人的水平可能都跟你差不多，所以你在业务中基本是看不到太多的优秀编码和思想。

而一个框架所包含的优秀设计和最佳实践就很多了，在阅读的时候有太多让你恍然大悟和惊艳的地方。即使你觉得自己现在段位不够，可能看不到那么多，但是源码对你的影响是潜移默化的。看多了优秀的代码，在你自己平时的编码中会不自觉的应用你学到的这些优秀编码方式。更何况 Vue 的大部分代码都是尤大自己写的，代码质量那是毋庸置疑的。

### 手写框架

至于 **手写框架是否有必要** ？只要你读了框架源码，就必须自己手写一个。理由很简单，你阅读框架源码的目的是学习，你说你对源码已经非常熟了，你说你都学会了，那怎么检验？检验的方式也很简单，把你学到的东西向外输出，分三个阶段：

1. 写技术博客、画思维导图（掌握 30%）

2. 给他人分享，比如组内分享、录视频都行（掌握 60%）

3. 手写框架，造轮子是检验你学习成果最好的方式（掌握 90%）

有没有发现前两阶段都是在讲他人的东西，你说你学到了，确实，你能向外输出，学你肯定是学到了，但是学到了多少呢？我觉得差不多是 60%，举个例子：

别人问你 Vue 的响应式原理是什么？经过前两个阶段的输出，你可能说的头头是道，比如 Object.defineProperty、getter、setter、依赖收集、依赖通知 watcher 更新等等。但是这整个过程你能否写出来呢？如果你第一次写，大概率是不行的，实现的时候会发现，根本不像你说的那么简单，要考虑东西远不止你说的那些。如果不信大家可以试试，检验一下。

要知道，造轮子的过程其实就是你应用的过程，只有你真的写出来了，你才算是真的学到了。如果只看不写，基本上可以算是进阶版的 **只看不练**。

所以，检验你是否真的学会并深入理解某个框架的实现原理，模仿 **造轮子** 是最好的检验方式。

## 手写 Vue1.x

在开始之前，我们先做好准备工作，在自己的工作目录下，新建我们的源码目录，比如：

```shell
mkdir lyn-vue && cd lyn-vue
```
这里我不想额外安装和配置打包工具，太麻烦了，采用现代浏览器原生支持的 ESM 的方式，所以大家需要在本地装一个 serve，起一个服务器。vite 就是这个原理，只不过它的服务端是自己实现的，因为它需要针对 import 的不同资源做相应的处理，比如解析 import 请求的是 node_modules 还是 用户自己的模块，亦或者是 TS 模块的转译等等。

```shell
npm i serve -g
```
安装好之后，在 `lyn-vue` 目录下执行 `serve` 命令，会在本地起一个服务器，接下来就进入编码阶段。

### 目标

下面的示例代码就是今天的目标，用我们自己手写的 Vue 框架把这个示例跑起来。

我们需要实现以下能力：

* 数据响应式拦截

  * 原始值
  
  * 普通对象
  
  * 数组
  
* 数据响应式更新

  * 依赖收集，Dep
  
  * 依赖通知 Watcher 更新
  
  * 编译器，compiler
  
* methods + 事件 + 数据响应式更新

* v-bind 指令

* v-model 双向绑定

  * input 输入框
  
  * checkbox
  
  * select
  
> /vue1.0.html

```html
<!DOCTYPE html>
<html lang="en">

<head>
  <meta charset="UTF-8">
  <title>Lyn Vue1.0</title>
</head>

<body>
  <div id="app">
    <h3>数据响应式更新 原理</h3>
    <div>{{ t }}</div>
    <div>{{ t1 }}</div>
    <div>{{ arr }}</div>
    <h3>methods + 事件 + 数据响应式更新 原理</h3>
    <div>
      <p>{{ counter }}</p>
      <button v-on:click="handleAdd"> Add </button>
      <button v-on:click="handleMinus"> Minus </button>
    </div>
    <h3>v-bind</h3>
    <span v-bind:title="title">右键审查元素查看我的 title 属性</span>
    <h3>v-model 原理</h3>
    <div>
      <input type="text" v-model="inputVal" />
      <div>{{ inputVal }}</div>
    </div>
    <div>
      <input type="checkbox" v-model="isChecked">
      <div>{{ isChecked }}</div>
    </div>
    <div>
      <select v-model="selectValue">
        <option value="1">1</option>
        <option value="2">2</option>
        <option value="3">3</option>
      </select>
      <div>{{ selectValue }}</div>
    </div>
  </div>
  <script type="module">
    import Vue from './src/index.js'
    const ins = new Vue({
      el: '#app',
      data() {
        return {
          // 原始值和对象的响应式原理
          t: 't value',
          t1: {
            tt1: 'tt1 value'
          },
          // 数组的响应式原理
          arr: [1, 2, 3],
          // 响应式更新
          counter: 0,
          // v-bind
          title: '看我',
          // v-model
          inputVal: 'test',
          isChecked: true,
          selectValue: 2
        }
      },
      // methods + 事件 + 数据响应式更新 原理
      methods: {
        handleAdd() {
          this.counter++
        },
        handleMinus() {
          this.counter--
        }
      },
    })
    // 数据响应式拦截
    setTimeout(() => {
      console.log('********** 属性值为原始值时的 getter、setter ************')
      console.log(ins.t)
      ins.t = 'change t value'
      console.log(ins.t)
    }, 1000)

    setTimeout(() => {
      console.log('********** 属性的新值为对象的情况 ************')
      ins.t = {
        tt: 'tt value'
      }
      console.log(ins.t.tt)
    }, 2000)

    setTimeout(() => {
      console.log('********** 验证对深层属性的 getter、setter 拦截 ************')
      ins.t1.tt1 = 'change tt1 value'
      console.log(ins.t1.tt1)
    }, 3000)

    setTimeout(() => {
      console.log('********** 将值为对象的属性更新为原始值 ************')
      console.log(ins.t1)
      ins.t1 = 't1 value'
      console.log(ins.t1)
    }, 4000)

    setTimeout(() => {
      console.log('********** 数组操作方法的拦截 ************')
      console.log(ins.arr)
      ins.arr.push(4)
      console.log(ins.arr)
    }, 5000)
  </script>
</body>

</html>

```

### 数据响应式拦截

#### Vue 构造函数

> /src/index.js

```javascript
/**
 * Vue 构造函数
 * @param {*} options new Vue(options) 时传递的配置对象
 */
export default function Vue(options) {
  this._init(options)
}

```

#### this._init

> /src/index.js

```javascript
/**
 * 初始化配置对象
 * @param {*} options 
 */
Vue.prototype._init = function (options) {
  // 将 options 配置挂载到 Vue 实例上
  this.$options = options
  // 初始化 options.data
  // 代理 data 对象上的各个属性到 Vue 实例
  // 给 data 对象上的各个属性设置响应式能力
  initData(this)
}

```

#### initData

> /src/initData.js

```javascript
/**
 * 1、初始化 options.data
 * 2、代理 data 对象上的各个属性到 Vue 实例
 * 3、给 data 对象上的各个属性设置响应式能力
 * @param {*} vm 
 */
export default function initData(vm) {
  // 获取 data 选项
  let { data } = vm.$options
  // 设置 vm._data 选项，保证它的值肯定是一个对象
  if (!data) {
    vm._data = {}
  } else {
    vm._data = typeof data === 'function' ? data() : data
  }
  // 代理，将 data 对象上的的各个属性代理到 Vue 实例上，支持 通过 this.xx 的方式访问
  for (let key in vm._data) {
    proxy(vm, '_data', key)
  }
  // 设置响应式
  observe(vm._data)
}

```

#### proxy

> /src/utils.js

```javascript
/**
 * 将 key 代理到 target 上，
 * 比如 代理 this._data.xx 为 this.xx
 * @param {*} target 目标对象，比如 vm
 * @param {*} sourceKey 原始 key，比如 _data
 * @param {*} key 代理的原始对象上的指定属性，比如 _data.xx
 */
export function proxy(target, sourceKey, key) {
  Object.defineProperty(target, key, {
    // target.key 的读取操作实际上返回的是 target.sourceKey.key
    get() {
      return target[sourceKey][key]
    },
    // target.key 的赋值操作实际上是 target.sourceKey.key = newV
    set(newV) {
      target[sourceKey][key] = newV
    }
  })
}

```

#### observe

> /src/observe.js

```javascript
/**
 * 通过 Observer 类为对象设置响应式能力
 * @returns Observer 实例
 */
export default function observe(value) {
  // 避免无限递归
  // 当 value 不是对象直接结束递归
  if (typeof value !== 'object') return

  // value.__ob__ 是 Observer 实例
  // 如果 value.__ob__ 属性已经存在，说明 value 对象已经具备响应式能力，直接返回已有的响应式对象
  if (value.__ob__) return value.__ob__

  // 返回 Observer 实例
  return new Observer(value)
}

```

#### Observer

> /src/observer.js

```javascript
/**
 * 为普通对象或者数组设置响应式的入口 
 */
export default function Observer(value) {
  // 为对象设置 __ob__ 属性，值为 this，标识当前对象已经是一个响应式对象了
  Object.defineProperty(value, '__ob__', {
    value: this,
    // 设置为 false，禁止被枚举，
    // 1、可以在递归设置数据响应式的时候跳过 __ob__ 
    // 2、将响应式对象字符串化时也不限显示 __ob__ 对象
    enumerable: false,
    writable: true,
    configurable: true
  })

  if (Array.isArray(value)) {
    // 数组响应式
    protoArgument(value)
    this.observeArray(value)
  } else {
    // 对象响应式
    this.walk(value)
  }
}

/**
 * 遍历对象的每个属性，为这些属性设置 getter、setter 拦截
 */
Observer.prototype.walk = function (obj) {
  for (let key in obj) {
    defineReactive(obj, key, obj[key])
  }
}

// 遍历数组的每个元素，为每个元素设置响应式
// 其实这里是为了处理元素为对象的情况，以达到 this.arr[idx].xx 是响应式的目的
Observer.prototype.observeArray = function (arr) {
  for (let item of arr) {
    observe(item)
  }
}

```

#### defineReactive

> /src/defineReactive.js

```javascript
/**
 * 通过 Object.defineProperty 为 obj.key 设置 getter、setter 拦截
 */
export default function defineReactive(obj, key, val) {
  // 递归调用 observe，处理 val 仍然为对象的情况
  observe(val)

  Object.defineProperty(obj, key, {
    // 当发现 obj.key 的读取行为时，会被 get 拦截
    get() {
      console.log(`getter: key = ${key}`)
      return val
    },
    // 当发生 obj.key = xx 的赋值行为时，会被 set 拦截
    set(newV) {
      console.log(`setter: ${key} = ${newV}`)
      if (newV === val) return
      val = newV
      // 对新值进行响应式处理，这里针对的是新值为非原始值的情况，比如 val 为对象、数组
      observe(val)
    }
  })
}

```

#### protoArgument

> /src/protoArgument.js

```javascript
/**
 * 通过拦截数组的七个方法来实现
 */

// 数组默认原型对象
const arrayProto = Array.prototype
// 以数组默认原型对象为原型创建一个新的对象
const arrayMethods = Object.create(arrayProto)
// 被 patch 的七个方法，通过拦截这七个方法来实现数组响应式
// 为什么是这七个方法？因为只有这七个方法是能更改数组本身的，像 cancat 这些方法都是会返回一个新的数组，不会改动数组本身
const methodsToPatch = ['push', 'pop', 'unshift', 'shift', 'splice', 'sort', 'reverse']

// 遍历 methodsToPatch
methodsToPatch.forEach(method => {
  // 拦截数组的七个方法，先完成本职工作，再额外完成响应式的工作
  Object.defineProperty(arrayMethods, method, {
    value: function(...args) {
      // 完成方法的本职工作，比如 this.arr.push(xx)
      const ret = arrayProto[method].apply(this, args)
      // 将来接着实现响应式相关的能力
      console.log('array reactive')
      return ret
    },
    configurable: true,
    writable: true,
    enumerable: true
  })
})

/**
 * 覆盖数组（arr）的原型对象
 * @param {*} arr 
 */
export default function protoArgument(arr) {
  arr.__proto__ = arrayMethods
}

```

#### 效果

能达到如下效果，则表示数据响应式拦截功能完成。即能跑通目标中示例代码的 “数据响应式拦截” 部分的代码（最后的那堆 setTimeout）。

动图地址: https://gitee.com/liyongning/typora-image-bed/raw/master/202203092000920.image

![响应式原理.gif](https://gitee.com/liyongning/typora-image-bed/raw/master/202203092000920.image)

### 数据响应式更新

现在已经能拦截到对数据的获取和更新，接下来就可以在拦截数据的地方增加一些 “能力”，以完成 **数据响应式更新** 的功能。

增加的这些能力其实就是大家耳熟能详的东西：在 getter 中进行依赖收集，setter 中依赖通知 watcher 更新。

Vue1.x 中响应式数据对象的所有属性（key）和 dep 是一一对应对应关系，一个 key 对应一个 dep；响应式数据在页面中每引用一次就会产生一个 watcher，所以在 Vue1.0 中 dep 和 watcher 是一对多的关系。

#### 依赖收集

##### Dep

> /src/dep.js

```javascript
/**
 * Dep
 * Vue1.0 中 key 和 Dep 是一一对应关系，举例来说：
 * new Vue({
 *   data() {
 *     return {
 *       t1: xx,
 *       t2: {
 *         tt2: xx
 *       },
 *       arr: [1, 2, 3, { t3: xx }]
 *     }
 *   }
 * })
 * data 函数 return 回来的对象是一个 dep
 * 对象中的 key => t1、t2、tt2、arr、t3 都分别对应一个 dep
 */
export default function Dep() {
  // 存储当前 dep 实例收集的所有 watcher
  this.watchers = []
}

// Dep.target 是一个静态属性，值为 null 或者 watcher 实例
// 在实例化 Watcher 时进行赋值，待依赖收集完成后在 Watcher 中又重新赋值为 null
Dep.target = null

/**
 * 收集 watcher
 * 在发生读取操作时（vm.xx) && 并且 Dep.target 不为 null 时进行依赖收集
 */
Dep.prototype.depend = function () {
  // 防止 Watcher 实例被重复收集
  if (this.watchers.includes(Dep.target)) return
  // 收集 Watcher 实例
  this.watchers.push(Dep.target)
}

/**
 * dep 通知自己收集的所有 watcher 执行更新函数
 */
Dep.prototype.notify = function () {
  for (let watcher of this.watchers) {
    watcher.update()
  }
}

```

##### Watcher

> /src/watcher.js

```javascript
import Dep from "./dep.js"

/**
 * @param {*} cb 回调函数，负责更新 DOM 的回调函数
 */
export default function Watcher(cb) {
  // 备份 cb 函数
  this._cb = cb
  // 赋值 Dep.target
  Dep.target = this
  // 执行 cb 函数，cb 函数中会发生 vm.xx 的属性读取，进行依赖收集
  cb()
  // 依赖收集完成，Dep.target 重新赋值为 null，防止重复收集
  Dep.target = null
}

/**
 * 响应式数据更新时，dep 通知 watcher 执行 update 方法，
 * 让 update 方法执行 this._cb 函数更新 DOM
 */
Watcher.prototype.update = function () {
  this._cb()
}

```

##### Observer

改造 Observer 构造函数，在 value.__ob__ 对象上设置一个 dep 实例。这个 dep 是对象本身的 dep，方便在更新对象本身时使用，比如：数组依赖通知更新时就会用到。

> /src/observer.js

```javascript
/**
 * 为普通对象或者数组设置响应式的入口
 */
export default function Observer(value) {
  // 为对象本身设置一个 dep，方便在更新对象本身时使用，比如 数组通知依赖更新时就会用到
  this.dep = new Dep()  
  // ... 省略已有内容
}

```

##### defineReactive

改造 defineReactive 方法，增加依赖收集和依赖通知更新的代码

> /src/defineReactive.js

```javascript
/**
 * 通过 Object.defineProperty 为 obj.key 设置 getter、setter 拦截
 * getter 时收集依赖
 * setter 时依赖通过 watcher 更新
 */
export default function defineReactive(obj, key, val) {
  // 递归调用 observe，处理 val 仍然为对象的情况
  const childOb = observe(val)

  const dep = new Dep()

  Object.defineProperty(obj, key, {
    // 当发现 obj.key 的读取行为时，会被 get 拦截
    get() {
      // 读取数据时 && Dep.target 不为 null，则进行依赖收集
      if (Dep.target) {
        dep.depend()
        // 如果存在子 ob，则顺道一块儿完成依赖收集
        if (childOb) {
          childOb.dep.depend()
        }
      }
      console.log(`getter: key = ${key}`)
      return val
    },
    // 当发生 obj.key = xx 的赋值行为时，会被 set 拦截
    set(newV) {
      console.log(`setter: ${key} = ${newV}`)
      if (newV === val) return
      val = newV
      // 对新值进行响应式处理，这里针对的是新值为非原始值的情况，比如 val 为对象、数组
      observe(val)
      // 数据更新，让 dep 通知自己收集的所有 watcher 执行 update 方法
      dep.notify()
    }
  })
}

```

##### protoArgument

改造七个数组方法的 patch 补丁，当数组新增元素时，对新元素进行响应式处理和依赖通知更新。

> /src/protoArgument.js

```javascript
/**
 * 通过拦截数组的七个方法来实现
 */

// 数组默认原型对象
const arrayProto = Array.prototype
// 以数组默认原型对象为原型创建一个新的对象
const arrayMethods = Object.create(arrayProto)
// 被 patch 的七个方法，通过拦截这七个方法来实现数组响应式
// 为什么是这七个方法？因为只有这七个方法是能更改数组本身的，像 cancat 这些方法都是会返回一个新的数组，不会改动数组本身
const methodsToPatch = ['push', 'pop', 'unshift', 'shift', 'splice', 'sort', 'reverse']

// 遍历 methodsToPatch
methodsToPatch.forEach(method => {
  // 拦截数组的七个方法，先完成本职工作，再额外完成响应式的工作
  Object.defineProperty(arrayMethods, method, {
    value: function(...args) {
      // 完成方法的本职工作，比如 this.arr.push(xx)
      const ret = arrayProto[method].apply(this, args)
      // 将来接着实现响应式相关的能力
      console.log('array reactive')
      // 新增的元素列表
      let inserted = []
      switch(method) {
        case 'push':
        case 'unshift':
          inserted = args
          break;
        case 'splice':
          // this.arr.splice(idx, num, x, x, x)
          inserted = args.slice(2)
          break;
      }
      // 如果数组有新增的元素，则对新增的元素进行响应式处理
      inserted.length && this.__ob__.observeArray(inserted)
      // 依赖通知更新
      this.__ob__.dep.notify()
      return ret
    },
    configurable: true,
    writable: true,
    enumerable: true
  })
})

/**
 * 覆盖数组（arr）的原型对象
 * @param {*} arr 
 */
export default function protoArgument(arr) {
  arr.__proto__ = arrayMethods
}

```

到这里依赖收集就全部完成了。但是你会发现页面还是没有发生任何变化，响应式数据在页面没有得到渲染，数据更新时页面更是没有任何变化。这是为什么？还需要做什么事情吗？

其实回顾依赖收集的代码会发现有一个被我们遗漏的地方，大家有没有发现 Watcher 构造函数似乎从来没有被实例化过，那也就是说依赖收集其实从来没有被触发过，因为只有实例化 Watcher 时 Dep.target 才会被赋值。

那么问题就来了，Watcher 应该在什么什么时候被实例化呢？大家可能没有看过 Vue1 的源码，但是 Vue2 的源码前面带大家看过了，仔细回想一下，什么时候会去实例化 Watcher。

答案是 mountComponent，也就是挂载阶段，初始化完成后执行 \$mount，$mount 调用 mountComponent，mountComponent 方法中有一步就是在实例化 Watcher。如果这块儿有遗忘，大家可以再去翻看一下这部分的源码。

所以接下来我们要实现的就是编译器了，也就是 \$mount 方法。

#### 编译器

这部分利用 DOM 操作实现了一个简版的编译器。从中你可以看到节点树的编译过程，明白文本节点、v-on:click、v-bind、v-model 指令的实现原理。

##### $mount

> /src/index.js

```javascript
Vue.prototype._init = function (options) {
  ... 省略
  
  // 如果存在 el 配置项，则调用 $mount 方法编译模版
  if (this.$options.el) {
    this.$mount()
  }
}

Vue.prototype.$mount = function () {
  mount(this)
}

```

##### mount

> /src/compiler/index.js

```javascript
/**
 * 编译器
 */
export default function mount(vm) {
  // 获取 el 选择器所表示的元素
  let el = document.querySelector(vm.$options.el)

  // 编译节点
  compileNode(Array.from(el.childNodes), vm)
}

```

##### compileNode

> /src/compiler/compileNode.js

```javascript
/**
 * 递归编译整棵节点树
 * @param {*} nodes 节点
 * @param {*} vm Vue 实例
 */
export default function compileNode(nodes, vm) {
  // 循环遍历当前节点的所有子节点
  for (let i = 0, len = nodes.length; i < len; i++) {
    const node = nodes[i]
    if (node.nodeType === 1) { // 元素节点
      // 编译元素上的属性节点
      compileAttribute(node, vm)
      // 递归编译子节点
      compileNode(Array.from(node.childNodes), vm)
    } else if (node.nodeType === 3 && node.textContent.match(/{{(.*)}}/)) {
      // 编译文本节点
      compileTextNode(node, vm)
    }
  }
}

```

##### compileTextNode

文本节点响应式更新的原理

> /src/compiler/compileTextNode.js

```javascript
/**
 * 编译文本节点
 * @param {*} node 节点
 * @param {*} vm Vue 实例
 */
export default function compileTextNode(node, vm) {
  // <span>{{ key }}</span>
  const key = RegExp.$1.trim()
  // 当响应式数据 key 更新时，dep 通知 watcher 执行 update 函数，cb 会被调用
  function cb() {
    node.textContent = JSON.stringify(vm[key])
  }
  // 实例化 Watcher，执行 cb，触发 getter，进行依赖收集
  new Watcher(cb)
}

```

##### compileAttribute

v-on:click、v-bind 和 v-model 指令的原理

> /src/compiler/compileAttribute.js

```javascript
/**
 * 编译属性节点
 * @param {*} node 节点
 * @param {*} vm Vue 实例
 */
export default function compileAttribute(node, vm) {
  // 将类数组格式的属性节点转换为数组
  const attrs = Array.from(node.attributes)
  // 遍历属性数组
  for (let attr of attrs) {
    // 属性名称、属性值
    const { name, value } = attr
    if (name.match(/v-on:click/)) {
      // 编译 v-on:click 指令
      compileVOnClick(node, value, vm)
    } else if (name.match(/v-bind:(.*)/)) {
      // v-bind
      compileVBind(node, value, vm)
    } else if (name.match(/v-model/)) {
      // v-model
      compileVModel(node, value, vm)
    }
  }
}

```

###### compileVOnClick

> /src/compiler/compileAttribute.js

```javascript
/**
 * 编译 v-on:click 指令
 * @param {*} node 节点
 * @param {*} method 方法名
 * @param {*} vm Vue 实例
 */
function compileVOnClick(node, method, vm) {
  // 给节点添加一个 click 事件，回调函数是对应的 method
  node.addEventListener('click', function (...args) {
    // 给 method 绑定 this 上下文
    vm.$options.methods[method].apply(vm, args)
  })
}

```

###### compileVBind

> /src/compiler/compileAttribute.js

```javascript
/**
 * 编译 v-bind 指令
 * @param {*} node 节点
 * @param {*} attrValue 属性值
 * @param {*} vm Vue 实例
 */
function compileVBind(node, attrValue, vm) {
  // 属性名称
  const attrName = RegExp.$1
  // 移除模版中的 v-bind 属性
  node.removeAttribute(`v-bind:${attrName}`)
  // 当属性值发生变化时，重新执行回调函数
  function cb() {
    node.setAttribute(attrName, vm[attrValue])
  }
  // 实例化 Watcher，当属性值发生变化时，dep 通知 watcher 执行 update 方法，cb 被执行，重新更新属性
  new Watcher(cb)
}

```

###### compileVModel

> /src/compiler/compileAttribute.js

```javascript
/**
 * 编译 v-model 指令
 * @param {*} node 节点 
 * @param {*} key v-model 的属性值
 * @param {*} vm Vue 实例
 */
function compileVModel(node, key, vm) {
  // 节点标签名、类型
  let { tagName, type } = node
  // 标签名转换为小写
  tagName = tagName.toLowerCase()
  if (tagName === 'input' && type === 'text') {
    // <input type="text" v-model="inputVal" />

    // 设置 input 输入框的初始值
    node.value = vm[key]
    // 给节点添加 input 事件，当事件发生时更改响应式数据
    node.addEventListener('input', function () {
      vm[key] = node.value
    })
  } else if (tagName === 'input' && type === 'checkbox') {
    // <input type="checkbox" v-model="isChecked" />

    // 设置选择框的初始状态
    node.checked = vm[key]
    // 给节点添加 change 事件，当事件发生时更改响应式数据
    node.addEventListener('change', function () {
      vm[key] = node.checked
    })
  } else if (tagName === 'select') {
    // <select v-model="selectedValue"></select>

    // 设置下拉框初始选中的选项
    node.value = vm[key]
    // 添加 change 事件，当事件发生时更改响应式数据
    node.addEventListener('change', function () {
      vm[key] = node.value
    })
  }
}

```

## 总结

到这里，一个简版的 Vue1.x 就实现完了。回顾一下，我们实现了如下功能：

* 数据响应式拦截

  * 普通对象
  
  * 数组

* 数据响应式更新

  * 依赖收集
  
    * Dep
    
    * Watcher
  
  * 编译器
  
    * 文本节点
    
    * v-on:click
    
    * v-bind
    
    * v-model
    
    

**目标** 中示例代码的执行结果如下：

动图地址：https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fcceda69f08a4d0a8b4f9c1e96032ad6~tplv-k3u1fbpfcp-watermark.image

![May-26-2021 09-12-48.gif](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fcceda69f08a4d0a8b4f9c1e96032ad6~tplv-k3u1fbpfcp-watermark.image)

**面试官 问**：Vue1.x 数据响应式是如何实现的？

**答**：

Vue 数据响应式的核心原理是 `Object.defineProperty`。

通过递归遍历整个 data 对象，为对象中的每个 key 设置一个 getter、setter。如果 key 为数组，则走数组响应式的流程。

数组响应式是通过 Object.defineProperty 去拦截数组的七个方法实现的。首先增强了那个七个方法，在完成方法本职工作的基础上增加了依赖通知更新的能力，而且如果有新增数据，则新数据也会被进行响应式处理。

数据响应式更新的能力是通过数据响应式拦截结合 Dep、Watcher、编译器来实现的。

当做完数据初始化工作以后（即响应式拦截），就进入挂载阶段，开始编译整棵 DOM 树，编译过程中 **碰到响应式数据，实例化 Watcher**，这时会发生数据读取操作，触发 getter，进行依赖收集，将 Watcher 实例放到当前响应式属性对应的 dep 中。

待将来响应式数据更新时，触发 setter，然后出发 dep 通知自己收集的所有 Watcher 实例去执行 update 方法，触发回调函数的执行，从而更新 DOM。

以上 Vue1.x 的整个响应式原理的实现。

**面试官 问**：你如何评价 Vue1.x 响应式原理的设计？

**答**：

Vue1.x 其实是尤大为了解决自己工作上的痛点而实现的，当时他觉得各种 DOM 操作太繁琐了，初始化时需要通过 DOM 操作将数据设置到节点上，还要监听 DOM 操作，当 DOM 更新时，更新相应的数据。于是他就想着能不能把这个过程自动化，这就产生了 Vue1.x。

这么一想，Vue1.x 的实现其实就很合理了，确实达到了预期的目标。通过 Object.defineProperty 拦截数据的读取和设置，页面初次渲染时，通过编译器编译整棵 DOM 树，给 DOM 节点设置初始值，当 DOM 节点更新时又自动更新了响应式数据，或者响应式数据更新时，通过 Watcher 自动更新对应的 DOM 节点。

这个时候的 Vue 在完成中小型 Web 系统是没有任何问题的。而且相比于 Vue 2.x 性能会更好，因为响应式数据更新时，Watcher 可以直接更新对应的 DOM 节点，没有 2.x 的 VNode 开销和 Diff 过程。

但是大型 Web 系统就搞不定了，理由也很简单，也是因为它的设计。因为 Vue1.x 中 Watcher 和 模版中响应式数据是 一一对应 关系，也就是说页面中每引用一次响应式数据，就会产生一个 Watcher。在大型系统中，一个页面的数据量可能是非常大的，那就会产生大量的 Watcher，占用大量资源，导致性能下降。

所以一句话总结就是，Vue1.x 在中小型系统中性能会很好，定向更新 DOM 节点，但是大型系统由于 Watcher 太多，导致资源占用过多，性能下降。

于是 Vue2.x 中通过引入 VNode 和 Diff 的来解决这个问题，具体的实现原理将在下一篇文章 **手写 Vue 系列之 Vue2.x** 中去介绍。

## 预告

接下来的文章，会将本篇文章中实现的 Vue1.x 升级为 Vue2.x，引入 Vnode、diff 算法来解决 Vue1.x 的性能瓶颈。

另外会额外实现一些其它的核心原理，比如 computed、异步更新队列、child component、插槽 等。

## 链接

* [配套视频，微信公众号回复](https://gitee.com/liyongning/typora-image-bed/raw/master/202202171742614.jpg)："精通 Vue 技术栈源码原理视频版" 获取
* [精通 Vue 技术栈源码原理 专栏](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzA3NTk4NjQ1OQ==&action=getalbum&album_id=2273541436891693065#wechat_redirect)
* [github 仓库 liyongning/Vue](https://github.com/liyongning/Vue) 欢迎 Star



感谢各位的：**关注**、**点赞**、**收藏**和**评论**，我们下期见。

***

**当学习成为了习惯，知识也就变成了常识。** 感谢各位的 **关注**、 **点赞**、**收藏**和**评论**。

新视频和文章会第一时间在微信公众号发送，欢迎关注：[李永宁lyn](https://gitee.com/liyongning/typora-image-bed/raw/master/202202171742614.jpg)

文章已收录到 [github 仓库 liyongning/blog](https://github.com/liyongning/blog)，欢迎 Watch 和 Star。