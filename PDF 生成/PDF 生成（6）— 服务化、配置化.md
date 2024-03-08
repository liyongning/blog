**当学习成为了习惯，知识也就变成了常识。** 感谢各位的 **关注**、**点赞**、**收藏**和**评论**。

新视频和文章会第一时间在微信公众号发送，欢迎关注：[李永宁lyn](https://gitee.com/liyongning/typora-image-bed/raw/master/202202171742614.jpg)

文章已收录到 [github 仓库 liyongning/blog](https://github.com/liyongning/blog)，欢迎 Watch 和 Star。

# 封面
![封面.png](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081316288.png)
# 回顾
前面我们分别通过 [PDF 生成（1）— 开篇](https://github.com/liyongning/blog/issues/42)、[PDF 生成（2）— 生成 PDF 文件](https://github.com/liyongning/blog/issues/43)、[PDF 生成（3）— 封面、尾页](https://github.com/liyongning/blog/issues/44)、[PDF 生成（4）— 目录页](https://github.com/liyongning/blog/issues/45)、[PDF 生成（5）— 内容页支持由多页面组成](https://github.com/liyongning/blog/issues/46) 五篇来讲解 PDF 生成的整个方案，到目前为止，整套方案基本完成了：

- 我们通过 PDF 文件合并技术让一份 PDF 文件包含封面、内容页和尾页三部分
- 通过在内容页的开始位置动态插入 HTML 锚点、页面缩放、锚点元素高度计算、换页高度补偿等技术让 PDF 文件拥有了包含准确页码 + 页面跳转能力的目录页
- 通过多页面合并技术 + 样式沙箱解决了用户在复杂 PDF 场景下前端代码维护问题，让用户的开发更自由、更符合业务逻辑

至此，PDF 生成的能力齐了，但怎么给用户使用呢？这就是本文要解决的问题了。
# 简介
前面我们花了大量的精力来完善整个 PDF 生成方案，现在从 PDF 生成角度来说，能力已经齐备，但整个服务以及相关配置都运行在本地，没办法直接给用户使用。

所以本文我们就将 PDF 生成能力通过**服务化**暴露给用户，相关资源**配置化**来适配不同的用户。

# 服务化
通过为项目引入 Koa 框架来对外提供服务。

- 安装 koa 和 @koa/router，`npm i koa @koa/router`
- 新建`/server/koa-server.mjs`文件

**/server/koa-server.mjs：**
```javascript
import Koa from 'koa'
import KoaRouter from '@koa/router'
import { generatePDF } from './index.mjs'

const app = new Koa()
const router = new KoaRouter()

// 当用户请求 http://localhost:3000 时，触发 generatePDF() 函数生成 PDF 文件
router.get('/', function(ctx) {
  generatePDF()

  ctx.body = {
    errno: 0,
    data: [],
    msg: '正在生成 PDF 文件'
  }
})

app.use(router.routes())

app.listen(3000, () => {
  console.log('koa-server start at 3000 port')
})
```
**/server/index.mjs** 导出 generatePDF 方法

![image.png](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081316897.png)

通过 node 或 nodemon 执行 **/server/koa-server.mjs**，然后在浏览器直接访问`http://localhost:3000`会看到 PDF 生成服务开始运行，并生成 PDF 文件。这样，我们的 PDF 生成能力就实现了对外的**服务化**。

# 配置化
目前可以发现，PDF 文件的目录页配置、前端页面的 URL 等信息都是写死在代码中的，我们需要将这些信息以接入方为维度进行统一维护，并以服务的形式暴露给 PDF 生成服务。

- 安装 axios，用来请求**配置服务**，`npm i axios`
- 分别对`/server/koa-server.mjs`和`/server/index.mjs`进行如下改造

**/server/koa-server.mjs**
```javascript
import Koa from 'koa'
import KoaRouter from '@koa/router'
import { generatePDF } from './index.mjs'
import axios from 'axios'

const app = new Koa()
const router = new KoaRouter()

// 当用户请求 http://localhost:3000 时，触发 generatePDF() 函数生成 PDF 文件
router.get('/', async function (ctx) {
  const appId = ctx.query.appId
  const { data: configData } = await axios.get(`http://localhost:3000/get-pdf-config?appId=${appId}`)
  // 异常情况
  if (configData.errno) {
    ctx.body = configData
    return
  }

  const { data } = configData
  generatePDF(data)

  ctx.body = {
    errno: 0,
    data: [],
    msg: '正在生成 PDF 文件'
  }
})

// 获取指定 appId 所对应的配置信息
router.get('/get-pdf-config', function (ctx) {
  const pdfConfig = {
    // 为接入方分配唯一的 uuid
    '59edaf80-ca75-8699-7ca7-b8121d01d136': {
      name: 'PDF 生成服务测试',
      // 目录页配置
      dir: [
        { title: '锚点 1', id: 'anchor1' },
        { title: '锚点 2', id: 'anchor2' },
        { title: '第二个内容页 —— 锚点 1', id: 'second-content-page-anchor1' },
        { title: '第二个内容页 —— 锚点 2', id: 'second-content-page-anchor2' },
      ],
      // 接入方的前端页面链接
      pageInfo: {
        // 封面
        "cover": "file:///Users/liyongning/studyspace/generate-pdf/fe/cover.html",
        // 内容页
        "content": [
          "file:///Users/liyongning/studyspace/generate-pdf/fe/exact-page-num.html",
          "file:///Users/liyongning/studyspace/generate-pdf/fe/second-content-page.html"
        ],
        // 尾页
        "lastPage": "file:///Users/liyongning/studyspace/generate-pdf/fe/last-page.html"
      },
      // ... 还可以增加其他配置
    }
  }

  const appId = ctx.query.appId || ''
  if (!pdfConfig[appId]) {
    ctx.body = {
      errno: 100,
      data: [],
      msg: '无效的 appId，请联系服务提供方申请接入'
    }
    return
  }

  ctx.body = {
    errno: 0,
    data: pdfConfig[appId],
    msg: 'success'
  }
})

app.use(router.routes())

app.listen(3000, () => {
  console.log('koa-server start at 3000 port')
})
```
增加了配置服务`/get-pdf-config`，并在 PDF 生成服务中调用，获取配置内容，并将配置内容传递给了`generatePDF`方法。

**/server/index.mjs**

![image.png](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081317990.png)
![image.png](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081317982.png)
![image.png](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081317431.png)

PDF 的目录页配置、封面、内容页、尾页均改成了使用配置服务传递过来的数据，我们在浏览器访问`http://localhost:3000/?appId=59edaf80-ca75-8699-7ca7-b8121d01d136`即可生成 PDF 文件，如果访问时没有传 appId 会收到异常提示：

![image.png](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081318451.png)

好了，配置化就讲到这里了，就像代码中提到的一样，所有和配置相关的信息都可以通过配置服务来维护，可根据自己的需求来进行扩充。

# 并发控制（队列）
现在我们的 PDF 生成能力以服务的形式对外提供，并通过配置服务来维护接入方信息。经过一段时间的推广后，接入的用户越来越多，服务的调用量越来越大，这时候就会遇到服务稳定性的问题。

每个请求我们都会启动一个浏览器，一台 2核 4G内存的机器，三四个并发基本上就超负荷运行了，如果同时有更多的请求过来，直接就宕机了。所以，我们需要为服务增加一个并发控制。思路如下：

- 给服务增加一个任务队列，这个队列可以通过 kafka 实现，也可以通过 redis 来实现，最差也可以程序自己维护一个单机版的内存队列（不推荐）
- 每个请求进来时，先入队
- 当队列中监听到有任务存在时，从队列中取出一个任务然后执行，这个取任务的频率可以由程序自己控制

这样，程序就不会因为请求量过大，而导致机器宕机。基于队列我们也可以做任务失败重试。
## 任务分类
服务又稳定的运行了一段时间，有天又收到了一个接入申请，这个接入方的使用场景是不定期的生成几千几万份报告，然后将这些报告打包发给销售，让销售进一步跟进用户。

这个需求很合理，但是会对我们现有的服务造成影响，试想，如果这个任务一旦启动，短时间就会在队列中堆积几万个待执行的任务，要消费完这些任务可能需要好几个小时甚至一整天，这会影响其他任务的执行，后入队的任务一直排在队尾，迟迟得不到执行。

这时候，我们就需要对任务进行分类，将任务分为实时和非实时，实时任务进入实时队列，非实时任务进入非实时队列，程序有优先消费实时队列中的任务，当实时队列为空时去消费非实时队列的任务，当两个队列都为空时，程序停止。

# 其他优化
本系列的重点是演示 PDF 生成的核心思路和逻辑，所以有些地方的代码写的比较简单，比如没有做很好的模块化拆分、异常处理等，但这些完全不影响对整体架构的理解。

技术架构中我们还有一些能力没有实现，比如：

- PDF 文件上传 S3，并将下载地址回传给接入方
- 服务的安全校验，可以设置复杂的校验，也可以通过简单的参数签名来做，根据使用场景来决定
- 告警推送，比如 PDF 文件生成异常告警、PDF 文件下载链接推送入群或发给个人等

剩下的这些功能都依赖一些内网服务，所以这里就没有一一演示了，只提供一些思路，大家可以根据自己的实际情况有选择性的学习和迭代。

# 部署问题

项目开发结束后，一般都需要部署到服务器上，这时候你可能会遇到一些困难，比如：

* 启动项目后，会发现有如下报错，其原因是服务器上缺少相关安装包，具体可查看 [鼓掌排除](https://pptr.nodejs.cn/troubleshooting) 下的 **Chrome 无法在 linux 上启动**

![img](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081836545.png)

* 如果遇到如下错误，是因为 nss 库版本过低，可通过 `rpm -q nss` 命令查看已安装的库信息，然后使用 `yum update nss` 进行升级

![](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081841798.png)

* 这会儿，服务应该起来了，但执行的时候发现又报错了，这时候需要禁用沙箱，可以查看  [鼓掌排除](https://pptr.nodejs.cn/troubleshooting) 的 **设置 Chrome Linux 沙箱**

![image.png](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081843576.png)

* 这时候 PDF 终于生成了，但可能会发现 — **乱码了**，这是字符集问题，即服务器上没有对应的字体库，具体操作参考下面的**字体库**章节

![img](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081846020.png)

## 字体库

如果生成的 PDF 文件出现了乱码问题，是因为服务器缺少字体库文件，我们需要为服务器增加相应的字体。比如我们使用的是 PingFang 和思源黑体，去找设计同学要一份字体文件，然后拷贝到 `/usr/share/fonts` 目录下，其中涉及到如下命令：

* `fc-list | grep 'PingFang SC'` 查看是否有该字体库
* 字体库配置文件 `/etc/fonts/fonts.config`，打开后发现，这就是为什么把新增字体文件放 `/usr/share/fonts` 目录的原因

![img](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202403081852909.png)

* 新增字体文件后执行 `fc-cache -f -v` 清空字体缓存，并会生成新的字体缓存

# 总结
到这里，本文就结束了，我们来简单总结一些：

- 我们通过 Koa 框架，将 PDF 生成能力以服务的形式对外暴露
- 通过配置化服务来维护接入方的一些信息，比如业务名称、目录页的配置、PDF 文件封面、内容页、尾页对应的 URL 等，配置化服务配置的内容有很多，根据场景自行扩充
- 通过队列来做并发控制，保证服务的稳定性
- 通过对任务进行分类（实时和非实时），来保证实时任务的及时消费，非实时任务的稳定消费
- 最后给大家提了一些其他可迭代的点，比如文件上传、下载地址回传、服务安全校验、告警推送等
# 系列总结
如果你完整的阅读了整个系列，那么首先应该为自己鼓掌，毕竟又是成长的一段时间，另外一定要进行实操，光看不实践，学习效果还是会打一定的折扣。接下来我们就对本系列进行一个简单的回顾总结：

- 首先我们在 [PDF 生成（1）— 开篇](https://github.com/liyongning/blog/issues/42) 中讲解了 PDF 生成的技术背景、方案选型和决策，以及整个方案的技术架构图，所以后面的几篇一直都是在实现整套技术架构
- [PDF 生成（2）— 生成 PDF 文件](https://github.com/liyongning/blog/issues/43) 中我们通过 puppeteer 来生成 PDF 文件，并讲了自定义页眉、页脚的使用和其中的**坑**。本文结束之后 puppeteer 在 PDF 文件生成场景下的能力也基本到头了，所以，接下来的内容就全是基于 puppeteer 的增量开发了，也是整套架构的**核心**和**难点**
- [PDF 生成（3）— 封面、尾页](https://github.com/liyongning/blog/issues/44) 通过 PDF 文件合并技术让一份 PDF 文件包含封面、内容页和尾页三部分。
- [PDF 生成（4）— 目录页](https://github.com/liyongning/blog/issues/45) 通过在内容页的开始位置动态插入 HTML 锚点、页面缩放、锚点元素高度计算、换页高度补偿等技术让 PDF 文件拥有了包含准确页码 + 页面跳转能力的目录页
- [PDF 生成（5）— 内容页支持由多页面组成](https://github.com/liyongning/blog/issues/46) 通过多页面合并技术 + 样式沙箱解决了用户在复杂 PDF 场景下前端代码维护问题，让用户的开发更自由、更符合业务逻辑
- [PDF 生成（6）— 服务化、配置化](https://github.com/liyongning/blog/issues/47) 就是本文了，本系列的最后一篇，以服务化的方式对外提供 PDF 生成能力，通过配置服务来维护接入方的信息，通过队列来做并发控制和任务分类
- [代码仓库](https://github.com/liyongning/generate-pdf) **欢迎 Star**

感谢大家花时间阅读，希望大家能从本系列学到对自己有用的知识，不论是 PDF 生成本身，还是整个思考迭代过程，亦或者是其中的某些点。

---

**当学习成为了习惯，知识也就变成了常识。** 感谢各位的 **关注**、**点赞**、**收藏**和**评论**。

新视频和文章会第一时间在微信公众号发送，欢迎关注：[李永宁lyn](https://gitee.com/liyongning/typora-image-bed/raw/master/202202171742614.jpg)

文章已收录到 [github 仓库 liyongning/blog](https://github.com/liyongning/blog)，欢迎 Watch 和 Star。

