**当学习成为了习惯，知识也就变成了常识。** 感谢各位的 **关注**、**点赞**、**收藏**和**评论**。

新视频和文章会第一时间在微信公众号发送，欢迎关注：[李永宁lyn](https://gitee.com/liyongning/typora-image-bed/raw/master/202202171742614.jpg)

文章已收录到 [github 仓库 liyongning/blog](https://github.com/liyongning/blog)，欢迎 Watch 和 Star。

# 封面
![封面.png](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081233902.png)
# 简介
本系列旨在介绍纯前端技术方案下的 PDF 生成最佳实践。内容涵盖业务背景、选型思路和实践历程，从简单的 PDF 文件生成到复杂的配置化与服务化。

整个实践过程以技术为驱动，同时也展示了如何打造技术产品的过程。是一份适合任何人实践的教程。

# 背景
需求来自业务对公司战略的拆解 — **安全运维托管服务**，为用户提供全日制的数字化资产安全运维、监控、告警、专家分析等服务。一句话总结就是，用户付钱找我们为用户提供全方位的资产运维服务。

在这个服务中我们为用户做了很多事情，我们需要让用户看到我们的价值，所以会以日报、周报的形式为用户推送**运维报告**，而这份报告就是以 **PDF 文件**的形式呈现。

所以，这份报告承载了产品能力和价值的传递，业务对 PDF 文件内容的展现提出了明确的要求：**需要呈现出色彩鲜明、精美的设计，简单描述就是好看 + 酷炫**。

于是，设计同学的设计稿就来了

> **本系列出现的所有和托管服务相关的配图版权均归 360 企业安全云所有**

![image (3) (1).png](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081234029.png)

看到设计稿的瞬间，就在想，这效果用 PDF 能呈现？最后会不会是这结果？

![image.png](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081235222.png)

**因此，业务需求可以归纳为一份出色、惊艳的 PDF 文件**。

# 技术调研
讲了业务背景，接下来就该技术调研了，经过调研，PDF 文件生成可以总结为两大类：原生方案和转化方案。
## 原生方案
利用开源工具库直接操作 PDF 文件，在文件内绘制内容，比如 iText、PDFKit、pdf-lib。

- **优点**，性能高，适用于内容简单的场景
- **缺点**，难以处理具有复杂排版和样式的场景
## 转化方案
将内容通过中间媒介转化成 PDF 文件，主要包括：Word 转 PDF、HTML/CSS 转 PDF。

**Word 转 PDF** 的缺点和原生方案一样，在复杂排版和样式场景上有心无力。大概原理是通过 Word 提供的 API 操作编写 Word 文档，然后 Word 转换成 PDF 文件。

**HTML/CSS 转 PDF**，主要有如下三种方案：

- **模版引擎**，利用模版引擎生成 HTML/CSS，然后结合下面的两个方案生成 PDF 文件，一般后端同学会用这个方案
- **Canvas**，前端常用的方案，例如 html2canvas + jsPDF，但在 PDF 分页、内容截断问题上难以解决，PDF 目录页不支持页面跳转和展示页码
- **浏览器打印系统**，利用浏览器的布局、渲染、打印能力，通过 DevTools 协议控制 Chrome/Chromiun，实现 PDF 文件的打印，即 chrome 浏览器**右键 -> 打印**的自动化版本
# 技术决策
经过调研和众多方案的分析，最终我们选择了**浏览器打印系统**方案，具体的实现上我们选择了 [Puppeteer](https://pptr.dev/) 框架，它是一个 Node.js 库，提供高级 API 控制 Chrome/Chromiun 浏览器，我们在浏览器中手动执行的大多数操作它都可以完成，例如执行 **page.pdf** 方法即可将当前渲染的页面打印成 PDF 文件，简单易用。

**为什么选择基于浏览器打印系统的 puppeteer 方案？**

- 经过方案调研之后的综合对比，基于浏览器打印系统的方案更符合业务的诉求
- 我们是前端团队，这套方案更符合团队的技术栈
- 人力和时间成本，其他几个方案基本上就是只能服务端同学自己做，前端很难参与进去，对服务端团队的研发资源造成压力，影响部分业务的吞吐率

这套方案前后端同学各司其职、通力合作，分别做自己擅长的事。服务端同学开发页面接口供前端同学调用，前端同学负责开发酷炫的页面，PDF 生成服务将前后端同学开发的页面转成 PDF 文件

![image.png](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081235614.png)

于是，产品和设计同学就可以在这张静态的 A4 纸上尽情发挥，不受技术限制。

# 技术架构
![image.png](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081235679.png) 

方案的技术架构，分为三大块，分别是：

- **接入方**，即 PDF 生成服务的调用方，就是一个普通的 Web 项目（前端 + 后端）
- **PDF 生成服务**，对外暴露 API，一次 API 调用，产出一份 PDF 文件的下载地址
- **配置服务**，维护接入方的信息，为 PDF 生成服务提供必要的配置信息，比如接入方 Web 项目的页面地址，PDF 生成服务会负责将这些页面生成 PDF 文件

整体执行流程如下：

![image.png](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081235300.png)

- **接入方**，带着分配的 APP ID 和 其它参数调用生成 PDF 服务的 API 接口，**其它参数**是接入方前后端自己需要用到的参数，调用时提供的所有参数会原封不动的通过 URL 查询参数带到接入方的前端页面地址上
- **PDF 生成服务**
   - 接收到请求后，将请求放入队列
   - 监听到队列有内容进入，通知生成 PDF 文件的模块，启动 PDF 生成任务
   - 任务拿着 APP ID 请求**配置服务**，获取到对应的配置信息
   - 任务将配置信息中指定的所有页面打印成 PDF 文件
   - 将 PDF 文件上传到智慧云（S3）上，并将 PDF 文件的下载地址通过回调接口回传给接入方
# 总结
到这里本文就结束了，本文主要讲了如下内容：

- 业务背景，要求技术能够产出一份**漂亮 + 酷炫**的 PDF 文件
- 技术调研，主要分为原生方案和转化方案
- 技术决策，结合业务诉求、各个方案的优缺点、团队技术栈和部门人力、时间成本，最终选择**基于浏览器打印系统的 puppeteer 方案**
- 整个方案的技术架构设计

一个完善的技术架构是随着业务持续迭代而产生的，接下来我们将从零开始逐步实现整套架构，因此这是一份适合任何人实践的教程
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
