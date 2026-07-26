# 《原生与前端共舞》2026 版课程设计

## 1. 目标

为具有扎实前端基础、正在转型为端工程师的读者，重新编写一套关于 JavaScript 与 iOS 原生协作、跨端选型、声明式 UI、渲染机制和动态化治理的五讲课程。

课程应当：

- 使用读者熟悉的 TypeScript、React/Vue、浏览器运行时和渲染管线建立直觉。
- 系统补齐 Swift、iOS、跨端框架和端侧工程知识。
- 区分稳定原理、当前实现和带有前提的工程判断。
- 以 Swift 为主要示例语言，只在运行时或引擎边界保留 Objective-C/Objective-C++。
- 每讲包含一个可运行的最小 Xcode 实验。
- 取代 2019 年课程中已经过期或不严谨的知识，不复述原课程正文。

## 2. 读者与前置知识

目标读者是一名：

- 熟悉 TypeScript、JavaScript、浏览器原理和现代前端框架的前端专家。
- 正在向 iOS、跨端和完整端工程方向转型。
- 已经开始学习 Swift，但不预设具备 Objective-C 或 iOS 工程经验。

课程不重复讲解基础前端概念。首次出现的端侧概念需要给出准确的前端类比，同时明确类比的边界。

## 3. 课程结构

### 第一讲：Swift 与 JavaScript 如何双向通信

主线：

- JavaScriptCore 的 VM、Context、Value、GC、线程与并发。
- Swift 调用 JavaScript，以及 JavaScript 调用 Swift。
- `JSExport`、Closure、`JSManagedValue`、异常和类型转换。
- JavaScriptCore 与 WKWebView 消息桥的本质区别。
- Bridge 的协议、类型安全、超时、错误、生命周期和安全边界。
- Hermes、V8、JavaScriptCore 的角色差异。

实验：用 Swift 实现一个支持请求 ID、结构化错误和异步返回的双向 Bridge。

### 第二讲：跨端不是二选一

主线：

- React Native 的 Hermes、JSI、Fabric 和 TurboModules。
- Flutter 的 Dart JIT/AOT、Widget 体系和 Impeller。
- Kotlin Multiplatform、Compose Multiplatform、WebView Hybrid、Server-Driven UI 和纯原生。
- 团队能力、体验目标、原生能力占比、性能、包体积、生态和升级成本。
- 使用决策矩阵替代“寻找最终赢家”。

实验：用统一业务模型实现 SwiftUI 原生页和 Web 混合页，识别真正能够复用的边界。

### 第三讲：从 DOM、Flexbox 到 SwiftUI

主线：

- Frame、Auto Layout、UIStackView 和声明式布局的演进。
- `UI = f(state)` 在 React/Vue、SwiftUI、React Native 和 Flutter 中的不同实现。
- Result Builder、View Identity、状态、依赖追踪和布局协商。
- SwiftUI View 与虚拟 DOM 的相似点和根本差异。
- SwiftUI 与 UIKit 的双向集成。

实验：实现一组可复用 SwiftUI 组件，并观察状态、身份和布局变化如何驱动更新。

### 第四讲：一帧是怎样产生的

主线：

- 浏览器的 DOM/CSSOM/Layout/Paint/Composite。
- SwiftUI/UIKit、Core Animation、Render Server、Metal 和 GPU。
- React Native Fabric 的 Shadow Tree、Render、Commit 和 Mount。
- Flutter 的 Widget、Element、RenderObject、Layer 和 Impeller。
- 主线程、布局、图片解码、光栅化、合成与帧预算。
- 为什么不能脱离场景讨论“哪种技术一定更快”。

实验：主动制造主线程阻塞、图片解码和布局问题，并使用端侧工具定位掉帧原因。

### 第五讲：动态化的边界

主线：

- Remote Config、A/B Test、Web 内容、Server-Driven UI、JS Bundle 更新、方法热修复和下载代码的差异。
- App Store 审核规则与动态能力边界。
- Schema、能力白名单、协议版本、签名、灰度、回滚和 Kill Switch。
- 脚本或配置执行环境的安全模型。
- JSPatch、OCS、自建解释器和 LLVM 只作为历史与原理案例。

