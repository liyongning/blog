**当学习成为了习惯，知识也就变成了常识。** 感谢各位的 **关注**、**点赞**、**收藏**和**评论**。

新视频和文章会第一时间在微信公众号发送，欢迎关注：[李永宁lyn](https://gitee.com/liyongning/typora-image-bed/raw/master/202202171742614.jpg)

文章已收录到 [github 仓库 liyongning/blog](https://github.com/liyongning/blog)，欢迎 Watch 和 Star。

# 封面

![封面.png](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081300507.png)
# 回顾
上一篇 [PDF 生成（3）— 封面、尾页](https://github.com/liyongning/blog/issues/44) 介绍了如何为通过 Puppeteer 生成的 PDF 文件添加封面和尾页，现在再来整体回顾一下：

- 首先，技术方案决定了一个页面对应一份 PDF 文件，这是大前提，因为 page.xx 方法的所有配置都是针对当前页的
- 在大前提下，我们通过 PDF 文件合并方案（pdf-lib），分别将封面 PDF、内容页 PDF 和尾页 PDF 三份文件合并为一份报告包含封面、内容页和尾页的完整 PDF

在上一篇结束后，PDF 文件的整体框架已经基本形成（包括封面、内容页、尾页），但还有一点点不完整，比如缺少**目录页**。一份完整的文件或文章怎么能没有目录呢？
# 简介
本文详细阐述了如何为 PDF 文件增加目录页，让文件更加完整和易于阅读。目录页在一本书或一篇长文中扮演着非常重要的角色，它是内容的整体概览，可以帮助读者快速了解内容的整体结构，并定位到感兴趣的章节。

PDF 文件也不例外，比如前面我们生成的**百度新闻**的 PDF 文件的内容部分已经有 5页了，共 12个版块，但每个版块的内容是什么，用户只有全部浏览一遍才能知道，效率太低，体验太差。

所以我们的 PDF 文件需要一个目录页，能让用户快速了解到这份文件都包含哪些板块，每个版块在什么位置，并能直接定位过去。
PDF 文件生成目录页的方案非常少，几乎没找到，比如 jsPDF 这个库，它可以生成一份 PDF 文件，并对指定位置的内容设置链接，跳转到某个页面，但它不擅长加载已有的 PDF 文件，并编辑它。

在我们这套技术架构下，有一个现成的方案，就是 **HTML 锚点**，因为通过浏览器打印系统生成的 PDF 文件，可以保留部分 HTML + CSS 的能力，接下来就进入实战阶段，为百度新闻 PDF 文件增加目录页。

**注意：为 PDF 生成目录页是整套方案中的难点之一，特别是页码部分**

# 生成目录页
前面我们提到了目录页的方案是 **HTML 锚点**，也就是说目录页中的每个目录项都是一个 `a` 标签，通过点击 a 标签实现页面跳转，因此，我们需要在内容页的前面增加一段由 a 标签组成的 html。

但可惜百度新闻页不是我们自己的页面，没办法直接在页面的开始位置增加这段 HTML，而且这段 HTML 属于 PDF 文件独有的，直接加到现有页面上也不合适。怎么办？

想想我们在 [PDF 生成（2）— 生成 PDF 文件](https://github.com/liyongning/blog/issues/43) 一文中 **打印完整网页（网页滚动 — 懒加载场景）** 模块，我们通过 `page.evaluate` 方法到回调函数操作页面滚动从而加载更多内容。同样，这里我们可以通过该方法为当前页面增加目录页 DOM。代码如下：

![image.png](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081301652.png)
![image.png](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081301810.png)
![image.png](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081302509.png)

整体思路是：

- 通过 `page.evaluate` 方法为浏览器注入一段代码，这段代码会通过 JS 操作 DOM 的方式，完成整个目录页 DOM 的创建，并将 DOM 插入到新闻页 DOM 的最前面
- 目录显示了当前文档都有哪些版块，并且**通过 a 标签的锚点**实现页面跳转的效果

效果如下：

![292667592-e0ce8dae-1c1f-4418-83e7-ce9b274a1d0b.gif](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081302626.gif)

发现目录的基本能力我们已经实现了，但仔细看，会发现有两个问题：

- 目录项没有页码
- 目录页一般都是独立自成一页，但现在却和内容页混到了一页

第二个问题比较简单，我们先来解决。
# 目录页自成一页
直接上结论：这个问题可以通过 `break-after: page` 这个 CSS 样式来解决。

这里讲三个用于在打印中控制元素如何分页或分割的样式，分别是 break-after、break-before 和 break-inside。简单讲，这三个样式的作用类似，都是用来在页面中合适的位置设置断点（即元素边界），比如：

- break-after，用来控制当前元素后面内容的行为，比如 `break-after: page` 意思是当前元素后面的内容强制分页（即新开一页）
- break-before，用来控制当前元素之前的分页行为
- break-inside，用来控制当前元素内部的分页行为

具体内容，大家可以查询 MDN，有详细讲解。

所以，我们只需要给目录的容器节点设置 `page-after: page` 样式，让后面的内容页强制新开一页，和目录页分开。

代码如下：
![image.png](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081303233.png)

效果如下：

![image.png](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081303642.png)

# 目录项页码
一样，先上结论：**目录项的页码 = 锚点对应的元素距离页面顶部的高度 / PDF 一页的高度**。这里大家思考一下再继续，原理很好理解，就不细讲了，直接上代码。

![image-20240308130347761](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081303812.png)这里有一点

需要**注意**：在计算页码之前需要将页面滚动回顶部，因为前面我们为了加载完整的页面，将页面滚动到了底部，直接计算的话，会发现大部分元素的`const { y } = anchorEl.getBoundingClientRect()`为负值，出现这个问题的原因是因为`el.getBoundingClientRect()`是基于视口来计算的，如果不在页面底部，可以想象一下，相关元素大都在视口的上方了，所以计算的 y 就是负值了。

![image-20240308130433665](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081304703.png)

效果如下：

![image.png](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081304383.png)

但这时候发现，页码准确性有点问题，比如最后一个**图片新闻**：

![image.png](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081304443.png)

出现这个问题的原因是因为计算公式中**PDF 一页的高度**的假设，大家可以看上面的代码，这个高度设定的是 **1123**，那为什么是 1123px 呢？

这里有一个知识点 — **A4 纸在不同屏幕分辨率下的像素尺寸**。

A4 纸的标准尺寸是 210mm * 297mm，但是，在屏幕上显示 A4 纸的像素尺寸取决于屏幕的分辨率，比如：

| 屏幕分辨率（像素/英寸） | A4 纸像素尺寸（宽 × 高） |
| --- | --- |
| 72 | 595 × 842 |
| 96（默认） | 794 × 1123 |
| 120 | 1487 x 2105 |
| 150 | 1240 x 1754 |
| 300 | 2480 × 3508 |

其中默认 DPI 为 96，即一英寸显示 96 个像素点，1英寸 = 25.4mm，即 25.4mm = 96px，所以 `(210 / 25.4 * 96) * (297 / 25.4 * 96) = 794 * 1123`。

这就是为什么 PDF 一页的高度设置为 1123px 了。

但现在按照这个值计算出来的页码有问题，是因为百度新闻页的开发（设计）尺寸不是 794px * 1123px。

所以，这里有一个结论：**如果不知道页面的设计尺寸，没办法计算出准确的页码**。

# 如何构造准确的目录页（目录项页码的准确性）
本节不是反驳上面的结论，在当前 Demo 的场景下确实是没办法计算出准确的页码。但我们在实际的 PDF 文件生成业务中，是一定要保证的页码的正确性的。这是整个方案的难点之一，探索过程很难，但知道了结果，发现也就那样，所以也算是一个最佳实践吧。接下来我们就一步步进行，去构造一个拥有准确页码的目录页。
## 设计稿尺寸要求
在 PDF 生成业务场景中，我们是一定知道页面设计尺寸的（毕竟 UI 都是我们自己写的），而且 PDF 页面一般是 A4 纸大小，在 master go 中 A4 纸的大小是 595 * 842 像素。

但设计稿需要以 2 倍图的尺寸来设计，因为真按照 595 * 842 的尺寸来设计的话，最终的设计稿会发现文字的大小基本上都是在 12px 以下，这非常不利于开发。**这点需要注意，属于最佳实践。**

*注意：** `page.pdf({ format: 'A4' })`一定要和设计稿规范一致，比如都是 A4 纸大小，否则计算页码时还会有问题，原因请继续往下看。

## 页面缩放
先上结论：**页面缩放**是指将设计稿尺寸为 X * Y 的页面放到 M * N 的 PDF 页面中，比如将 1190 * 1684 的 UI 页面放到 794 * 1123 的 PDF 页面中。

> 1190 * 1684 是 A4 纸设计稿的 2 倍图尺寸，794 * 1123 是 DPI 为 96（默认）的分辨率对应的 A4 纸的尺寸

现在，我们通过一个 Demo 来模拟真实的 PDF 生成需求，假设有如下场景：

- 设计稿尺寸：1190 * 1684
- 有一个页面，分别在 10px 和 1500px 像素的位置有两个锚点
- 页眉、页脚，页面左右两侧留白空间都为 80px，如下图所示

![image.png](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081305122.png)

- 屏幕分辨率：96 像素/英寸，即对应的 A4 纸的像素是 794 * 1123

根据需求，我们对代码做如下改动：

增加`exact-page-num.html`作为 Demo 页面，代码如下：

![image.png](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081305361.png)

改动`/server/index.mjs`：

- 替换访问的页面为我们的 Demo 页

![image.png](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081306230.png)

- 设置新的目录项

![image.png](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081306471.png)

- 设置页眉、页脚和左右两侧的留白空间

![image.png](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081306057.png)

- 设置 PDF 页面的像素高度

![image.png](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081306211.png)

改造完成，重新生成 PDF，效果如下：

![image.png](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081306865.png)
![image.png](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081306380.png)
![image.png](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081307805.png)

看完生成的 PDF 文件，可以发现如下问题：

- 目录页中的两个目录项对应的页码都是 2，但这两项实际分别在第 2 页和第 3 页
- 设计稿的尺寸是 1190 * 1684，Demo 页面的高度也是 1684px，那理论上 PDF 的内容应该只有一页，但现在却有两页
- 页眉、页脚、左右两侧的空间太大了，根本不是 80px 应该有的效果

**为什么会出现这三个问题呢？**

其原因是我们前面提到的分辨率，设计稿是按照 1190 * 1684 来设计的，对应的 DPI 是 144，但实际生成 PDF 时的 DPI 只有 96，对应的像素尺寸是 794 × 1123。

> 1190 * 1684 是 595 * 842 的 2 倍图尺寸，前面的分辨率和 A4 纸像素尺寸表中显示 595 * 842 对应的 DPI 是 72

因此就出现了一页的设计稿，生成 PDF 之后就变成了两页，因为 PDF 一页的高度只有 1123px，另外也导致了页码计算错误。

上述情况，就相当于高分辨率的内容放到了低分屏上显示。因此，要想生成的 PDF 内容和设计稿一致，我们需要缩放页面，即开头提到的，将 1190 * 1684 的页面放到 794 * 1123 的页面中。

`page.pdf` 方法提供了一个 `scale` 的参数，专门用来缩放页面，默认是 1，我们需要的缩放比例是 1123 / 1684，我们对代码做如下改动：

![image.png](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081307561.png)
![image.png](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081307984.png)

效果如下：

![image.png](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081307288.png)
![image.png](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081307270.png)

现在，目录页中目录项对应的页码终于对了，PDF 内容页的显示也对了。但还没结束，因为还有一个场景需要处理。

**说明**：关于缩放这块儿，其实页眉、页脚的内容也需要缩放，这部分内容我们没有演示。

## 换页 — 高度补充
有时候我们会要求**大标题**新开一页，就像书籍一样，下一章或下一节一般会新起一页。在 PDF 生成业务中同样要求如此，比如，要求锚点 2 新开一页。

实现层面我们可以通过前面介绍的`break-before: page`样式来完成，代码如下：

![image.png](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081307032.png) 

生成的 PDF 效果如下：

![image.png](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081308573.png)
![image.png](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081308648.png)

锚点 2 新开了一页，很完美？但大家再看下目录页中目录项的页码：

![image.png](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081308800.png)

发现两个目录项对应的页码还是 2，但实际页码分别是 2 和 3，这个结果很好解释：**页码的计算方式没变，`break-before: page` 属于打印样式，只在打印场景生效，即浏览器直接打开页面看不到 break-before: page 的样式效果**。

也就是说，锚点是按照页面中的高度来计算的，计算时它所在的高度是第 2 页，但生成 PDF 的时候，由于发现锚点上有`break-before: page`样式，于是就把本该在第二页的元素放到了第 3 页（新起一页）。

原因很简单，但结果是，目录页中目录项的页码显示错误，这个问题该怎么解决呢？

可以发现，如果没有`break-before: page`，一切都很完美，都是因为加上这个样式之后，生成的 PDF 文件中“莫名其妙”的多出来一段空白（想象 Word 文档中的换页效果），这段空白区域导致页码不准。

假设这样一个场景，如果每个锚点的位置都很完美，刚好在下一页的开始位置，那这个问题就不存在了。那有没有可能我们人为的去构造这样一个完美的场景呢？**答案是可以**。

我们知道每页的高度，也知道锚点在页面中的位置（高度)，这就可以计算出当前锚点是否处于 PDF 页面的开头，如果刚好在开头，我们不做任何处理，如果不在，就意味着我们需要让它移动到页面的开头，那怎么让锚点元素移动到开头呢？有两种方案：

- 用一个实际的元素来填充空白区域，将当前锚点元素顶到下一页的开始位置
- 修正元素的计算高度，假如元素距离下一页的开始位置差 20px，那就在计算页码时，将当前元素的高度手动补 20px，这样就可以假设元素在页面的开始位置，从而计算出正确的页码

方案一的效果：

![image.png](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081308958.png)

但我们最终采用了方案二，虽然两个方案原理和效果一样，但方案二的容错性更强，方案一可能会因为一些意外情况（比如计算错误、精度问题等）导致填充元素高度异常，导致产生额外的空白页，比如：锚点x 本来就在页面的开始位置，但由于计算错误，填充一个高度为 2px 的空白区域，导致本该在页面开始位置的元素距离顶部多了 2px，在打印时就会被自动移动到下一页，我们实际看到的 PDF 页面就会多出来一页空白页；方案二同样存在意外，但最差的情况也就是部分目录项的页码计算错误，而不会引起很明显的显示问题。

代码如下：

![image.png](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081309518.png) 
![image-20240308130938911](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081309948.png)

效果如下：

![image.png](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081309984.png)

内容页没变，和前面的效果一样。到这里，一份完善的 PDF 目录页总算是出来了。

# 总结
我们再来回顾一下本文的内容：

- 开头，我们通过 `page.evaluate` 方法为浏览器注入 JS 代码，通过这段 JS 在 PDF 内容页的开始位置（body 的第一个子元素）插入由 a 标签和对应的样式组成的目录页 DOM，从而通过 HTML 锚点实现目录项的页面跳转能力
- 接下来，我们通过为目录页的容器元素设置`break-after: page`样式实现目录页自成一页的效果（和内容页分别两页）
- 然后剩下的所有篇幅都是在讲如何生成带有准确页码的目录项
   - 首先，页码是按照**锚点元素在页面中的高度 / PDF 一页的高度**来计算的
   - 后来，我们通过下面三步来保证目录页中目录项对应页码的准确性
      - 规范化设计稿尺寸（按照 A4 纸对应的 2 倍图尺寸设计）
      - 通过页面缩放解决设计稿 DPI 和实际生成 PDF 时 DPI 的差异问题（彻底统一计算时 PDF 一页的像素高度）
      - 通过页面高度补充的方案解决章节标题换页引起目录项页码计算错误的问题

到这里，PDF 文件的整体框架已完全成型，包括封面、目录页、内容页和尾页四部分。但系列还没结束，接下来我们会通过 [PDF 生成（5）— 内容页支持由多页面组成](https://github.com/liyongning/blog/issues/46) 来提升接入方的使用体验和前端代码可维护性。
开始之前，大家回顾一下现在 PDF 文件内容页的生成，站在接入方的角度看，是否存在问题？假设一个场景，接入方的 PDF 文件呈现的内容量非常大，比如拥有几十甚至几百页的内容，那接入方的这个前端页面的代码该怎么维护呢？页面性能该怎么保证呢？
# 链接

- [PDF 生成（1）— 开篇](https://github.com/liyongning/blog/issues/42) 中讲解了 PDF 生成的技术背景、方案选型和决策，以及整个方案的技术架构图，所以后面的几篇一直都是在实现整套技术架构
- [PDF 生成（2）— 生成 PDF 文件](https://github.com/liyongning/blog/issues/43) 中我们通过 puppeteer 来生成 PDF 文件，并讲了自定义页眉、页脚的使用和其中的**坑**。本文结束之后 puppeteer 在 PDF 文件生成场景下的能力也基本到头了，所以，接下来的内容就全是基于 puppeteer 的增量开发了，也是整套架构的**核心**和**难点**
- [PDF 生成（3）— 封面、尾页](https://github.com/liyongning/blog/issues/44) 通过 PDF 文件合并技术让一份 PDF 文件包含封面、内容页和尾页三部分。
- [PDF 生成（4）— 目录页](https://github.com/liyongning/blog/issues/45) 通过在内容页的开始位置动态插入 HTML 锚点、页面缩放、锚点元素高度计算、换页高度补偿等技术让 PDF 文件拥有了包含准确页码 + 页面跳转能力的目录页
- [PDF 生成（5）— 内容页支持由多页面组成](https://github.com/liyongning/blog/issues/46) 通过多页面合并技术 + 样式沙箱解决了用户在复杂 PDF 场景下前端代码维护问题，让用户的开发更自由、更符合业务逻辑
- [PDF 生成（6）— 服务化、配置化](https://github.com/liyongning/blog/issues/47) 就是本文了，本系列的最后一篇，以服务化的方式对外提供 PDF 生成能力，通过配置服务来维护接入方的信息，通过队列来做并发控制和任务分类
- [代码仓库](https://github.com/liyongning/generate-pdf) **欢迎 Star**

---

**当学习成为了习惯，知识也就变成了常识**。 感谢各位的 **关注**、**点赞**、**收藏**和**评论**。

新视频和文章会第一时间在微信公众号发送，欢迎关注：[李永宁lyn](https://gitee.com/liyongning/typora-image-bed/raw/master/202202171742614.jpg)

文章已收录到 [github 仓库 liyongning/blog](https://github.com/liyongning/blog)，欢迎 Watch 和 Star。
