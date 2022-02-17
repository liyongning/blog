# 手写 Vue2 系列 之 异步更新队列

**当学习成为了习惯，知识也就变成了常识。** 感谢各位的 **关注**、**点赞**、**收藏**和**评论**。

新视频和文章会第一时间在微信公众号发送，欢迎关注：[李永宁lyn](https://gitee.com/liyongning/typora-image-bed/raw/master/202202171742614.jpg)

文章已收录到 [github 仓库 liyongning/blog](https://github.com/liyongning/blog)，欢迎 Watch 和 Star。

## 封面

![](https://gitee.com/liyongning/typora-image-bed/raw/master/202203141833875.png)

## 前言

上一篇文章 [手写 Vue 系列 之 computed](https://mp.weixin.qq.com/s?__biz=MzA3NTk4NjQ1OQ==&mid=2247485354&idx=1&sn=5e7f0d88192c9119012c49f131327fee&chksm=9f6964dea81eedc8a84d4b214f9e0675ac5bc869d718b3583d94086498c1d0752f1cd7634dc8#rd) 实现了 Vue 的 computed 计算属性。

## 目标

本篇文章是 `手写 Vue 系列` 的最后一篇，实现 Vue 的异步更新队列。

读过源码，相信大家都知道 Vue 异步更新的大概流程：依赖收集结束之后，当响应式数据发生变化 -> 触发 setter 执行 dep.notify -> 让 dep 通知 自己收集的所有 watcher 执行 update 方法 -> watch.update 调用 queueWatcher 将自己放到 watcher 队列 -> 接下来调用 nextTick 方法将刷新 watcher 队列的方法放到 callbacks 数组 -> 然后将刷新 callbacks 数组的方法放到浏览器的异步任务队列 -> 待将来执行时最终触发 watcher.run 方法，执行 watcher.get 方法。

## 实现

接下来会完整实现 Vue 的异步更新队列，让你彻底理解 Vue 的异步更新过程都发生了什么。

### Watcher

> /src/watcher.js

```javascript
// 用来标记 watcher
let uid = 0

**
 * @param {*} cb 回调函数，负责更新 DOM 的回调函数
 * @param {*} options watcher 的配置项
 */
export default function Watcher(cb, options = {}, vm = null) {
  // 标识 watcher
  this.uid = uid++
  // ...
}

```

### watcher.update

> /src/watcher.js

```javascript
/**
 * 响应式数据更新时，dep 通知 watcher 执行 update 方法，
 * 让 update 方法执行 this._cb 函数更新 DOM
 */
Watcher.prototype.update = function () {
  if (this.options.lazy) { // 懒执行，比如 computed 计算属性
    // 将 dirty 置为 true，当页面重新渲染获取计算属性时就可以执行 evalute 方法获取最新的值了
    this.dirty = true
  } else {
    // 将 watcher 放入异步 watcher 队列
    queueWatcher(this)
  }
}

```

### watcher.run

> /src/watcher.js

```javascript
/**
 * 由刷新 watcher 队列的函数调用，负责执行 watcher.get 方法
 */
Watcher.prototype.run = function () {
  this.get()
}

```

### 异步更新队列

> /src/asyncUpdateQueue.js

```javascript
/**
 * 异步更新队列
 */

// 存储本次更新的所有 watcher
const queue = []

// 标识现在是否正在刷新 watcher 队列
let flushing = false
// 标识，保证 callbacks 数组中只会有一个刷新 watcher 队列的函数
let waiting = false
// 存放刷新 watcher 队列的函数，或者用户调用 Vue.nextTick 方法传递的回调函数
const callbacks = []
// 标识浏览器当前任务队列中是否存在刷新 callbacks 数组的函数
let pending = false

```

#### queueWatcher

> /src/asyncUpdateQueue.js

```javascript
/**
 * 将 watcher 放入队列
 * @param {*} watcher 待会儿需要被执行的 watcher，包括渲染 watcher、用户 watcher、computed
 */
export function queueWatcher(watcher) {
  if (!queue.includes(watcher)) { // 防止重复入队
    if (!flushing) { // 现在没有在刷新 watcher 队列
      queue.push(watcher)
    } else { // 正在刷新 watcher 队列，比如用户 watcher 的回调函数中更改了某个响应式数据
      // 标记当前 watcher 在 for 中是否已经完成入队操作
      let flag = false
      // 这时的 watcher 队列时有序的(uid 由小到大)，需要保证当前 watcher 插入进去后仍然有序
      for (let i = queue.length - 1; i >= 0; i--) {
        if (queue[i].uid < watcher.uid) { // 找到了刚好比当前 watcher.uid 小的那个 watcher 的位置
          // 将当前 watcher 插入到该位置的后面
          queue.splice(i + 1, 0, watcher)
          flag = true
          break;
        }
      }
      if (!flag) { // 说明上面的 for 循环在队列中没找到比当前 watcher.uid 小的 watcher
        // 将当前 watcher 插入到队首 
        queue.unshift(watcher)
      }
    }
    if (!waiting) { // 表示当前 callbacks 数组中还没有刷新 watcher 队列的函数
      // 保证 callbacks 数组中只会有一个刷新 watcher 队列的函数
      // 因为如果有多个，没有任何意义，第二个执行的时候 watcher 队列已经为空了
      waiting = true
      nextTick(flushSchedulerQueue)
    }
  }
}

```

#### flushSchedulerQueue

> /src/asyncUpdateQueue.js

```javascript
/**
 * 负责刷新 watcher 队列的函数，由 flushCallbacks 函数调用
 */
function flushSchedulerQueue() {
  // 表示正在刷新 watcher 队列
  flushing = true
  // 给 watcher 队列排序，根据 uid 由小到大排序
  queue.sort((a, b) => a.uid - b.uid)
  // 遍历队列，依次执行其中每个 watcher 的 run 方法
  while (queue.length) {
    // 取出队首的 watcher
    const watcher = queue.shift()
    // 执行 run 方法
    watcher.run()
  }
  // 到这里 watcher 队列刷新完毕
  flushing = waiting = false
}

```

#### nextTick

> /src/asyncUpdateQueue.js

```javascript
/**
 * 将刷新 watcher 队列的函数或者用户调用 Vue.nextTick 方法传递的回调函数放入 callbacks 数组
 * 如果当前的浏览器任务队列中没有刷新 callbacks 的函数，则将 flushCallbacks 函数放入任务队列
 */
function nextTick(cb) {
  callbacks.push(cb)
  if (!pending) { // 表明浏览器当前任务队列中没有刷新 callbacks 数组的函数
    // 将 flushCallbacks 函数放入浏览器的微任务队列
    Promise.resolve().then(flushCallbacks)
    // 标识浏览器的微任务队列中已经存在 刷新 callbacks 数组的函数了
    pending = true
  }
}

```

#### flushCallbacks

> /src/asyncUpdateQueue.js

```javascript
/**
 * 负责刷新 callbacks 数组的函数，执行 callbacks 数组中的所有函数
 */
function flushCallbacks() {
  // 表示浏览器任务队列中的 flushCallbacks 函数已经被拿到执行栈执行了
  // 新的 flushCallbacks 函数可以进入浏览器的任务队列了
  pending = false
  while(callbacks.length) {
    // 拿出最头上的回调函数
    const cb = callbacks.shift()
    // 执行回调函数
    cb()
  }
}

```

## 总结

到这里 `精通 Vue 系列` 就要结束了，现在我们再回头看下整个系列：从 `Vue 源码解读` 开始到现在的 `手写 Vue`，总共 20 篇文章。如果你是从头到尾跟下来的，相信我们最初定的目标早已实现，这会儿你是否可以在自己的简历上写上：精通 Vue 源码原理。

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