**当学习成为了习惯，知识也就变成了常识。** 感谢各位的 **关注**、**点赞**、**收藏**和**评论**。

新视频和文章会第一时间在微信公众号发送，欢迎关注：[李永宁lyn](https://gitee.com/liyongning/typora-image-bed/raw/master/202202171742614.jpg)

文章已收录到 [github 仓库 liyongning/blog](https://github.com/liyongning/blog)，欢迎 Watch 和 Star。

# 封面
![封面.png](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081258617.png)
# 回顾
[PDF 生成（2）— 生成 PDF 文件](https://github.com/liyongning/blog/issues/43) 我们以百度新闻页为例为大家展示了 puppeteer 的基本使用：

- 通过短短的 10行 代码将百度新闻页打印成一份 PDF 文件
- 通过 puppeteer 的 page.evaluate 方法为浏览器注入一段 JS 代码，用代码来模拟页面滚动，以解决懒加载的问题，从而保证 PDF 文件内容的完整性
- 通过自定义页眉、页脚的方式讲解了 puppeteer 中关于页眉、页脚相关选项的基本使用和其中的**坑**

文章最后也提到了 **puppeteer 在 PDF 文件生成场景下的能力基本到头了**，但现有内容在我们的技术架构中只是九牛一毛，所以，接下来的内容就全是基于 puppeteer 的增量开发了，也是整套架构的**核心**和**难点**。
# 问题
一份专业的 PDF 文件都会有自己的**封面**和**尾页**。在本文开始之前，大家先想想，基于现状如何为我们之前生成的 PDF 文件增加封面和尾页呢？比如

![image-20240308125818882](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081258927.png)

所以，本文的内容就是为我们在上文中生成的 PDF 文件增加封面和尾页。

# 分析
不知道大家是否还记得在 [PDF 生成（1）— 开篇](https://github.com/liyongning/blog/issues/42) 中的技术架构图，为什么架构图中的 PDF 生成服务会产出 3份 PDF 文件？带着问题接着往下看。

![image.png](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081250574.png)

假设前文中我们用的**百度新闻页**就是我们自己开发的一个页面，那在页面的开始和结束位置分别加上封面和尾页的 DOM，然后直接生成 PDF 文件，是不是就可以了？想想，这样做最简单了，一个页面搞定所有内容，比如：

![组 3 (1).png](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081253510.png)
但稍微一分析，就发现不行，因为我们我们在 `page.pdf` 方法中设置的 margin 属性和页眉、页脚是针对整个 PDF 文件的，但封面和尾页不需要边距和页眉、页脚。

**一个页面（URL)对应一份 PDF 文件，这是大方向，是由技术方案本身的特性所决定的**，因此封面和尾页不能和内容页放一起。

经过分析，结合架构图的指引，我们的实现思路是**一份完整的 PDF 文件至少应该包括三个页面 —— 封面页、内容页、尾页，每个页面对应一份 PDF 文件，最后将三份 PDF 合并成一份 PDF**，接下来就进入实战。

# 实战
前端页面的开发不是重点，所以这里我们就简单写了。
## 封面页 — /fe/cover.html
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Document</title>
  <style>
    body {
      width: 100%;
      height: 1123px;
      background: linear-gradient(173deg, #F5F8FF 20%, #E1E9FC 80%, rgba(225, 233, 252, 0) 86%);
      display: flex;
      justify-content: center;
      align-items: center;
    }
  </style>
</head>
<body>
  <h1>我是封面</h1>
</body>
</html>
```
## 尾页 — **/fe/last-page.html**
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Document</title>
  <style>
    body {
      width: 100%;
      height: 1123px;
      background: linear-gradient(173deg, #F5F8FF 20%, #E1E9FC 80%, rgba(225, 233, 252, 0) 86%);
      display: flex;
      justify-content: center;
      align-items: center;
    }
  </style>
</head>
<body>
  <h1>我是尾页</h1>
</body>
</html>
```
## PDF 生成服务 — **/server/index.mjs**
在 **/server/index.mjs** 中增加如下代码，用来生成封面和尾页的 PDF 文件

![image.png](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081253482.png)

截图中对应的代码如下：

```javascript
/* 省略之前的代码... */
  // 封面
  await page.goto('file:///Users/liyongning/studyspace/generate-pdf/fe/cover.html')
  await page.pdf({
    path: './cover.pdf',
    format: 'A4',
    printBackground: true
  })
  // 尾页
  await page.goto('file:///Users/liyongning/studyspace/generate-pdf/fe/last-page.html')
  await page.pdf({
    path: './last-page.pdf',
    format: 'A4',
    printBackground: true
  })
/* 省略之后的代码... */
```
生成的 PDF 效果如下：

![image.png](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081253670.png)
![image.png](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081254515.png)

解析来就是本文的重点了 — **PDF 文件合并**，因为我们最终交付的是一份 PDF 文件，而不是三份。

## PDF 文件合并
我们借助第三方库 `pdf-lib` 来完成 PDF 文件的合并。

- 首先安装 pdf-lib —— `npm i pdf-lib`
- 新建 `/server/merge-pdf.mjs` 文件来编写文件合并的代码

实现如下：
### /server/index.mjs：
![image-20240308125635526](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081256573.png)
![image.png](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081254262.png)

### **/server/merge-pdf.mjs：**
![image-20240308125603095](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081256128.png)

这里大家可能会有两个疑问点：

- 为什么不直接通过 Buffer.concat 合并内容，然后直接写盘，而是要通过 第三方库 先合并再写盘（page.pdf 的返回值是一个 Buffer 类型的数据）
- 为什么不新创建一份 PDF 文件，然后将三个文件合并到一起，或者是将内容页 PDF 的各个页面和尾页 PDF 的页面添加到封面 PDF 中，而是分别将封面 PDF 的页面和尾页 PDF 的页面插到内容 PDF 的对应位置

第一个问题的答案是：**数据格式问题**，虽然都是保存在内存中的二进制内容，但是 PDF 文件的二进制内容格式有点特殊，如果直接通过 `Buffer.concat` 将内容拼接，会发现拼接的内容就丢了，所以这里需要借助专门操作 PDF 文件的第三方库。当然了，如果是一个普通的文本文件，通过 `Buffer.concat` 完全没问题，有兴趣的话大家可以自己写个简单的 Demo。

至于第二个问题，答案是：不行，简单解释就是 —— 在当前的技术架构下，会导致目录页中目录项的页面跳转能力失效，目录页会用到 HTML 锚点，这些锚点被 pdf-lib 处理之后就失效了。具体内容在后面 [PDF 生成（4）— 目录页](https://github.com/liyongning/blog/issues/45) 详细讲解。

最终效果图如下：

![image.png](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081254294.png)
![image.png](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081255029.png)

PDF 文件合并（`/server/merge-pdf.mjs`)的完整代码如下：

```javascript
import { PDFDocument } from 'pdf-lib'

/**
 * 将三份 PDF 文件合并为一份
 *    另外三个参数的类型都是 Buffer，是表示 PDF 文件加载到内存后二进制内容
 * @param { Buffer } coverBuffer 封面 PDF
 * @param { Buffer } contentBuffer 内容页 PDF
 * @param { Buffer } lastPageBuffer 尾页 PDF
 * @returns 合并后的 PDF 文件的二进制内容
 */
export default async function mergePDF(coverBuffer, contentBuffer, lastPageBuffer) {
  // 通过 pdf-lib 加载现有的 3份 PDF 文档
  const { load } = PDFDocument
  const [coverPdfDoc, contentPdfDoc, lastPagePdfDoc] = await Promise.all([load(coverBuffer), load(contentBuffer), load(lastPageBuffer)])
  // 分别将封面文档和尾页文档的第一页拷贝到内容文档
  const [[coverPage], [lastPagePage]] = await Promise.all([contentPdfDoc.copyPages(coverPdfDoc, [0]), contentPdfDoc.copyPages(lastPagePdfDoc, [0])])
  // 将封面页插入到 内容文档 的第 0 页，即最开始的位置
  contentPdfDoc.insertPage(0, coverPage)
  // 将尾页添加到 内容文档 的最后一页
  contentPdfDoc.addPage(lastPagePage)
  // 将合并后的 内容文档 序列化为字节数组（Uint8Array），并以二进制的格式返回
  return Buffer.from(await contentPdfDoc.save())
}
```
# 总结
本文介绍了如何为通过 Puppeteer 生成的 PDF 文件添加封面和尾页，现在再来整体回顾一下：

- 首先，技术方案决定了一个页面对应一份 PDF 文件，这是大前提，因为 page.xx 方法的所有配置都是针对当前页的
- 在大前提下，我们通过 PDF 文件合并方案（pdf-lib），分别将封面 PDF、内容页 PDF 和尾页 PDF 三份文件合并为一份报告包含封面、内容页和尾页的完整 PDF

到这里，PDF 文件的整体框架已经基本形成（包括封面、内容页、尾页），但还有一点不完整，比如缺少**目录页**，一份完整的文件或文章怎么能没有目录呢？所以，接下来我们就讲 [PDF 生成（4）— 目录页](https://github.com/liyongning/blog/issues/45)。
# 链接

- [PDF 生成（1）— 开篇](https://github.com/liyongning/blog/issues/42) 中讲解了 PDF 生成的技术背景、方案选型和决策，以及整个方案的技术架构图，所以后面的几篇一直都是在实现整套技术架构
- [PDF 生成（2）— 生成 PDF 文件](https://github.com/liyongning/blog/issues/43) 中我们通过 puppeteer 来生成 PDF 文件，并讲了自定义页眉、页脚的使用和其中的**坑**。本文结束之后 puppeteer 在 PDF 文件生成场景下的能力也基本到头了，所以，接下来的内容就全是基于 puppeteer 的增量开发了，也是整套架构的**核心**和**难点**
- [PDF 生成（3）— 封面、尾页](https://github.com/liyongning/blog/issues/44) 通过 PDF 文件合并技术让一份 PDF 文件包含封面、内容页和尾页三部分。
- [PDF 生成（4）— 目录页](https://github.com/liyongning/blog/issues/45) 通过在内容页的开始位置动态插入 HTML 锚点、页面缩放、锚点元素高度计算、换页高度补偿等技术让 PDF 文件拥有了包含准确页码 + 页面跳转能力的目录页
- [PDF 生成（5）— 内容页支持由多页面组成](https://github.com/liyongning/blog/issues/46) 通过多页面合并技术 + 样式沙箱解决了用户在复杂 PDF 场景下前端代码维护问题，让用户的开发更自由、更符合业务逻辑
- [PDF 生成（6）— 服务化、配置化](https://github.com/liyongning/blog/issues/47) 就是本文了，本系列的最后一篇，以服务化的方式对外提供 PDF 生成能力，通过配置服务来维护接入方的信息，通过队列来做并发控制和任务分类
- [代码仓库](https://github.com/liyongning/generate-pdf) **欢迎 Star**

---

**当学习成为了习惯，知识也就变成了常识。** 感谢各位的 **关注**、**点赞**、**收藏**和**评论**。

新视频和文章会第一时间在微信公众号发送，欢迎关注：[李永宁lyn](https://gitee.com/liyongning/typora-image-bed/raw/master/202202171742614.jpg)

文章已收录到 [github 仓库 liyongning/blog](https://github.com/liyongning/blog)，欢迎 Watch 和 Star。
