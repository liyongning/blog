# uni-app、Vue3 + ucharts 图表 H5 无法渲染

**当学习成为了习惯，知识也就变成了常识。** 感谢各位的 **关注**、**点赞**、**收藏**和**评论**。

新视频和文章会第一时间在微信公众号发送，欢迎关注：[李永宁lyn](https://gitee.com/liyongning/typora-image-bed/raw/master/202202171742614.jpg)

文章已收录到 [github 仓库 liyongning/blog](https://github.com/liyongning/blog)，欢迎 Watch 和 Star。

## 封面

![logo (14)](https://gitee.com/liyongning/typora-image-bed/raw/master/202202161818405.png)

## 简介

从问题定位开始，到给框架（uni-app）提 issue、出解决方案(PR)，再到最后的思考，详细记录了整个过程。

## 前序

当你在业务中不幸踩了开源框架的某些坑，这是你的不幸，但这同时也是你的幸运，因为这是你给自己简历中增加亮点的绝佳机会。

而给开源社区贡献 PR 是你证明自己技术侧拥有 P7 实力的绝佳方式，P7 的评判标准无非是业务和技术，业务上有收益，技术上有深度和广度（别人有的你能做的更好，别人没有的你能有）。

这次整个过程历时 3-4 天，在此之前我也没读过 uni-app 和 ucharts 的源码，所以这里把整个过程分享出来也是给大家一个解决问题的思路。

## 环境

- uni-app cli 版本 3.0.0-alpha-3030820220114011
- hbuilder 版本 3.3.8.20220114-alpha
- ucharts 版本 uni-modules 2.3.7-20220122
## 现象

uni-app、vue3 + ucharts 绘制图表，开发环境正常，但是打包上线后，H5 无法绘制图表，也不报任何错误。

|  | 开发 | 线上 |
| --- | --- | --- |
| APP | 正常 | 正常 |
| H5 | 正常 | 无法绘制 |

## 问题定位
给 ucharts 的社区提 issue，经过交流，维护者 “怀疑“ 是 uni-app 的 vue3 的 renderjs 有问题，但是他也给不了一个肯定的答复，让去 uni-app 的社区提 issue 而且示例中不能用 ucharts。个人对于该回答持怀疑态度，于是决定自己去定位问题。
### 怀疑是 ucharts 的 bug

- ucharts 视图部分的关键代码
```vue
<view ...其它属性 :prop="uchartsOpts" :change:prop="rdcharts.ucinit">
  <canvas ...属性 />
</view>
```

> **这里有一个知识点需要补充**：当 prop 发生改变，change:prop 的回调会被调用，这是 uni-app 框架提供的能力，但官方文档没有提及，从源码中可以看到。

- 看了 ucharts 的源码，绘制图表时的代码执行过程如下：

![](https://gitee.com/liyongning/typora-image-bed/raw/master/202202161521960.png)

可是打包后的 H5 线上环境，当执行 `this.uchartsOpts = newConfig` 之后却没有触发 `change:prop` 事件，所以这看起来似乎是 uni-app 的 view 组件有问题

> **感谢** ucharts 官方，在定位问题过程中，和社区进行交流后，ucharts 免费赠送了一个**永久超级会员**，感谢 🙏 🙏 !!

### view 组件的 prop 和 change:prop

提供如下示例：

```vue
<template>
  <view>
    <view :prop="counter" :change:prop="changeProp"></view>
		<view>{{ msg }}</view>
  </view>
</template>

<script setup lang="ts">
import { onBeforeUnmount, onMounted, ref } from "vue";

const counter = ref(1)
const msg = ref('hello')

function changeProp() {
	msg.value = 'hello' + counter.value
}

// @ts-ignore
let timer = null
onMounted(() => {
	timer = setInterval(() => {
		counter.value += 1
	}, 1000)
})

onBeforeUnmount(() => {
	// @ts-ignore
	clearInterval(timer)
})
</script>

<style>
</style>

```
|  | H5 开发环境 | H5 打包后 |
| --- | --- | --- |
| vue2 | 正常 | 正常 |
| vue3 | 正常 | change:prop 未执行 |

因为开发环境没有问题，所以在开发环境中通过在 change:prop 方法中打断点，查看调用栈，找到触发 change:prop 回调的方法，再一步步往上看，终于发现了 uni-app 重写渲染器（render 函数）的地方，在 @dcloudio/uni-h5-vue/dist/vue.runtime.esm.js 中。​

通过阅读 uni-app 的源码，得到如下内容：

响应式数据发生变化，触发 vue 的响应式更新。比如你的响应式数据作为元素的 prop 属性传递，则在 patch 阶段会触发 patchProps 方法， 触发该方法后，方法内判断新老 props 是否发生改变，如果变了，则遍历新的 props 对象，将其中的每个属性、值和老的对比，如果不相等 或者 props 的 key 为 change:xx 则直接调用 patchProp 方法，如果 `__UNI_FEATURE_WXS__`为真并且 props 的 key 为 change: 开头，则调用 patchWxs，patchWxs 方法最终会通过 nextTick 调用 change:prop 的回调方法。

以下为上述执行过程的流程图：

![](https://gitee.com/liyongning/typora-image-bed/raw/master/202202161549669.jpg)

最终定位到问题就出在 `__UNI_FEATURE_WXS__`上，发现开发环境中它是 true，但是打包后就变成了 false。

### \_\_UNI_FEATURE_WXS__
`__UNI_FEATURE_WXS__`是一个全局变量，所以肯定是通过 vite 的 define 选项进行设置的。

于是接下来的目的就是需要找到 `__UNI_FEATURE_WXS__`是在什么地方进行设置的。可以全局搜该变量，然后找到在 `@dcloudio/uni-cli-shared` 包中找到一个叫 `initFeatures` 的方法，该方法中声明了一个 `features` 对象：
```javascript
const {
  wx,
  wxs,
  // ...其它变量
} = extend(
  initManifestFeature(options),
  // ... 其它方法
)

const features = {
  // vue
  __VUE_OPTIONS_API__: vueOptionsApi, // enable/disable Options API support, default: true
  __VUE_PROD_DEVTOOLS__: vueProdDevTools, // enable/disable devtools support in production, default: false
  // uni
  __UNI_FEATURE_WX__: wx, // 是否启用小程序的组件实例 API，如：selectComponent 等（uni-core/src/service/plugin/appConfig）
  __UNI_FEATURE_WXS__: wxs, // 是否启用 wxs 支持，如：getComponentDescriptor 等（uni-core/src/view/plugin/appConfig）
  // ... 其它属性
}
```
看了该对象的设置没什么问题，`wxs`在开发和生产环境下都是 true。那接下来就需要找到谁调用了 initFeatures 方法，而且可能调用完了以后通过判断当前命令，比如：执行 build 时，将 `__UNI_FEATURE_WXS__`设置为了 false。

刚开始想正向推导。vite-plugin-uni 是 uni-app 提供给 vite 的一个插件框架，uni-app 中的 vite 配置都来自于这里。

插件当中的 uni 插件提供了 config 选项，config 选项的值是调用 createConfig 方法返回的函数，该函数会返回一个对象，该对象会和 vite 的配置做深度合并；该对象有 define 选项，该选项的值为 createDefine 函数的返回值，该返回值是一个对象，其中调用了 initDefine，再往下看发现不对，然后路 走死了。

发现上面正向推导的方式走不通以后，于是开始反向推导，即全局搜索，都有哪些地方调用了 initFeatures，然后一步步的往下推，得到如下正确的流程图：

![](https://gitee.com/liyongning/typora-image-bed/raw/master/202202161552762.png)

经过最终的调试，发现 启动开发环境和打包时最终的调用路径是：uniH5Plugin -> createConfig -> configDefine -> initFeatures。
而最终的问题也就是出在了 initFeatures 方法调用的 initManifestFeature 方法中。

## 答案
最终定位到出问题的地方在 `@dcloudio/uni-cli-shared/src/vite/features.ts` 文件的 `initManifestFeature` 方法中。有如下对比：

- github 仓库的最新代码，版本号：3.0.0-alpha-3030820220114011
```javascript
if (command === 'build') {
    // TODO 需要预编译一遍？
    // features.wxs = false
    // features.longpress = false
  }
```

- 已发版的代码，最高版本号：3.0.0-alpha-3031120220208001
```javascript
if (command === 'build') {
    // TODO 需要预编译一遍？
    features.wxs = false;
    features.longpress = false;
}
```
已发版的版本居然高于仓库内的最新版本号。查看 npm 上的发布版本信息：

![](https://gitee.com/liyongning/typora-image-bed/raw/master/202202161555456.png)

发现版本号发生了回退。这几次回退的版本号都是不符合规范的版本号，而且其中可能携带了 bug，比如上面提到的最高版本。

发版出现版本号不符合规范的情况是由于项目还没有一个规范的发版流程导致的，但是已经是 alpha 版本了，这种低级错误还是应该避免的。

更致命的操作是，回退版本号。uni-app 目前每次升级都是升级的最小版本号后面的数值，而业务项目的 package.json 都是 `"@dcloudio/uni-app": "^xxx"` 的形式，这就意味着，你每次重新装包（比如自动化部署时）或者升级包时，都会更新到这个存在 bug 的高版本，这就会导致线上系统报 bug。
## 解决方案
所以这里正确的处理方式是重新发一个更高版本的包，而不是回退版本。因为该操作会导致用户线上的系统出 bug，即以下代码无法正常执行：
```vue
<view :prop="msg" :change:prop="cb"></view>
```
当正常情况下，当 msg 改变后，change:prop 的回调会执行。但是这个携带 bug 的高版本包，在打包时（npm run build）将 `__UNI_FEATURE_WXS__`设置为了 false，导致 change:prop 的回调不会被调用。
## 总结
代码可以回退，但是版本号不要回退，应该基于当前稳定版本，重新发一版版本号更高的版本。

于是就给官方提了 [issue 和 解决方案](https://github.com/dcloudio/uni-app/issues/3251)。
## 结果
官方已采纳该解决方案，基于当前稳定版重新发布一版版本号更高的版本。
## 思考
针对 uni-app 这种处于 alpha 版本的框架，项目内部也确实不应该继续使用 ^ 符号，还是应该将版本号写死为最新的 tag 版本，因为总跟随 alpha 的最新版，确实可能会踩坑。



## 链接

* [精通 uni-app 专栏](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzA3NTk4NjQ1OQ==&action=getalbum&album_id=2271926502294519808#wechat_redirect)
* [精通 Vue 技术栈源码原理 专栏](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzA3NTk4NjQ1OQ==&action=getalbum&album_id=2273541436891693065#wechat_redirect)
* [组件库 专栏](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzA3NTk4NjQ1OQ==&action=getalbum&album_id=2259813235891863559#wechat_redirect)
* [微前端 专栏](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzA3NTk4NjQ1OQ==&action=getalbum&album_id=2251416802327232513#wechat_redirect)



感谢各位的：**关注**、**点赞**、**收藏**和**评论**，我们下期见。

***

**当学习成为了习惯，知识也就变成了常识。** 感谢各位的 **关注**、 **点赞**、**收藏**和**评论**。

新视频和文章会第一时间在微信公众号发送，欢迎关注：[李永宁lyn](https://gitee.com/liyongning/typora-image-bed/raw/master/202202171742614.jpg)

文章已收录到 [github 仓库 liyongning/blog](https://github.com/liyongning/blog)，欢迎 Watch 和 Star。
