**当学习成为了习惯，知识也就变成了常识。** 感谢各位的 **关注**、**点赞**、**收藏**和**评论**。

新视频和文章会第一时间在微信公众号发送，欢迎关注：[李永宁lyn](https://gitee.com/liyongning/typora-image-bed/raw/master/202202171742614.jpg)

文章已收录到 [github 仓库 liyongning/blog](https://github.com/liyongning/blog)，欢迎 Watch 和 Star。

**[更多精彩内容](https://github.com/liyongning/blog/blob/main/README.md)**

<hr />

# 简介

Chrome DevTools Protocol（CDP）允许工具对 Chromium、Chrome 和其他基于 Blink 的浏览器进行检测、检查、调试和分析。有许多[项目](https://github.com/ChromeDevTools/awesome-chrome-devtools)都在使用，比如我们最常见的 [Chrome DevTools](https://developers.google.com/web/tools/chrome-devtools/)，并且 CDP 的 API 是由该团队在维护。

CDP 分为多个域（Domain），比如 DOM、Element、Network、Debugger 等，我们平时在 DevTools 面板中看到的每个 Tab 都包含一个或多个域。每个域定义了一些它支持的命令和事件。这些命令和事件都是固定结构的序列化 JSON 对象。

# Protocol Monitor

通过 Protocol Monitor 面板可以很方便的了解到 DevTools 是如何使用 CDP 的，你可以很直观的看到所有的请求和响应方法。

如何打开 Protocol Monitor？如果你是第一次打开，请按照如下步骤操作：

1. 打开 DevTools 面板
2. 点击面板右上角的设置按钮，进入设置面板
3. 选择侧边栏中的 Experiments（实验） 菜单，在右侧面板的 Filter 输入框中输入 Protocol Monitor，选中找到的选项
4. 关闭当前 DevTools 面板，并再次打开，以后就可以直接从下一步直接开始了
5. 点击 DevTools 面板，点击右上角的三个点，并选择 Moro Tools 菜单，然后从弹出的菜单中选择 Protocol monitor 菜单，即可打开 Protocol Monitor 面板

![image-20240714203026153](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202407142034795.png)

![image-20240714203411677](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202407142034708.png)

能看到面板中显示着已经发出的一系列命令

![image-20240714204133498](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202407142041530.png)

可以看到面板的底部有一个按钮和输入框，我们可以在这里发出自己的命令，如果你对 CDP 很熟，可以直接在输入框中输入 CDP 命令，如果不熟悉可以点击输入框左侧的按钮，左侧会打开一个可视化的命令编辑器，你可以选择输入自己想要操作的命令，比如查看 Cookie 信息，点击下方的发送按钮，即可完成操作。

![image-20240714205910442](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202407142059475.png)

当然，你也可以在控制台中（Console）输入如下内容进行操作，同样的效果（当然你会看到好几条其他命令），另外有个前提是能正常加载到 main.js 文件

```javascript
const main = await import('./devtools-frontend/front_end/entrypoints/main/main.js')
await main.MainImpl.sendOverProtocol('Network.getCookies', {})
```

 # CDP 在 Chrome 扩展中的使用

浏览器扩展程序可以使用 `chrome.debugger` API 与 Chrome DevTools 进行交互，这个 API 通过 JSON 格式的消息与 Chrome DevTools 进行通信，以发送命令和接收响应。

`chrome.debugger` 的一个重要应用场景是为 Web IDE 提供调试功能， `chrome.debugger` API 可以在 Web IDE 中实现类似于Chrome DevTools 的调试功能，如设置断点、控制台求值、实时编辑等。但`chrome.debugger` 为 Web IDE 实现的调试工具和浏览器自带的 DevTools 是冲突的，当你在扩展中使用 `chrome.debugger` 建立了调试连接时，如果打开浏览器自带的DevTools，这个连接会断开。这意味着你不能同时使用这两者

# 能力

一句话总结就是：能手动在 devTools 中完成的操作，通过 CDP 都能做，而且能做的更细致。其本质是浏览器将内部发生的行为通过特定的通信渠道（比如 Socket）以 CDP 协议的格式暴露出来，然后我们可以将这些协议再写入另一个 devTools 工具。

浏览器默认是不开启该能力的，可以通过下面的命令启动浏览器，并指定远程调试端口，我们就可以以 CDP 协议通过该端口和浏览器通信了，比如导航到特定的页面，获取当前渲染的 DOM 信息等。

```shell
/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome --remote-debugging-port=9222 --user-data-dir=./tmp
```

# 应用场景

* 基于 CDP 协议的爬虫，比如前端耳熟能详的 puppeteer
* chrome-remote-interface 库，相对于 puppeteer 来说更底层，是对 CDP 协议的封装
* 远程调试，场景是：在本地调试用户侧的代码，可以完整的利用用户侧的环境，更容易复现问题
  * 开源的 [devtools-remote-debugger](https://github.com/Nice-PLQ/devtools-remote-debugger)，这是一个很优秀的库，它通过前端 SDK 的形式拦截系统的 DOM、Console、Network、Source 等 Domain 的数据，并将这些数据组装成 CPD 协议对应的数据结构，通过 WebSocket 发送到服务端，服务端再将数据写入本地的 DevTools，可以很好的复现（观察）到用户侧发生了什么。但它的缺点是无法调试，比如你看到了 Console 中的代码报错，但是你无法在用户侧的 Source 中打断点
  * 自研方案，提供一个一键开启用户侧的浏览器调试能力的工具，好处是不需要侵入项目代码，这个方案能弥补上一个方案远程写入能力，可以完整的控制用户浏览器行为，也能将用户浏览器的行为同步到本地，比如页面切换。但它的缺点是无法完整复现用户侧的异常，你能将用户侧的一切同步回来，比如 Console 的异常，但可能的结果是：你在 Console 看到了用户侧的报错信息，但是你本地却没有发生异常，因为你本地的 UI 渲染、JS 执行等全部走的是你本地的环境

调研 DevTools 是想将浏览器的远程调试能力应用在业务中，但目前的结论是不可行。发现有一些讲解远程调试的博客，基本上都只能做到问题重放，没办法做到实际的调试，比如打断点。

但 CDP 能力确实很强大，值得持续探索和尝试。

<hr />

**当学习成为了习惯，知识也就变成了常识。** 感谢各位的 **关注**、**点赞**、**收藏**和**评论**。

新视频和文章会第一时间在微信公众号发送，欢迎关注：[李永宁lyn](https://gitee.com/liyongning/typora-image-bed/raw/master/202202171742614.jpg)

文章已收录到 [github 仓库 liyongning/blog](https://github.com/liyongning/blog)，欢迎 Watch 和 Star。

**[更多精彩内容](https://github.com/liyongning/blog/blob/main/README.md)**
