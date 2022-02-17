# 手写 Vue2 系列 之 patch —— diff

**当学习成为了习惯，知识也就变成了常识。** 感谢各位的 **关注**、**点赞**、**收藏**和**评论**。

新视频和文章会第一时间在微信公众号发送，欢迎关注：[李永宁lyn](https://gitee.com/liyongning/typora-image-bed/raw/master/202202171742614.jpg)

文章已收录到 [github 仓库 liyongning/blog](https://github.com/liyongning/blog)，欢迎 Watch 和 Star。

## 封面

![](https://gitee.com/liyongning/typora-image-bed/raw/master/202203151930501.png)

## 前言

上一篇文章 [手写 Vue2 系列 之 初始渲染](https://mp.weixin.qq.com/s?__biz=MzA3NTk4NjQ1OQ==&mid=2247485327&idx=1&sn=62d8c7e821a22c8591aa347e84ea8630&chksm=9f6964fba81eeded7cdc07a547a2643e3844c1010e43f82380094a3c2f3df0acf7d680f664e8#rd) 中完成了原始标签、自定义组件、插槽的的初始渲染，当然其中也涉及到 v-bind、v-model、v-on 指令的原理。完成首次渲染之后，接下来就该进行后续的更新了：

响应式数据发生更新 -> setter 拦截到更新操作 -> dep 通知 watcher 执行 update 方法 -> 进而执行 updateComponent 方法更新组件 -> 执行 render 生成新的 vnode -> 将 vnode 传递给 vm._update 方法 -> 调用 patch 方法 -> 执行 patchVnode 进行 DOM diff 操作 -> 完成更新

## 目标

所以，本篇的目标就是实现 DOM diff，完成后续更新。涉及知识点只有一个：DOM diff。

## 实现

接下来就开始实现 DOM diff，完成响应式数据的后续更新。

### patch

> /src/compiler/patch.js

```javascript
/**
 * 负责组件的首次渲染和后续更新
 * @param {VNode} oldVnode 老的 VNode
 * @param {VNode} vnode 新的 VNode
 */
export default function patch(oldVnode, vnode) {
  if (oldVnode && !vnode) {
    // 老节点存在，新节点不存在，则销毁组件
    return
  }

  if (!oldVnode) { // oldVnode 不存在，说明是子组件首次渲染
  } else {
    if (oldVnode.nodeType) { // 真实节点，则表示首次渲染根组件
    } else {
      // 后续的更新
      patchVnode(oldVnode, vnode)
    }
  }
}

```

### patchVnode

> /src/compiler/patch.js

```javascript
/**
 * 对比新老节点，找出其中的不同，然后更新老节点
 * @param {*} oldVnode 老节点的 vnode
 * @param {*} vnode 新节点的 vnode
 */
function patchVnode(oldVnode, vnode) {
  // 如果新老节点相同，则直接结束
  if (oldVnode === vnode) return

  // 将老 vnode 上的真实节点同步到新的 vnode 上，否则，后续更新的时候会出现 vnode.elm 为空的现象
  vnode.elm = oldVnode.elm

  // 走到这里说明新老节点不一样，则获取它们的孩子节点，比较孩子节点
  const ch = vnode.children
  const oldCh = oldVnode.children

  if (!vnode.text) { // 新节点不存在文本节点
    if (ch && oldCh) { // 说明新老节点都有孩子
      // diff
      updateChildren(ch, oldCh)
    } else if (ch) { // 老节点没孩子，新节点有孩子
      // 增加孩子节点
    } else { // 新节点没孩子，老节点有孩子
      // 删除这些孩子节点
    }
  } else { // 新节点存在文本节点
    if (vnode.text.expression) { // 说明存在表达式
      // 获取表达式的新值
      const value = JSON.stringify(vnode.context[vnode.text.expression])
      // 旧值
      try {
        const oldValue = oldVnode.elm.textContent
        if (value !== oldValue) { // 新老值不一样，则更新
          oldVnode.elm.textContent = value
        }
      } catch {
        // 防止更新时遇到插槽，导致报错
        // 目前不处理插槽数据的响应式更新
      }
    }
  }
}

```

### updateChildren

> /src/compiler/patch.js

```javascript
/**
 * diff，比对孩子节点，找出不同点，然后将不同点更新到老节点上
 * @param {*} ch 新 vnode 的所有孩子节点
 * @param {*} oldCh 老 vnode 的所有孩子节点
 */
function updateChildren(ch, oldCh) {
  // 四个游标
  // 新孩子节点的开始索引，叫 新开始
  let newStartIdx = 0
  // 新结束
  let newEndIdx = ch.length - 1
  // 老开始
  let oldStartIdx = 0
  // 老结束
  let oldEndIdx = oldCh.length - 1
  // 循环遍历新老节点，找出节点中不一样的地方，然后更新
  while (newStartIdx <= newEndIdx && oldStartIdx <= oldEndIdx) { // 根为 web 中的 DOM 操作特点，做了四种假设，降低时间复杂度
    // 新开始节点
    const newStartNode = ch[newStartIdx]
    // 新结束节点
    const newEndNode = ch[newEndIdx]
    // 老开始节点
    const oldStartNode = oldCh[oldStartIdx]
    // 老结束节点
    const oldEndNode = oldCh[oldEndIdx]
    if (sameVNode(newStartNode, oldStartNode)) { // 假设新开始和老开始是同一个节点
      // 对比这两个节点，找出不同然后更新
      patchVnode(oldStartNode, newStartNode)
      // 移动游标
      oldStartIdx++
      newStartIdx++
    } else if (sameVNode(newStartNode, oldEndNode)) { // 假设新开始和老结束是同一个节点
      patchVnode(oldEndNode, newStartNode)
      // 将老结束移动到新开始的位置
      oldEndNode.elm.parentNode.insertBefore(oldEndNode.elm, oldCh[newStartIdx].elm)
      // 移动游标
      newStartIdx++
      oldEndIdx--
    } else if (sameVNode(newEndNode, oldStartNode)) { // 假设新结束和老开始是同一个节点
      patchVnode(oldStartNode, newEndNode)
      // 将老开始移动到新结束的位置
      oldStartNode.elm.parentNode.insertBefore(oldStartNode.elm, oldCh[newEndIdx].elm.nextSibling)
      // 移动游标
      newEndIdx--
      oldStartIdx++
    } else if (sameVNode(newEndNode, oldEndNode)) { // 假设新结束和老结束是同一个节点
      patchVnode(oldEndNode, newEndNode)
      // 移动游标
      newEndIdx--
      oldEndIdx--
    } else {
      // 上面几种假设都没命中，则老老实的遍历，找到那个相同元素
    }
  }
  // 跳出循环，说明有一个节点首先遍历结束了
  if (newStartIdx < newEndIdx) { // 说明老节点先遍历结束，则将剩余的新节点添加到 DOM 中

  }
  if (oldStartIdx < oldEndIdx) { // 说明新节点先遍历结束，则将剩余的这些老节点从 DOM 中删掉

  }
}

```

### sameVNode

> /src/compiler/patch.js

```javascript
/**
 * 判断两个节点是否相同
 * 这里的判读比较简单，只做了 key 和 标签的比较
 */
function sameVNode(n1, n2) {
  return n1.key == n2.key && n1.tag === n2.tag
}

```

## 结果

好了，到这里，虚拟 DOM 的 diff 过程就完成了，如果你能看到如下效果图，则说明一切正常。

动图地址：https://gitee.com/liyongning/typora-image-bed/raw/master/202203151929235.image

![Jun-18-2021 09-11-18.gif](https://gitee.com/liyongning/typora-image-bed/raw/master/202203151929235.image)

可以看到，页面已经完全做到响应式数据的初始渲染和后续更新。其中关于 Computed 计算属性的内容仍然没有正确的显示出来，这很正常，因为还没实现这个功能，所以接下来就会去实现 conputed 计算属性，也就是下一篇内容 **手写 Vue2 系列 之 computed**。

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