**当学习成为了习惯，知识也就变成了常识。** 感谢各位的 **关注**、**点赞**、**收藏**和**评论**。

新视频和文章会第一时间在微信公众号发送，欢迎关注：[李永宁lyn](https://gitee.com/liyongning/typora-image-bed/raw/master/202202171742614.jpg)

文章已收录到 [github 仓库 liyongning/blog](https://github.com/liyongning/blog)，欢迎 Watch 和 Star。

**[更多精彩内容](https://github.com/liyongning/blog/blob/main/README.md)**

<hr />

# 问题

开始之前提几个问题，如果你能都清楚的回答出来，那就不用再继续往下看了，但如果发现有疑问，则可以继续往下阅读。

* script 标签上的 async 和 defer 属性有什么作用？在什么场景中使用？这两个属性有什么区别？
* `<script type="module"><script>`  和 带有 async、defer 属性的 script 标签在执行顺序有什么异同？
* type="module" 的 script、带有 async 和 defer 的 script 是否会阻塞 DomContentLoaded 事件的触发？
* rel="modulepreload" 有什么作用和使用场景？

# 简介



js 文件都是同步代码，没有逻辑，都是 console.log 输出

![[图片]](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202509101535389.png)

![[图片]](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202509101535203.png)

结论是：

1. 同步脚本，比如 b.js，不论什么位置，执行一定会阻塞渲染，因为渲染和 js 执行都在主线程上
2. module 和 defer 确实效果一样，都属于异步加载，并且在 DomContentLoaded 触发之前按照脚本顺序执行，换个角度说 moduel 和 defer 会阻塞 DomContentLoaded 的触发，这也是线上需要着重优化的方向
3. module 和 defer 的区别是，module 会通过递归方式查找整个依赖树，即示例中的 d.js 依赖 f.js，并且 f.js 会阻塞 d.js 的执行，这里有个优化点就是设置 modulepreload，可以提前解析和加载整个依赖树，否则只能是先解析 d.js 发现有依赖项，再递推的去解析和加载依赖项
4. async 脚本，异步加载，加载完就执行，但执行时机不确定，首先和脚本位置无关，另外一点就是不会阻塞 DomContentLoaded 事件的触发，但 defer 和 module 会阻塞


昨晚改的那一版，把 quc 和 jQuery 放在了 body 中并且是同步加载，理论上是能解决问题了。但是，性能优化之前是放在 head 中同步加载的，刚试了下，打包后的效果如图，理论上不应该出现前端和客户端登录态不同步的现象，暂时不知道原因，大家可以再探索下，有没有别的可能

![[图片]](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202509101535779.png)

不同步现象：
![[图片]](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202509101534922.png)
