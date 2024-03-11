**当学习成为了习惯，知识也就变成了常识。** 感谢各位的 **关注**、**点赞**、**收藏**和**评论**。

新视频和文章会第一时间在微信公众号发送，欢迎关注：[李永宁lyn](https://gitee.com/liyongning/typora-image-bed/raw/master/202202171742614.jpg)

文章已收录到 [github 仓库 liyongning/blog](https://github.com/liyongning/blog)，欢迎 Watch 和 Star。

# 封面
![封面.png](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081241365.png)
# 回顾
前面我们在 [PDF 生成（1）— 开篇](https://github.com/liyongning/blog/issues/42) 讲了业务背景、技术调研、技术决策和整个方案的技术架构设计。知道了为什么做，也知道了最后的成果，接下来我们就进入实操阶段，带大家从零开始逐步实现整套架构。
# 简介
本文我们以 [百度新闻](https://news.baidu.com) 为例，讲解如何通过 puppeteer 将百度新闻页打印成一份完整的 PDF 文件。
# 构建项目

- 执行 `mkdir generate-pdf && cd generate-pdf && npm init -y` 命令，初始化项目，然后用 vscode 打开创建项目目录。
- 执行 `npm i puppeteer` 安装 puppeteer
- 分别创建 `/server`、`/fe` 两个目录来存放 Node 和 前端代码，项目目录结构如下：
![](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081241154.png)
# 生成 PDF 文件
创建 `/server/index.mjs` 文件，进行代码编写，这里我们以 [百度新闻](https://news.baidu.com) 为例，生成一份 PDF 文件。代码主要意思是：

- 以界面化模式打开一个浏览器（browser）
- 浏览器上新开一个 Tab 页（page)
- 当前 Tab 页打开 `https://news.baidu.com` 链接
- 调用 `page.pdf` 方法将当前页打印成 PDF 文件
- 关闭浏览器

代码如下：
```javascript
import puppeteer from "puppeteer";

/**
 * 生成 PDF 文件
 */
async function generatePDF() {
  // 启动浏览器。为了演示效果，暂时关闭无头模式，以浏览器界面形式运行
  const browser = await puppeteer.launch({ headless: false })
  // 打开一个新的 Tab 页
  const page = await browser.newPage()
  // 在当前 Tab 页上打开 “百度新闻” 页。第二个配置参数，意思是当页面触发 load 事件，并且 500ms 内没有新的网络连接，则继续往下执行
  await page.goto('https://news.baidu.com', { waitUntil: ['load', 'networkidle0']})
  // 将当前页打印成 PDF 文件
  await page.pdf({
    // PDF 文件的存储路径，如果不设置则会以二进制的形式放到内存中
    path: './news.pdf',
    // 以 A4 纸的尺寸来打印 PDF
    format: 'A4',
    // 设置 PDF 文件的页边距，避免内容完全贴边
    margin: {
      top: 40,
      right: 40,
      bottom: 40,
      left: 40
    },
    // 打印的时候打印背景色
    printBackground: true,
  })
  // 关闭浏览器
  await browser.close()
}

generatePDF()
```
PDF 效果如下：

![image.png](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081241484.png)
![image.png](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081241202.png)

短短的 10行 代码就能将一个现成的网页打印成一份 PDF 文件，是不是很简单。

仔细观察，会发现 PDF 文件的内容比网页的实际内容要少，这因为网页随着滚动会再动态加载一些内容（懒加载)

# 打印完整网页（网页滚动 — 懒加载场景）
这里我们只处理有限滚动场景，无限滚动虽然原理一样，但处理没有尽头，这块儿可以根据业务需要自行特殊处理，比如打印前 10屏。
这里用 **代码来模拟滚动**，让浏览器加载完整内容，核心代码如下：

![image.png](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081241786.png)

生成的 PDF 文件效果如下（为了节省篇幅，只截取了 开始和结尾 两页）

![image.png](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081241163.png)
![image.png](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081241226.png)

完整代码：

```javascript
import puppeteer from "puppeteer";

/**
 * 生成 PDF 文件
 */
async function generatePDF() {
  // 启动浏览器。为了演示效果，暂时关闭无头模式，以浏览器界面形式运行
  const browser = await puppeteer.launch({ headless: false })
  // 打开一个新的 Tab 页
  const page = await browser.newPage()
  // 在当前 Tab 页上打开 “百度新闻” 页。第二个配置参数，意思是当页面触发 load 事件，并且 500ms 内没有新的网络连接，则继续往下执行
  await page.goto('https://news.baidu.com', { waitUntil: ['load', 'networkidle0'] })
  // 滚动页面，加载完整内容。evaluate 的回调函数会在浏览器中执行，evalaute 方法的返回值是回调函数的返回值
  await page.evaluate(function () {
    return new Promise(resolve => {
      // 通过递归来滚动页面
      function scrollPage() {
        // { 浏览器窗口可视区域的高度，页面的总高度，已滚动的高度 }
        const { clientHeight, scrollHeight, scrollTop } = document.documentElement
        // 如果滚动高度 + 视口高度 < 总高度，则继续滚动，否则就任务滚动到底部了
        if (scrollTop + clientHeight < scrollHeight) {
          document.documentElement.scrollTo(0, scrollTop + clientHeight)
          // 加一个 setTimeout 来保证滚动的稳定性
          setTimeout(() => {
            scrollPage()
          }, 500)
        } else {
          resolve()
        }
      }
      scrollPage()
    })
  })
  // 将当前页打印成 PDF 文件
  await page.pdf({
    // PDF 文件的存储路径，如果不设置则会以二进制的形式放到内存中
    path: './news.pdf',
    // 以 A4 纸的尺寸来打印 PDF
    format: 'A4',
    // 设置 PDF 文件的页边距，避免内容完全贴边
    margin: {
      top: 40,
      right: 40,
      bottom: 40,
      left: 40
    },
    // 打印的时候打印背景色
    printBackground: true,
  })
  // 关闭浏览器
  await browser.close()
}

generatePDF()
```
# 页眉、页脚
我们经常能在 PDF 文件中看到页眉、页脚。页眉、页脚可以展示文件的作者、日期、版权、页码等信息，对于读者了解和阅读 PDF 文件有很大的帮助。那在当前技术方案下该如何为打印的 PDF 文件设置页眉、页脚呢？

puppeteer 的 `page.pdf` 方法提供了相应的配置参数。只需要一个 `displayHeaderFooter: true` 的配置项就可以

![image.png](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081242650.png)

效果如下：

![image.png](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081242267.png)

可以看到，页眉的左边是 PDF 文件生成的时间，中间位置是页面的 title，页脚的左边是当前页面的 URL，右边是当前页码/总页码。说实话，展示的效果还是不错的，但它的能力不止于此。

puppeteer 还提供了两个配置项，分别是 **headerTemplate** 和 **footerTemplate**，可以让使用者通过有效的 HTML 字符串来自定义页眉、页脚，并且其中还内置了一些特殊的变量，比如 date、title、url、pageNumber、totalPages，分别对应默认的页眉、页脚信息。

接下来我们实现如下效果的页眉、页脚：

![image.png](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081242863.png)

核心代码如下：

![image.png](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081242082.png)
![image.png](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081242423.png)

在实现页眉页脚时，需要注意如下内容：

- 所有内容都需要放在模版字符串中，不能从外部引入，比如 CSS、图片，可以看到 img 的 src 值是 base64 之后的内容
- 页眉天生会有 20px 的上边距，需要处理掉。如果不知道的话，会发现无法很难做到垂直居中，甚至看到页眉页脚空白
- 页脚天生会有 18px 的下边距，需要处理掉

完整代码：
```javascript
import puppeteer from "puppeteer";
import { footerTemplate, headerTemplate } from "./header-footer-template.mjs";

/**
 * 生成 PDF 文件
 */
async function generatePDF() {
  // 启动浏览器。为了演示效果，暂时关闭无头模式，以浏览器界面形式运行
  const browser = await puppeteer.launch({ headless: false })
  // 打开一个新的 Tab 页
  const page = await browser.newPage()
  // 在当前 Tab 页上打开 “百度新闻” 页。第二个配置参数，意思是当页面触发 load 事件，并且 500ms 内没有新的网络连接，则继续往下执行
  await page.goto('https://news.baidu.com', { waitUntil: ['load', 'networkidle0'] })
  // 滚动页面，加载完整内容。evaluate 的回调函数会在浏览器中执行，evalaute 方法的返回值是回调函数的返回值
  await page.evaluate(function () {
    return new Promise(resolve => {
      // 通过递归来滚动页面
      function scrollPage() {
        // { 浏览器窗口可视区域的高度，页面的总高度，已滚动的高度 }
        const { clientHeight, scrollHeight, scrollTop } = document.documentElement
        // 如果滚动高度 + 视口高度 < 总高度，则继续滚动，否则就任务滚动到底部了
        if (scrollTop + clientHeight < scrollHeight) {
          document.documentElement.scrollTo(0, scrollTop + clientHeight)
          // 加一个 setTimeout 来保证滚动的稳定性
          setTimeout(() => {
            scrollPage()
          }, 500)
        } else {
          resolve()
        }
      }
      scrollPage()
    })
  })
  // 将当前页打印成 PDF 文件
  await page.pdf({
    // PDF 文件的存储路径，如果不设置则会以二进制的形式放到内存中
    path: './news.pdf',
    // 以 A4 纸的尺寸来打印 PDF
    format: 'A4',
    // 设置 PDF 文件的页边距，避免内容完全贴边
    margin: {
      top: 40,
      right: 40,
      bottom: 40,
      left: 40
    },
    // 开启页眉、页脚
    displayHeaderFooter: true,
    // 通过 HTML 模版字符串自定义页眉、页脚
    headerTemplate: headerTemplate(),
    footerTemplate: footerTemplate(),
    // 打印的时候打印背景色
    printBackground: true,
  })
  // 关闭浏览器
  await browser.close()
}

generatePDF()
```
新建 `/server/header-footer-template.mjs` 文件
```javascript
/**
 * 页眉页脚
 * 需要注意的点：
 *    1. 所有内容都需要放在模版字符串中，不能从外部引入，比如 CSS、图片，可以看到 img 的 src 值是 base64 之后的内容
 *    2. 页眉天生会有 20px 的上边距，需要处理掉。如果不知道的话，会发现无法很难做到垂直居中，甚至看到页眉页脚空白
 *    3. 页脚天生会有 18px 的下边距，需要处理掉
 */
import crypto from 'crypto'

// 页眉
export function headerTemplate() {
  return `<div style="box-sizing: border-box; width: 100%; height: 40px; text-align: right; margin-right: 40px; margin-top: -20px; display: flex; justify-content: flex-end; align-items: center;">
    <img style="width: 83px; height: 16px;" src='data:image/png;base64,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'></img>
  </div>`
}

// 页脚
export function footerTemplate() {
  return `<div style="box-sizing: border-box; width: 100%; height: 40px; display: flex; justify-content: space-between; align-items: center; margin-bottom: -18px; padding: 0 40px; font-family: PingFangSC-Regular; font-size: 12px;">
    <div style="color: #fafafa;">${crypto.randomUUID()}</div>
    <div style="display: flex; justify-content: space-between; align-items: center; width: 70px; color: #666666;">
      <div><span>共</span> <span class="totalPages"></span> <span>页</span></div>
      <div class="pageNumber" style="font-family: PingFangSC-Semibold; font-weight: bold; color: #BFBFBF;"></div>
    </div>
  </div>`
}
```
效果如下：

![image.png](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081242306.png)

# 总结
本文我们以百度新闻页为例为大家展示了 puppeteer 的基本使用：

- 通过短短的 10行 代码将百度新闻页打印成一份 PDF 文件
- 通过 puppeteer 的 page.evaluate 方法为浏览器注入一段 JS 代码，用代码来模拟页面滚动，以解决懒加载的问题，从而保证 PDF 文件内容的完整性
- 通过自定义页眉、页脚的方式讲解了 puppeteer 中关于页眉、页脚相关选项的基本使用和其中的**坑**

随着本文的结束，基于 puppeteer 的 PDF 文件生成基本架子就搭起来了，而 puppeteer 在 PDF 文件生成场景下的能力也基本到头了，但现有内容在我们的技术架构中只是九牛一毛，所以，接下来的内容就全是基于 puppeteer 的增量开发了，也是整套架构的**核心**和**难点**。
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
