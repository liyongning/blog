# 手写 Vue 系列 之 从 Vue1 升级到 Vue2

**当学习成为了习惯，知识也就变成了常识。** 感谢各位的 **关注**、**点赞**、**收藏**和**评论**。

新视频和文章会第一时间在微信公众号发送，欢迎关注：[李永宁lyn](https://gitee.com/liyongning/typora-image-bed/raw/master/202202171742614.jpg)

文章已收录到 [github 仓库 liyongning/blog](https://github.com/liyongning/blog)，欢迎 Watch 和 Star。

## 封面

![](https://gitee.com/liyongning/typora-image-bed/raw/master/202203101859875.png)

## 前言

上一篇文章 [手写 Vue 系列 之 Vue1.x](https://mp.weixin.qq.com/s?__biz=MzA3NTk4NjQ1OQ==&mid=2247485261&idx=1&sn=63c8469dc639dac6c7a9395f42045087&chksm=9f696439a81eed2f9327625b09a9b3efaacbca41835ca616833a24c7cdd88a23c71d4f37f5f2#rd) 带大家从零开始实现了 Vue1 的核心原理，包括如下功能：

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
  

在最后也详细讲解了 Vue1 的诞生以及存在的问题：Vue1.x 在中小型系统中性能会很好，定向更新 DOM 节点，但是大型系统由于 Watcher 太多，导致资源占用过多，性能下降。于是 Vue2 中通过引入 VNode 和 Diff 的来解决这个问题，

所以接下来的系列内容就是升级上一篇文章编写的 `lyn-vue` 框架，将它从 Vue1 升级到 Vue2。所以建议整个系列大家按顺序去阅读学习，如若强行阅读，可能会产生云里雾里的感觉，事倍功半。

另外欢迎 **关注** 以防迷路，同时系列文章都会收录到 [精通 Vue 技术栈的源码原理](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzA3NTk4NjQ1OQ==&action=getalbum&album_id=2273541436891693065#wechat_redirect) 专栏，也欢迎关注该专栏。

## 目标

升级后的框架需要将如下示例代码跑起来

### 示例

```html
<!DOCTYPE html>
<html lang="en">

<head>
  <meta charset="UTF-8">
  <title>Lyn Vue2.0</title>
</head>

<body>
  <div id="app">
    <h3>数据响应式更新 原理</h3>
    <div>{{ t }}</div>
    <div>{{ t1 }}</div>
    <div>{{ arr }}</div>
    <h3>methods + computed + 异步更新队列 原理</h3>
    <div>
      <p>{{ counter }}</p>
      <div>{{ doubleCounter }}</div>
      <div>{{ doubleCounter }}</div>
      <div>{{ doubleCounter }}</div>
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
      <input type="checkbox" v-model="isChecked" />
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
    <h3>组件 原理</h3>
    <comp></comp>
    <h3>插槽 原理</h3>
    <scope-slot></scope-slot>
    <scope-slot>
      <template v-slot:default="scopeSlot">
        <div>{{ scopeSlot }}</div>
      </template>
    </scope-slot>
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
          title: "I am title",
          // v-model
          inputVal: 'test',
          isChecked: true,
          selectValue: 2,
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
      // computed + 异步更新队列 的原理
      computed: {
        doubleCounter() {
          console.log('evalute doubleCounter')
          return this.counter * 2
        }
      },
      // 组件
      components: {
        // 子组件
        'comp': {
          template: `
            <div>
              <p>{{ compCounter }}</p>
              <p>{{ doubleCompCounter }}</p>
              <p>{{ doubleCompCounter }}</p>
              <p>{{ doubleCompCounter }}</p>
              <button v-on:click="handleCompAdd"> comp add </button>
              <button v-on:click="handleCompMinus"> comp minus </button>
            </div>`,
          data() {
            return {
              compCounter: 0
            }
          },
          methods: {
            handleCompAdd() {
              this.compCounter++
            },
            handleCompMinus() {
              this.compCounter--
            }
          },
          computed: {
            doubleCompCounter() {
              console.log('evalute doubleCompCounter')
              return this.compCounter * 2
            }
          }
        },
        // 插槽
        'scope-slot': {
          template: `
            <div>
              <slot name="default" v-bind:slotKey="slotKey">{{ slotKey }}</slot>
            </div>
          `,
          data() {
            return {
              slotKey: 'scope slot content'
            }
          }
        }
      }
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

### 知识点

示例代码涉及的知识点包括：

* 基于模版解析的编译器

  * 解析模版得到 AST
  
  * 基于 AST 生成渲染函数
  
  * render helper
  
    * _c，创建指定标签的 VNode
    
    * _v，创建文本节点的 VNode
    
    * _t，创建插槽节点的 VNode
  
  * VNode
  
* patch

  * 原生标签和组件的初始渲染
  
    * v-model
    
    * v-bind
    
    * v-on
  
  * diff
  
* 插槽原理

* computed

* 异步更新队列

### 效果

示例代码最终的运行效果如下：

![Jun-13-2021 14-12-43.gif](https://gitee.com/liyongning/typora-image-bed/raw/master/202203092034307.image)

## 说明

该框架只为讲解 Vue 的核心原理，没有什么健壮性可言，说不定你换个示例代码可能就会报错、跑不起来，但是用来学习是完全足够了，基本上把 Vue 的核心原理（知识点）都实现了一遍。

所以接下来就开始正式的学习之旅吧，加油！！

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