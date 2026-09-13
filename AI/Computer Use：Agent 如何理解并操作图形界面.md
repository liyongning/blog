# Computer Use：Agent 如何理解并操作图形界面

**当学习成为了习惯，知识也就变成了常识。** 感谢各位的 **关注**、**点赞**、**收藏**和**评论**。

新视频和文章会第一时间在微信公众号发送，欢迎关注：[李永宁lyn](https://gitee.com/liyongning/typora-image-bed/raw/master/202202171742614.jpg)

文章已收录到 [github 仓库 liyongning/blog](https://github.com/liyongning/blog)，欢迎 Watch 和 Star。

**[更多精彩内容](https://github.com/liyongning/blog/blob/main/README.md)**

<hr />

如果一个系统提供了 API，Agent 可以直接发送结构化请求；如果任务是构建、测试或处理文件，Agent 可以调用 CLI。还有一类任务没有这样的入口：人可以打开应用、点击按钮、填写表单并完成工作，但系统只把这些能力放在图形界面里。

Computer Use 解决的就是这段“最后一公里”。它是 Agent 面向图形界面的工具能力：模型获得当前界面状态，提出输入动作，执行程序把动作交给浏览器、桌面应用或操作系统，再把变化后的状态返回给模型。

Computer Use 描述的是一种能力；Computer Use Tool 是这种能力的具体工具实现；Agent Loop 是组织工具调用的上层决策循环。三者不应混为一谈。不同实现可以只使用截图，也可以接入 Accessibility State、浏览器接口或其他结构化信息。

## 1. 先看一次完整运行

用浏览器提交一份表单，可以把整个过程压缩成下面这条链路：

~~~mermaid
sequenceDiagram
    participant U as 用户
    participant A as Agent Loop
    participant T as Computer Use Tool
    participant H as 执行适配器
    participant B as 浏览器与网页
    U->>A: 请提交表单并确认成功
    A->>T: 获取当前界面状态
    T->>H: 请求截图或语义状态
    H->>B: 读取当前页面
    B-->>H: 页面状态
    H-->>T: 状态结果
    T-->>A: 截图或 Accessibility State
    A->>T: 点击“提交”
    T->>H: 执行点击动作
    H->>B: 发送输入事件
    B-->>H: 页面发生变化
    H-->>T: 返回更新后的状态
    T-->>A: 新截图或语义状态
    A->>A: 判断是否提交成功
~~~

这里每个角色承担不同责任：

| 角色 | 负责什么 |
|---|---|
| Agent Loop | 理解目标、选择工具、决定下一步、验证结果 |
| Computer Use Tool | 暴露观察和动作接口 |
| 执行适配器 | 将工具请求转换为平台调用，并采集结果 |
| 浏览器或桌面应用 | 承载真实的界面和业务状态 |
| 权限与策略 | 控制可访问的应用、数据和高影响动作 |

执行适配器不一定是独立服务；它可以和工具实现放在同一个程序里。这里区分的是职责，不是固定的部署方式。

### 1.1 抽象的请求与响应

下面的消息只用于说明数据流，不对应某个产品的固定协议：

第一次，Agent 请求观察：

~~~json
{
  "tool_call": {
    "action": "screenshot"
  }
}
~~~

工具让执行适配器捕获当前页面，并返回：

~~~json
{
  "tool_result": {
    "type": "screenshot",
    "image": "<current screen>"
  }
}
~~~

模型看到页面后，提出动作：

~~~json
{
  "tool_call": {
    "action": "click",
    "target": {
      "description": "提交按钮",
      "x": 842,
      "y": 516
    }
  }
}
~~~

执行器完成点击，再返回更新后的状态：

~~~json
{
  "tool_result": {
    "type": "screenshot",
    "image": "<updated screen>"
  }
}
~~~

如果新状态出现“提交成功”，任务才算完成。工具返回“点击已执行”，不等于业务已经成功。

这就是 Computer Use 的基本闭环：

~~~text
观察 → 理解 → 规划 → 执行 → 再观察 → 验证
~~~

失败时，Agent 可能等待页面稳定、重新定位元素、换用语义动作，或请求人工接管。连续执行一串未经验证的坐标动作，会让一个小误差扩散到后续步骤。

## 2. 界面状态从哪里来

观察方式决定了 Agent 能知道什么。最常见的两条路径是 Screenshot 和 Accessibility State。

~~~mermaid
flowchart TB
    G[当前图形界面] --> S[Screenshot：像素快照]
    G --> X[Accessibility State：语义状态]
    S --> V[视觉模型理解]
    X --> L[角色、名称、状态和动作]
    V --> C[坐标或视觉判断]
    L --> E[元素定位和语义动作]
    C --> A[执行器]
    E --> A
~~~

两条路径可以独立使用，也可以结合使用：截图帮助模型理解最终画面，语义状态帮助它精确找到可操作控件。

### 2.1 Screenshot：看见最终画面

Screenshot 是某个时刻的屏幕或窗口像素。视觉模型可以从中读取文字、识别按钮、判断空间关系、检查颜色和布局，并估计目标区域。

~~~text
截图 → 视觉理解 → 定位目标区域 → 坐标点击或视觉验收
~~~

它的覆盖面很广，不要求应用提供控件语义。图片、Canvas、图表和渲染异常都能被观察到。缺点也很直接：分辨率、显示器缩放、窗口移动、遮挡、动画和弹窗都可能让原来的位置失效。

因此，像素操作不只是“算一个坐标”。执行器还要处理窗口和屏幕坐标系，Agent 要在界面变化后重新获取状态。截图也可能陈旧：截图完成后，页面立刻重排或弹出对话框，之前的判断就不再适用。

### 2.2 Accessibility State：按语义理解控件

本文把不同平台提供的可访问性语义及其状态统称为 Accessibility State。这不是跨平台统一协议。macOS、Windows、Android、浏览器和自动化框架的字段、树结构和操作接口各不相同；Accessibility Tree 是其中一种常见的树形组织方式。

一个抽象节点可能包含：

~~~text
角色：按钮
名称：提交
状态：已启用
可用动作：激活
~~~

Agent 可以用“名称为提交、角色为按钮”定位目标，而不必猜坐标。语义还会告诉它控件是否启用、是否展开或是否被选中。原生控件通常有基础语义，自定义控件则要由应用补充名称、角色、状态和动作。

Accessibility State 并不能替代截图。它未必包含颜色、图片、精确布局和最终渲染结果；这些内容仍需要通过视觉观察验证。

### 2.3 浏览器：把两种观察方式放在同一个案例中

浏览器本身是一种桌面应用，但它还负责处理网页的 DOM、HTML 语义、ARIA 和渲染状态。浏览器可能据此构建内部可访问性树，再通过平台接口或浏览器专用接口提供给自动化工具。最终进入可访问性树的内容取决于元素语义、可见性规则和浏览器实现，不是 DOM 的完整副本。

~~~mermaid
flowchart LR
    D[DOM、HTML 原生语义、ARIA] --> C[浏览器语义计算]
    R[可见性与交互状态] --> C
    C --> T[浏览器内部 Accessibility Tree]
    T --> O[平台 Accessibility API]
    T --> P[浏览器自动化接口]
    D --> Q[DOM 查询与脚本接口]
~~~

对“点击提交”这个任务，可以有两条 Computer Use 观察路径。

第一条只依赖截图：

~~~text
获取截图
→ 找到写着“提交”的按钮
→ 估计按钮中心坐标
→ 点击
→ 获取新截图
→ 检查成功提示
~~~

第二条使用语义状态：

~~~text
获取 Accessibility State
→ 找到 role=button、name=提交
→ 调用激活动作
→ 重新获取状态
→ 检查 status=提交成功
~~~

在真实系统中，两者往往配合使用：语义状态负责定位，截图负责确认页面是否真的呈现了预期结果。若网页缺少稳定语义，系统可能退回截图；若视觉结果复杂，语义状态又不足以完成验收。

网页语义化的作用也在这里体现出来。使用原生按钮、输入框和标签，通常能让浏览器生成更清晰的基础语义。ARIA 可以补充角色和状态，但错误配置可能覆盖正确语义；角色声明本身也不会自动实现完整的键盘和交互行为。W3C 将这些信息归入 accessibility semantics。[^1]

## 3. 浏览器中的其他工具路径

浏览器提供的接口比普通桌面应用更多，因此很适合比较不同工具的边界。

| 路径 | 观察或控制依据 | 适用场景 |
|---|---|---|
| Screenshot | 最终像素和视觉布局 | 视觉验收、Canvas、复杂渲染 |
| Accessibility Tree | 角色、名称、状态和层级 | 语义定位和标准控件操作 |
| Playwright | DOM、ARIA、locator、等待和断言 | 稳定网页流程和自动化测试 |
| CDP | Chromium DevTools Protocol | DOM、脚本、网络、控制台和输入控制 |
| 浏览器扩展 | 浏览器会话、标签页和授权 | 连接用户现有浏览器环境 |
| Computer Use Tool | 观察与输入组成的闭环 | 跨应用流程和缺少结构化接口的操作 |

Playwright 的 getByRole 根据网页角色和可访问名称定位元素：

~~~ts
await page.getByRole('button', { name: '提交' }).click();
~~~

这依赖 DOM 和 ARIA 语义，不等于读取操作系统 Accessibility Tree。Playwright 还会在动作前检查目标是否唯一、可见、稳定、能接收事件且已启用，并等待条件满足。它降低的是部分页面时序问题，仍需要业务断言确认“提交成功”。[^2]

CDP 是 Chromium 生态的调试和控制协议，接口更底层，覆盖 DOM、Accessibility、Runtime、Network 和 Input 等领域。它可以做局部查询，也可以一次返回大量 DOM、日志或网络数据；上下文成本取决于调用方式。浏览器扩展可以在权限范围内使用部分调试能力，但扩展、CDP 和 Computer Use 不是同一个概念。[^3]

## 4. 这些工具如何进入 Agent

API、CLI、Computer Use 和浏览器自动化都可以成为 Agent Tool。它们处理的对象不同：

- API 面向结构化业务请求；
- CLI 面向命令、输出和退出状态；
- Computer Use 面向图形界面和输入事件；
- Playwright 等工具面向浏览器页面结构和自动化断言。

Skill 更接近工具使用说明：它可以告诉 Agent 在什么条件下调用工具、动作顺序是什么、哪些操作需要确认，但它不是截图、Accessibility 或点击接口。

MCP（Model Context Protocol）解决的是工具连接、发现和调用。它可以包装业务 API、CLI、Playwright 或 Computer Use，但不规定这些工具内部如何获取状态和执行动作。

~~~mermaid
flowchart LR
    A[Agent] --> C[MCP Client]
    C -->|"MCP 协议"| S[MCP Server]
    S --> B[API / CLI]
    S --> P[Playwright / 浏览器]
    S --> U[Computer Use]
~~~

Agent 也可以直接集成这些工具，不必全部经过 MCP。MCP 的作用是统一连接和工具描述，不会自动带来更高准确率或更大的权限。[^4]

## 5. 为什么同样是点击，结果可能不同

操作效果不只取决于模型。排查一个失败动作时，可以依次检查：

1. 状态是否足够新。页面重排、动画或弹窗可能使截图和语义树过期。
2. 目标是否明确。同名按钮、只有图标的控件和嵌套弹窗都会增加定位难度。
3. 输入是否送到正确位置。窗口焦点、缩放、遮挡和坐标系都会影响点击。
4. 成功信号是否清楚。按钮变灰可能表示加载，也可能表示操作完成。
5. 重试是否会产生副作用。订单、支付、发送和删除不能在未知结果时盲目重复。
6. 是否有恢复路径。工具需要支持重新观察、等待、换路径、暂停或人工接管。

语义完整的表单通常更适合语义定位；缺少语义的画布可能需要截图。视觉操作也不是低级方案，因为颜色、布局和渲染质量只能通过视觉结果判断。准确率应在具体任务、工具路径和权限条件下验证，不能简单排一个固定的技术顺序。

## 6. 移动端：换执行环境，不换闭环

移动端需要替换桌面环境中的状态采集和输入接口，但决策闭环仍然相同：

~~~text
获取状态 → 理解 → 执行 → 验证 → 恢复
~~~

Android 中常见的职责划分如下：

| 能力 | 主要职责 |
|---|---|
| ADB | 设备通信、安装调试、Shell 命令，以及部分截图和输入 |
| UI Automator | 查找控件并与用户应用、系统应用交互 |
| Accessibility Service | 在获得授权后读取可访问性节点、接收事件并执行支持的动作 |
| Screenshot | 提供视觉状态和最终画面 |

ADB 是设备通信和命令通道，不等于 Accessibility Tree。Android 官方将 UI Automator 定义为用于构建 UI 测试、与用户应用和系统应用交互的 API。[^5]

iOS 的 XCTest/XCUIAutomation 可以在测试环境中查询 UI 元素、执行交互和检查结果。将其用于 Agent 控制，还要满足设备连接、签名、开发者配置和系统权限等条件；测试框架的存在不等于普通应用拥有跨应用控制权限。[^6]

## 7. 选择工具的原则

选择入口时，先看任务的真实接口、可用权限和结果能否验证：

- 结构化业务操作优先 API；
- 构建、测试、Git 和文件处理优先 CLI；
- 结构稳定的网页流程优先 Playwright 等浏览器工具；
- 必须经过真实界面、跨应用操作或需要视觉验收时使用 Computer Use；
- 需要统一连接多种工具时，可以使用 MCP。

Computer Use 的价值不在于“让模型会点击”，而在于为 Agent 补上图形界面的观察和操作路径。它能否稳定工作，取决于状态是否可获取、动作是否可执行、结果是否可验证，以及失败后是否可以恢复。一次完整的 Computer Use 系统，最终要把这些环节连成闭环。

## 参考资料

[^1]: [OpenAI API：Computer use](https://developers.openai.com/api/docs/guides/tools-computer-use/)；[W3C WAI-ARIA Authoring Practices：Read Me First](https://www.w3.org/WAI/ARIA/apg/practices/read-me-first/)；[Chrome for Developers：Full accessibility tree in Chrome DevTools](https://developer.chrome.com/blog/full-accessibility-tree)
[^2]: [Playwright：Locators](https://playwright.dev/docs/locators)；[Playwright：Auto-waiting](https://playwright.dev/docs/actionability)
[^3]: [Chrome DevTools Protocol](https://chromedevtools.github.io/devtools-protocol/)；[Chrome Extensions：chrome.debugger](https://developer.chrome.com/docs/extensions/reference/api/debugger)
[^4]: [Model Context Protocol：Architecture](https://modelcontextprotocol.io/docs/learn/architecture)
[^5]: [Android Developers：ADB](https://developer.android.com/tools/adb)；[UI Automator](https://developer.android.com/training/testing/other-components/ui-automator)；[Accessibility Service](https://developer.android.com/guide/topics/ui/accessibility/service)
[^6]: [Apple Developer：XCTest](https://developer.apple.com/documentation/xctest/)；[XCUIAutomation](https://developer.apple.com/documentation/xcuiautomation)；[在模拟器或真机上运行应用](https://developer.apple.com/documentation/xcode/running-your-app-on-simulated-or-physical-devices)

<hr />

**当学习成为了习惯，知识也就变成了常识。** 感谢各位的 **关注**、**点赞**、**收藏**和**评论**。

新视频和文章会第一时间在微信公众号发送，欢迎关注：[李永宁lyn](https://gitee.com/liyongning/typora-image-bed/raw/master/202202171742614.jpg)

文章已收录到 [github 仓库 liyongning/blog](https://github.com/liyongning/blog)，欢迎 Watch 和 Star。

**[更多精彩内容](https://github.com/liyongning/blog/blob/main/README.md)**
