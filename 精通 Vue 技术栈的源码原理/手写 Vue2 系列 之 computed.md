# 手写 Vue2 系列 之 computed

**当学习成为了习惯，知识也就变成了常识。** 感谢各位的 **关注**、**点赞**、**收藏**和**评论**。

新视频和文章会第一时间在微信公众号发送，欢迎关注：[李永宁lyn](https://gitee.com/liyongning/typora-image-bed/raw/master/202202171742614.jpg)

文章已收录到 [github 仓库 liyongning/blog](https://github.com/liyongning/blog)，欢迎 Watch 和 Star。

## 封面

![](https://gitee.com/liyongning/typora-image-bed/raw/master/202203161832220.png)

## 前言

上一篇文章 [手写 Vue2 系列 之 patch —— diff](https://mp.weixin.qq.com/s?__biz=MzA3NTk4NjQ1OQ==&mid=2247485338&idx=1&sn=c063df25b963f5db797e2a34402ec7ac&chksm=9f6964eea81eedf85e39ba0e32e579c2b4613fb63cabbb141e1d06b0155dd63fd363c08c3b43#rd) 实现了 DOM diff 过程，完成页面响应式数据的更新。

## 目标

本篇的目标是实现 computed 计算属性，完成模版中计算属性的展示。涉及的知识点：

* 计算属性的本质

* 计算属性的缓存原理

## 实现

接下来就开始实现 computed 计算属性，。

### _init

> /src/index.js

```javascript
/**
 * 初始化配置对象
 * @param {*} options 
 */
Vue.prototype._init = function (options) {
  // ...
  // 初始化 options.data
  // 代理 data 对象上的各个属性到 Vue 实例
  // 给 data 对象上的各个属性设置响应式能力
  initData(this)
  // 初始化 computed 选项，并将计算属性代理到 Vue 实例上
  // 结合 watcher 实现缓存
  initComputed(this)
  // 安装运行时的渲染工具函数
  renderHelper(this)
  // ...
}

```

### initComputed

> /src/initComputed.js

```javascript
/**
 * 初始化 computed 配置项
 * 为每一项实例化一个 Watcher，并将其 computed 属性代理到 Vue 实例上
 * 结合 watcher.dirty 和 watcher.evalute 实现 computed 缓存
 * @param {*} vm Vue 实例
 */
export default function initComputed(vm) {
  // 获取 computed 配置项
  const computed = vm.$options.computed
  // 记录 watcher
  const watcher = vm._watcher = Object.create(null)
  // 遍历 computed 对象
  for (let key in computed) {
    // 实例化 Watcher，回调函数默认懒执行
    watcher[key] = new Watcher(computed[key], { lazy: true }, vm)
    // 将 computed 的属性 key 代理到 Vue 实例上
    defineComputed(vm, key)
  }
}

```

### defineComputed

> /src/initComputed.js

```javascript
/**
 * 将计算属性代理到 Vue 实例上
 * @param {*} vm Vue 实例
 * @param {*} key computed 的计算属性
 */
function defineComputed(vm, key) {
  // 属性描述符
  const descriptor = {
    get: function () {
      const watcher = vm._watcher[key]
      if (watcher.dirty) { // 说明当前 computed 回调函数在本次渲染周期内没有被执行过
        // 执行 evalute，通知 watcher 执行 computed 回调函数，得到回调函数返回值
        watcher.evalute()
      }
      return watcher.value
    },
    set: function () {
      console.log('no setter')
    }
  }
  // 将计算属性代理到 Vue 实例上
  Object.defineProperty(vm, key, descriptor)
}

```

### Watcher

> /src/watcher.js

```javascript
/**
 * @param {*} cb 回调函数，负责更新 DOM 的回调函数
 * @param {*} options watcher 的配置项
 */
export default function Watcher(cb, options = {}, vm = null) {
  // 备份 cb 函数
  this._cb = cb
  // 回调函数执行后的值
  this.value = null
  // computed 计算属性实现缓存的原理，标记当前回调函数在本次渲染周期内是否已经被执行过
  this.dirty = !!options.lazy
  // Vue 实例
  this.vm = vm
  // 非懒执行时，直接执行 cb 函数，cb 函数中会发生 vm.xx 的属性读取，从而进行依赖收集
  !options.lazy && this.get()
}

```

#### watcher.get

> /src/watcher.js

```javascript
/**
 * 负责执行 Watcher 的 cb 函数
 * 执行时进行依赖收集
 */
Watcher.prototype.get = function () {
  pushTarget(this)
  this.value = this._cb.apply(this.vm)
  popTarget()
}

```

#### watcher.update

> /src/watcher.js

```javascript
/**
 * 响应式数据更新时，dep 通知 watcher 执行 update 方法，
 * 让 update 方法执行 this._cb 函数更新 DOM
 */
Watcher.prototype.update = function () {
  // 通过 Promise，将 this._cb 的执行放到 this.dirty = true 的后面
  // 否则，在点击按钮时，computed 属性的第一次计算会无法执行，
  // 因为 this._cb 执行的时候，会更新组件，获取计算属性的值的时候 this.dirty 依然是
  // 上一次的 false，导致无法得到最新的的计算属性的值
  // 不过这个在有了异步更新队列之后就不需要了，当然，毕竟异步更新对象的本质也是 Promise
  Promise.resolve().then(() => {
    this._cb()
  })
  // 执行完 _cb 函数，DOM 更新完毕，进入下一个渲染周期，所以将 dirty 置为 false
  // 当再次获取 计算属性 时就可以重新执行 evalute 方法获取最新的值了
  this.dirty = true
}

```

#### watcher.evalute

> /src/watcher.js

```javascript
Watcher.prototype.evalute = function () {
  // 执行 get，触发计算函数 (cb) 的执行
  this.get()
  // 将 dirty 置为 false，实现一次刷新周期内 computed 实现缓存
  this.dirty = false
}

```

### pushTarget

> /src/dep.js

```javascript
// 存储所有的 Dep.target
// 为什么会有多个 Dep.target?
// 组件会产生一个渲染 Watcher，在渲染的过程中如果处理到用户 Watcher，
// 比如 computed 计算属性，这时候会执行 evalute -> get
// 假如直接赋值 Dep.target，那 Dep.target 的上一个值 —— 渲染 Watcher 就会丢失
// 造成在 computed 计算属性之后渲染的响应式数据无法完成依赖收集
const targetStack = []

/**
 * 备份本次传递进来的 Watcher，并将其赋值给 Dep.target
 * @param {*} target Watcher 实例
 */
export function pushTarget(target) {
  // 备份传递进来的 Watcher
  targetStack.push(target)
  Dep.target = target
}

```

### popTarget

> /src/dep.js

```javascript
/**
 * 将 Dep.target 重置为上一个 Watcher 或者 null
 */
export function popTarget() {
  targetStack.pop()
  Dep.target = targetStack[targetStack.length - 1]
}

```

## 结果

好了，到这里，Vue computed 属性实现就完成了，如果你能看到如下效果图，则说明一切正常。

动图地址：https://gitee.com/liyongning/typora-image-bed/raw/master/202203161832189.image

![Jun-20-2021 10-50-02.gif](https://gitee.com/liyongning/typora-image-bed/raw/master/202203161832189.image)

可以看到，页面中的计算属性已经正常显示，而且也可以做到响应式更新，且具有缓存的能力（通过控制台查看 computed 输出）。

到这里，手写 Vue 系列就剩最后一部分内容了 —— **手写 Vue 系列 之 异步更新队列**。

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