实验：实现一个只接受声明式 JSON、组件能力由本地白名单控制的 Server-Driven UI 页面。

## 4. 单讲固定结构

每一讲使用相同的阅读结构：

1. 本讲要解决的问题。
2. 用前端知识建立第一层直觉。
3. iOS 或跨端侧的完整原理。
4. 关键数据结构与调用链。
5. Swift 主示例。
6. 必要的 Objective-C/Objective-C++ 边界。
7. 最小可运行 Xcode 实验。
8. 常见误区与过期知识纠正。
9. 工程实践与选型建议。
10. 小结、检查题和官方延伸资料。

内容深度以“能读懂架构、能完成业务实现、能够继续阅读源码”为准，不写成 API 枚举手册。

## 5. 类比原则

允许使用以下类比帮助读者迁移知识：

- `JSContext` 与 JavaScript Realm/执行环境。
- SwiftUI View 与虚拟 DOM 描述。
- Core Animation Layer Tree 与浏览器 Layer Tree。
- Server-Driven UI 与 Schema 驱动表单或低代码渲染。

每个类比都必须同时解释“不等价之处”，避免用前端术语覆盖端侧真实机制。

## 6. 示例代码原则

- 主要语言为 Swift。
- JavaScript/TypeScript 用于 Bridge 协议和前端对照。
- Objective-C/Objective-C++ 仅用于 Runtime、C/C++ 引擎胶水层或旧代码迁移说明。
- 示例必须包含必要的错误处理、线程约束、生命周期和内存管理。
- 不提供只有 happy path 的伪工程代码。
- 五个实验能够独立运行，也应使用一致的协议和命名，使其最终可以组合成一个小型 Hybrid App。

## 7. 目录与交付物

```text
blog/端工程师/原生与前端共舞/
├── README.md
├── 01-Swift与JavaScript双向通信.md
├── 02-跨端技术选型.md
├── 03-从Flexbox到SwiftUI.md
├── 04-端侧渲染全景.md
├── 05-动态化与合规边界.md
└── Examples/
    ├── JavaScriptBridge/
    ├── NativeAndWeb/
    ├── SwiftUILayout/
    ├── RenderingDiagnostics/
    └── ServerDrivenUI/
```

`README.md` 提供前置知识、阅读顺序、术语表、各讲关系、实验运行方式和资料更新时间。

## 8. 事实与版本管理

- 当前实现以 2026 年可获得的 Apple、WebKit、React Native、Flutter、Swift 和 LLVM 官方资料为准。
- 对版本敏感的实现明确记录版本或更新时间。
- 稳定原理不绑定无关版本号。
- 主观选型结论必须列出适用条件，不写无条件的性能排名或“最终赢家”。
- 动态化章节必须把技术可行性、审核合规性和安全性分开讨论。
- 引用优先使用官方或项目一手资料。

## 9. 验证

### 文档验证

- 检查五讲术语和协议命名是否一致。
- 检查所有外部链接和版本敏感结论。
- 搜索过期术语、绝对化性能结论和没有前提的推荐。
- 检查代码块语言、文件名和正文引用是否一致。

### 示例验证

- 每个示例至少完成静态检查或实际构建。
- 能使用本机 Xcode 和 iOS SDK 时，执行命令行构建。
- 无法实际构建的部分必须在 README 中标明原因和已完成的验证范围。
- Bridge 示例验证成功响应、JavaScript 异常、未知方法、超时和重复响应。
- SDUI 示例验证未知组件、非法属性、协议版本不兼容和回滚路径。
- 性能示例验证能够稳定制造问题，并说明观测指标而不是承诺固定数值。

## 10. 完成标准

课程完成时应满足：

- 五讲正文和系列 README 均完整，无占位内容。
- 读者不依赖原课程也能建立完整知识链。
- Swift 是主线语言，Objective-C 的出现都有明确必要性。
- 五个实验目录、运行步骤和预期结果完整。
- 当前框架架构、图形栈和审核规则均经过官方资料校验。
- 文档能够作为博客系列独立发布。

