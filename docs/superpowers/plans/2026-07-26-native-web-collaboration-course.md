# 《原生与前端共舞》2026 版 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 编写一套面向前端专家转型端工程师的五讲原创课程，并提供五个由 XcodeGen 组织的最小 iOS 实验。

**Architecture:** 课程按“运行时桥接 → 技术选型 → 声明式 UI → 渲染性能 → 动态化治理”递进。每讲是一篇可独立发布的 Markdown，配套一个隔离的 iOS App Target；五个 Target 由同一个 XcodeGen 工程组织，但不共享业务源码，以保证实验可以独立理解。

**Tech Stack:** Markdown、Swift 6、SwiftUI、UIKit、JavaScriptCore、WebKit、XcodeGen、React Native/Hermes/Fabric、Flutter/Impeller、Apple App Review Guidelines。

## Global Constraints

- 目标读者熟悉 TypeScript、JavaScript、现代前端框架和浏览器原理，但不预设 Objective-C 或 iOS 工程经验。
- 主示例语言为 Swift；JavaScript/TypeScript 只用于桥接协议和前端对照；Objective-C/Objective-C++ 只解释 Runtime 或 C++ 引擎边界。
- 当前实现以 2026 年 Apple、WebKit、React Native、Flutter、Swift 和 LLVM 官方资料为准。
- 每讲固定包含：问题、前端直觉、完整原理、数据结构与调用链、Swift 示例、必要的语言边界、实验、误区、工程建议、小结与检查题。
- 所有性能判断写明场景和指标，不使用无条件的“更快”“接近原生”或“最终赢家”。
- 动态化章节必须分别讨论技术可行性、App Store 合规性和安全性。
- 当前环境为 Apple Swift 6.3.1 命令行工具；`xcodebuild` 因未选择完整 Xcode 而不可用。计划使用 XcodeGen 校验工程结构，并明确记录未完成 iOS SDK 构建验证。

---

## File Map

### Course

- Create: `端工程师/原生与前端共舞/README.md` — 系列入口、阅读路线、术语、实验和版本说明。
- Create: `端工程师/原生与前端共舞/01-Swift与JavaScript双向通信.md` — JavaScriptCore 与 WKWebView Bridge。
- Create: `端工程师/原生与前端共舞/02-跨端技术选型.md` — 现代跨端架构和决策矩阵。
- Create: `端工程师/原生与前端共舞/03-从Flexbox到SwiftUI.md` — 声明式 UI、状态和布局。
- Create: `端工程师/原生与前端共舞/04-端侧渲染全景.md` — 四类渲染管线和性能诊断。
- Create: `端工程师/原生与前端共舞/05-动态化与合规边界.md` — SDUI、热更新、合规与安全。

### Examples

- Create: `端工程师/原生与前端共舞/Examples/project.yml` — 五个独立 iOS App Target。
- Create: `端工程师/原生与前端共舞/Examples/README.md` — 生成、打开、运行和验证说明。
- Create: `端工程师/原生与前端共舞/Examples/JavaScriptBridge/*.swift` — JavaScriptCore Bridge。
- Create: `端工程师/原生与前端共舞/Examples/NativeAndWeb/*` — SwiftUI 与本地 Web 页。
- Create: `端工程师/原生与前端共舞/Examples/SwiftUILayout/*.swift` — 状态、身份和布局实验。
- Create: `端工程师/原生与前端共舞/Examples/RenderingDiagnostics/*.swift` — 卡顿和掉帧实验。
- Create: `端工程师/原生与前端共舞/Examples/ServerDrivenUI/*.swift` — 白名单 SDUI。

### Verification

- Create: `端工程师/原生与前端共舞/scripts/verify-course.sh` — 目录、占位词、危险绝对化表达、Swift 纯模型和 XcodeGen 校验。

---

### Task 1: 系列骨架与可验证工程

**Files:**

- Create: `端工程师/原生与前端共舞/README.md`
- Create: `端工程师/原生与前端共舞/Examples/README.md`
- Create: `端工程师/原生与前端共舞/Examples/project.yml`
- Create: `端工程师/原生与前端共舞/scripts/verify-course.sh`

**Interfaces:**

- Produces: 五篇文章使用的相对链接、五个 Target 名称、统一验证命令 `bash scripts/verify-course.sh`。
- Target names: `JavaScriptBridge`, `NativeAndWeb`, `SwiftUILayout`, `RenderingDiagnostics`, `ServerDrivenUI`。
- Deployment target: iOS 18.0。

- [ ] **Step 1: 写系列 README**

README 必须包含：

- 读者画像和所需前端知识。
- 五讲顺序和每讲解决的问题。
- “稳定原理 / 当前实现 / 工程判断”标记约定。
- Swift、Objective-C、Objective-C++ 的使用边界。
- `Examples/NativeWebLabs.xcodeproj` 的生成方式。
- 资料校验日期 `2026-07-26`。

- [ ] **Step 2: 写 XcodeGen 工程定义**

`project.yml` 定义五个 iOS application target，每个 target：

```yaml
name: NativeWebLabs

targets:
  <target>:
    type: application
    platform: iOS
    deploymentTarget: "18.0"
    settings:
      base:
        PRODUCT_BUNDLE_IDENTIFIER: dev.liyongning.nativeweb.<target>
        SWIFT_VERSION: "6.0"
```

每个 target 只引用自己的同名目录。JavaScriptBridge 链接 `JavaScriptCore.framework`，NativeAndWeb 链接 `WebKit.framework`。

- [ ] **Step 3: 写实验 README**

给出以下准确命令：

```bash
cd 端工程师/原生与前端共舞/Examples
xcodegen generate
open NativeWebLabs.xcodeproj
```

说明当前自动验证只覆盖 XcodeGen 解析和纯 Foundation 模型；实际 iOS 构建需要完整 Xcode 与 iOS 18+ SDK。

- [ ] **Step 4: 写验证脚本**

脚本以 `set -euo pipefail` 开始并执行：

1. 检查 README、五篇文章和五个实验目录存在。
2. 搜索 `TBD|TODO|待补充|占位文本`，命中则失败。
3. 搜索 `性能一定|一定比原生|最终赢家|可以绕过审核`，命中则失败。
4. 运行 `swiftc -typecheck Examples/JavaScriptBridge/BridgeProtocol.swift`。
5. 运行 `swiftc -typecheck Examples/ServerDrivenUI/SDUIModel.swift`。
6. 在临时目录执行 `xcodegen generate --spec Examples/project.yml --project <temp> --project-root Examples`。

- [ ] **Step 5: 验证初始失败**

Run:

```bash
cd 端工程师/原生与前端共舞
bash scripts/verify-course.sh
```

Expected: 因五篇文章和模型文件尚未创建而失败；失败路径应指向第一个缺失文件，而不是脚本语法错误。

- [ ] **Step 6: Commit**

```bash
git add '端工程师/原生与前端共舞/README.md' \
  '端工程师/原生与前端共舞/Examples/README.md' \
  '端工程师/原生与前端共舞/Examples/project.yml' \
  '端工程师/原生与前端共舞/scripts/verify-course.sh'
git commit -m 'docs: scaffold native and web collaboration course'
```

---

### Task 2: 第一讲与 JavaScriptCore Bridge 实验

**Files:**

- Create: `端工程师/原生与前端共舞/01-Swift与JavaScript双向通信.md`
- Create: `端工程师/原生与前端共舞/Examples/JavaScriptBridge/JavaScriptBridgeApp.swift`
- Create: `端工程师/原生与前端共舞/Examples/JavaScriptBridge/ContentView.swift`
- Create: `端工程师/原生与前端共舞/Examples/JavaScriptBridge/BridgeProtocol.swift`
- Create: `端工程师/原生与前端共舞/Examples/JavaScriptBridge/JavaScriptRuntime.swift`

**Interfaces:**

- Produces:

```swift
struct BridgeRequest: Codable, Equatable {
    let id: UUID
    let method: String
    let payload: [String: String]
}

struct BridgeError: Codable, Equatable, Error {
    let code: String
    let message: String
}

struct BridgeResponse: Codable, Equatable {
    let id: UUID
    let result: [String: String]?
    let error: BridgeError?
}
```

- `@MainActor final class JavaScriptRuntime: ObservableObject` owns one `JSVirtualMachine` and one `JSContext`.
- Supported native method: `deviceInfo`, returning platform and runtime strings.

- [ ] **Step 1: 写可由命令行校验的 Bridge 协议**

`BridgeProtocol.swift` 只导入 Foundation，定义上述三个 Codable 类型，并提供：

```swift
enum BridgeCodec {
    static func decodeRequest(_ json: String) throws -> BridgeRequest
    static func encodeResponse(_ response: BridgeResponse) throws -> String
}
```

- [ ] **Step 2: 验证纯模型可编译**

Run:

```bash
swiftc -typecheck '端工程师/原生与前端共舞/Examples/JavaScriptBridge/BridgeProtocol.swift'
```

Expected: PASS，无输出。

- [ ] **Step 3: 实现 JavaScriptRuntime**

实现以下行为：

- 同一 `JSContext` 在 `@MainActor` 使用。
- 设置 `exceptionHandler`，把异常展示到 `lastError`。
- 将 `nativeCall` 作为 `@convention(block) (String) -> String` 注入 Context。
- 对未知 method 返回 `METHOD_NOT_FOUND`。
- 对非法 JSON 返回 `INVALID_REQUEST`。
- `runDemo()` 执行 JavaScript，调用 `nativeCall` 并返回解析后的文本。

- [ ] **Step 4: 实现 SwiftUI 实验界面**

界面包含：

- “运行 JavaScript → Swift → JavaScript”按钮。
- 成功结果区域。
- JavaScript 异常按钮和错误区域。
- 一段说明同一 VM 串行、不同 VM 才能并发的文本。

- [ ] **Step 5: 写第一讲正文**

正文必须覆盖：

- 浏览器 Realm 与 `JSContext` 的类比边界。
- JSVirtualMachine、JSContext、JSValue、JSManagedValue。
- 同一 VM 串行与 API 线程安全。
- Swift Closure、`JSExport` 和 Objective-C Runtime 的关系。
- WKWebView 为什么不是直接获取页面 `JSContext`。
- `WKScriptMessageHandler` 与 JavaScriptCore 的适用场景。
- Hermes、V8、JavaScriptCore 的区别。
- 请求 ID、协议版本、错误码、超时、取消、重复响应和白名单。
- 纠正“所有跨端框架都使用 JSC”“每个 VM 固定对应一个线程”“JSValue 调用 evaluateScript”。

- [ ] **Step 6: 生成工程验证**

Run:

```bash
tmp_dir="$(mktemp -d)"
xcodegen generate \
  --spec '端工程师/原生与前端共舞/Examples/project.yml' \
  --project "$tmp_dir" \
  --project-root '端工程师/原生与前端共舞/Examples'
test -d "$tmp_dir/NativeWebLabs.xcodeproj"
```

Expected: `JavaScriptBridge` target 生成成功。

- [ ] **Step 7: Commit**

```bash
git add '端工程师/原生与前端共舞/01-Swift与JavaScript双向通信.md' \
  '端工程师/原生与前端共舞/Examples/JavaScriptBridge'
git commit -m 'docs: add Swift and JavaScript bridge lesson'
```

---

### Task 3: 第二讲与 Native/Web 边界实验

**Files:**

- Create: `端工程师/原生与前端共舞/02-跨端技术选型.md`
- Create: `端工程师/原生与前端共舞/Examples/NativeAndWeb/NativeAndWebApp.swift`
- Create: `端工程师/原生与前端共舞/Examples/NativeAndWeb/ContentView.swift`
- Create: `端工程师/原生与前端共舞/Examples/NativeAndWeb/Product.swift`
- Create: `端工程师/原生与前端共舞/Examples/NativeAndWeb/WebProductView.swift`
- Create: `端工程师/原生与前端共舞/Examples/NativeAndWeb/product.html`

**Interfaces:**

- Produces:

```swift
struct Product: Codable, Identifiable, Equatable {
    let id: UUID
    let name: String
    let priceInCents: Int
}
```

- Native tab renders `[Product]` with SwiftUI.
- Web tab encodes the same values to JSON and calls `window.renderProducts(products)`.

- [ ] **Step 1: 实现统一业务模型和两个渲染出口**

创建固定的三条 Product 测试数据。SwiftUI 使用 `List`；WKWebView 加载 bundle 中的 `product.html`，页面加载完成后通过 `callAsyncJavaScript` 传入 JSON-compatible 对象。

- [ ] **Step 2: 实现对比界面**

使用 `TabView` 提供 Native 和 Web 两页，显示：

- 数据模型是否复用。
- UI 代码是否复用。
- 平台能力调用点。
- WebView 初始化和首屏的额外边界。

- [ ] **Step 3: 写第二讲正文**

正文必须覆盖：

- “跨端”至少拆成逻辑复用、UI 复用、渲染复用和平台能力复用。
- React Native 当前 Hermes、JSI、Fabric、TurboModules。
- Flutter 当前 Dart JIT/AOT、Widget、Engine 和 Impeller。
- Kotlin Multiplatform 与 Compose Multiplatform的不同复用层次。
- WebView Hybrid、Server-Driven UI 和纯原生。
- 组织、体验、性能、包体、生态、调试、升级和合规决策矩阵。
- 纠正“跨端只有 JSC 与非 JSC 两类”“RN 就是 JSON Bridge”“Flutter 成败取决于 Fuchsia”。
- 给出三个带约束的选型案例：内容型业务、复杂交互消费 App、已有双端大型团队。

- [ ] **Step 4: 检查资料和绝对化表达**

Run:

```bash
rg -n 'Hermes|JSI|Fabric|TurboModules|Impeller|Kotlin Multiplatform|Server-Driven UI' \
  '端工程师/原生与前端共舞/02-跨端技术选型.md'
! rg -n '最终赢家|一定比原生|Fuchsia.*决定' \
  '端工程师/原生与前端共舞/02-跨端技术选型.md'
```

Expected: 第一条命令覆盖所有关键词；第二条命令返回成功。

- [ ] **Step 5: Commit**

```bash
git add '端工程师/原生与前端共舞/02-跨端技术选型.md' \
  '端工程师/原生与前端共舞/Examples/NativeAndWeb'
git commit -m 'docs: add modern cross-platform selection lesson'
```

---

### Task 4: 第三讲与 SwiftUI 状态布局实验

**Files:**

- Create: `端工程师/原生与前端共舞/03-从Flexbox到SwiftUI.md`
- Create: `端工程师/原生与前端共舞/Examples/SwiftUILayout/SwiftUILayoutApp.swift`
- Create: `端工程师/原生与前端共舞/Examples/SwiftUILayout/ContentView.swift`
- Create: `端工程师/原生与前端共舞/Examples/SwiftUILayout/FlowLayout.swift`
- Create: `端工程师/原生与前端共舞/Examples/SwiftUILayout/IdentityDemo.swift`

**Interfaces:**

- `struct FlowLayout: Layout` implements `sizeThatFits` and `placeSubviews`.
- `IdentityDemo` shows stable `Identifiable` IDs versus index-based identity.
- `@Observable final class TagStore` owns mutable tags.

- [ ] **Step 1: 实现 FlowLayout**

实现横向放置、宽度不足换行、可配置 horizontal/vertical spacing。缓存不是本实验目标，不增加自定义 Cache。

- [ ] **Step 2: 实现状态和身份实验**

界面允许：

- 添加、删除和打乱 Tag。
- 在稳定 ID 与数组索引两种模式间切换。
- 给单个 Tag 保留本地状态，观察错误身份导致的状态错位。

- [ ] **Step 3: 写第三讲正文**

正文必须覆盖：

- Frame、Auto Layout、UIStackView、SwiftUI 的演进。
- CSS Flexbox 的单向布局与 SwiftUI proposal/response 布局协商。
- Result Builder，不再使用 Function Builder 旧名。
- SwiftUI View 是短生命周期值描述，不等于 UIView 或 DOM Node。
- View Identity、State lifetime、Observation 和更新失效范围。
- React reconciliation、Vue 依赖追踪与 SwiftUI 更新模型的可类比和不可类比之处。
- `UIViewRepresentable`、`UIHostingController` 的混合边界。
- 纠正“SwiftUI 就是前端 DSL”“Preview 等于 Flutter Hot Reload”“SwiftUI 会立即替代 UIKit”。

- [ ] **Step 4: 检查术语**

Run:

```bash
rg -n 'Result Builder|View Identity|Layout|Observation|UIViewRepresentable|UIHostingController' \
  '端工程师/原生与前端共舞/03-从Flexbox到SwiftUI.md'
! rg -n 'Function Builders|必然替代 UIKit|等同于虚拟 DOM' \
  '端工程师/原生与前端共舞/03-从Flexbox到SwiftUI.md'
```

Expected: 当前术语全部出现，过期或绝对化术语不出现。

- [ ] **Step 5: Commit**

```bash
git add '端工程师/原生与前端共舞/03-从Flexbox到SwiftUI.md' \
  '端工程师/原生与前端共舞/Examples/SwiftUILayout'
git commit -m 'docs: add declarative SwiftUI layout lesson'
```

---

### Task 5: 第四讲与渲染诊断实验

**Files:**

- Create: `端工程师/原生与前端共舞/04-端侧渲染全景.md`
- Create: `端工程师/原生与前端共舞/Examples/RenderingDiagnostics/RenderingDiagnosticsApp.swift`
- Create: `端工程师/原生与前端共舞/Examples/RenderingDiagnostics/ContentView.swift`
- Create: `端工程师/原生与前端共舞/Examples/RenderingDiagnostics/JankScenarios.swift`

**Interfaces:**

- `enum JankScenario`: `mainThreadWork`, `imageDecode`, `excessiveLayout`.
- `@MainActor final class JankController` records scenario, start time and measured duration.
- “修复版”把 CPU 工作移到 detached task，仅将结果提交回 MainActor。

- [ ] **Step 1: 实现三个可控问题**

- Main thread: 在按钮事件中执行固定规模的素数计算。
- Image decode: 生成大尺寸压缩图并在主线程首次解码。
- Excessive layout: 构造多层嵌套、随状态反复变化的布局。

每个场景同时提供原始版和修复版，不承诺固定耗时。

- [ ] **Step 2: 写诊断步骤到实验界面**

界面显示：

- 如何在 Instruments 选择 Time Profiler、Core Animation 和 Hangs。
- 应观察 Main Thread、frame time、CPU 和提交阶段。
- Simulator 与真机结果不能直接等价。

- [ ] **Step 3: 写第四讲正文**

正文必须覆盖：

- 浏览器 DOM/CSSOM/Layout/Paint/Raster/Composite。
- SwiftUI/UIKit、Core Animation transaction、Render Server、Metal。
- React Native Fabric Render/Commit/Mount。
- Flutter Widget/Element/RenderObject/Layer 与 Impeller。
- CPU、GPU、主线程、渲染线程和进程边界。
- 帧预算按刷新率计算，60 Hz 为约 16.67 ms、120 Hz 为约 8.33 ms，但应用拿不到全部预算。
- OpenGL ES 在 iOS 12 被弃用。
- WebKit GPU Process 与“WebView 必然白屏”的纠正。
- 性能比较必须指定首屏、交互延迟、动画、内存或包体指标。

- [ ] **Step 4: 检查现代图形栈**

Run:

```bash
rg -n 'Metal|Render Server|Fabric|Impeller|GPU Process|16\\.67|8\\.33' \
  '端工程师/原生与前端共舞/04-端侧渲染全景.md'
! rg -n 'Flutter.*Skia.*OpenGL.*当前|WebView.*白屏.*无法避免|几何着色器.*固定六阶段' \
  '端工程师/原生与前端共舞/04-端侧渲染全景.md'
```

Expected: 现代术语出现，过期结论不出现。

- [ ] **Step 5: Commit**

```bash
git add '端工程师/原生与前端共舞/04-端侧渲染全景.md' \
  '端工程师/原生与前端共舞/Examples/RenderingDiagnostics'
git commit -m 'docs: add modern rendering pipeline lesson'
```

---

### Task 6: 第五讲与白名单 SDUI 实验

**Files:**

- Create: `端工程师/原生与前端共舞/05-动态化与合规边界.md`
- Create: `端工程师/原生与前端共舞/Examples/ServerDrivenUI/ServerDrivenUIApp.swift`
- Create: `端工程师/原生与前端共舞/Examples/ServerDrivenUI/ContentView.swift`
- Create: `端工程师/原生与前端共舞/Examples/ServerDrivenUI/SDUIModel.swift`
- Create: `端工程师/原生与前端共舞/Examples/ServerDrivenUI/SDUIRenderer.swift`
- Create: `端工程师/原生与前端共舞/Examples/ServerDrivenUI/sample-page.json`

**Interfaces:**

- Produces:

```swift
struct PageSchema: Codable, Equatable {
    let protocolVersion: Int
    let minimumAppVersion: Int
    let root: ComponentNode
}

indirect enum ComponentNode: Codable, Equatable {
    case text(value: String, style: TextStyle)
    case image(assetName: String)
    case button(title: String, action: Action)
    case stack(axis: LayoutAxis, spacing: Double, children: [ComponentNode])
}

enum TextStyle: String, Codable, Equatable {
    case body
    case headline
}

enum LayoutAxis: String, Codable, Equatable {
    case horizontal
    case vertical
}

enum Action: Codable, Equatable {
    case openAllowlistedRoute(String)
    case track(String)
}
```

- Unknown component decoding fails closed.
- Only local asset names and allowlisted routes are accepted.
- Supported protocol version is exactly `1`.

- [ ] **Step 1: 写纯 Foundation Schema**

`SDUIModel.swift` 不导入 SwiftUI；为关联值枚举实现明确的 `type` discriminator 编解码。定义：

- `SchemaError.unsupportedProtocolVersion`
- `SchemaError.unsupportedComponent`
- `SchemaError.disallowedRoute`
- `SchemaValidator.validate(_:)`

- [ ] **Step 2: 验证 Schema 可编译**

Run:

```bash
swiftc -typecheck '端工程师/原生与前端共舞/Examples/ServerDrivenUI/SDUIModel.swift'
```

Expected: PASS，无输出。

- [ ] **Step 3: 实现渲染器和失败状态**

`SDUIRenderer` 只把已解码的四种组件映射为 SwiftUI。非法 Schema 显示本地错误页，不尝试猜测或跳过未知组件。界面提供：

- 加载合法 Schema。
- 加载未知组件。
- 加载不兼容协议版本。
- 回退到本地静态页面。

- [ ] **Step 4: 写第五讲正文**

正文必须覆盖：

- Remote Config、A/B Test、Web 内容、SDUI、JS Bundle、方法热修复和下载可执行代码的能力阶梯。
- Apple App Review Guidelines 2.5.2 的边界。
- “数据驱动 UI”与“下载新功能代码”的区别。
- Schema 版本、最低客户端版本、能力白名单、签名、灰度、回滚、Kill Switch 和可观测性。
- JSPatch、OCS、LLVM Interpreter/JIT 作为历史案例。
- JavaScriptCore 是 LLInt/Baseline/DFG/FTL 分层优化，不是笼统的 tracing JIT。
- LLVM 新 JIT 项目优先 ORCv2/LLJIT，但这不消除 iOS 平台执行权限与审核约束。
- 给出上线前威胁模型和失败策略。

- [ ] **Step 5: 检查合规表达**

Run:

```bash
rg -n '2\\.5\\.2|能力白名单|协议版本|签名|灰度|回滚|Kill Switch|fail.closed|ORCv2' \
  '端工程师/原生与前端共舞/05-动态化与合规边界.md'
! rg -n '绕过审核|有了解释器就能热更新|动态化.*随意发版' \
  '端工程师/原生与前端共舞/05-动态化与合规边界.md'
```

Expected: 合规与治理关键词出现，危险结论不出现。

- [ ] **Step 6: Commit**

```bash
git add '端工程师/原生与前端共舞/05-动态化与合规边界.md' \
  '端工程师/原生与前端共舞/Examples/ServerDrivenUI'
git commit -m 'docs: add compliant dynamic delivery lesson'
```

---

### Task 7: 全系列一致性、资料和工程验收

**Files:**

- Modify: `端工程师/原生与前端共舞/README.md`
- Modify: `端工程师/原生与前端共舞/Examples/README.md`
- Modify as needed: `端工程师/原生与前端共舞/*.md`
- Modify as needed: `端工程师/原生与前端共舞/Examples/**/*`

**Interfaces:**

- Consumes: 五篇正文、五个实验和统一验证脚本。
- Produces: 可独立阅读、可生成 Xcode 工程、验证范围明确的完整课程。

- [ ] **Step 1: 执行完整验证**

Run:

```bash
cd '端工程师/原生与前端共舞'
bash scripts/verify-course.sh
```

Expected: PASS，并输出课程文件检查、两个纯模型 typecheck 和 XcodeGen generation 均成功。

- [ ] **Step 2: 检查五讲固定结构**

Run:

```bash
for file in 0*.md; do
  rg -q '本讲要解决的问题' "$file"
  rg -q '前端' "$file"
  rg -q '实验' "$file"
  rg -q '常见误区' "$file"
  rg -q '工程' "$file"
  rg -q '检查题' "$file"
  rg -q '延伸资料' "$file"
done
```

Expected: 所有命令成功。

- [ ] **Step 3: 检查内部链接**

提取所有相对 Markdown 链接，逐一验证目标存在；README 中的五篇正文链接和五个实验目录链接必须全部有效。

- [ ] **Step 4: 检查版本敏感事实**

逐项核对并在相邻段落附官方链接：

- React Native 0.70 Hermes 默认、0.76 新架构默认、0.84 Hermes V1。
- Flutter 3.27 Impeller 在 iOS 和 Android API 29+ 默认。
- OpenGL ES 从 iOS 12 起弃用。
- Swift Function Builders 更名为 Result Builders。
- Apple App Review Guidelines 2.5.2。
- LLVM ORCv2/LLJIT 相对 MCJIT 的当前定位。

- [ ] **Step 5: 记录构建验证边界**

在 `Examples/README.md` 明确写出：

- 已通过 Swift 6.3.1 对两个纯 Foundation 文件进行 typecheck。
- 已通过 XcodeGen 解析并生成五 Target 工程。
- 未运行 `xcodebuild`，因为当前 active developer directory 是 CommandLineTools，而非完整 Xcode。
- 用户安装并选择完整 Xcode 后的验证命令：

```bash
sudo xcode-select --switch /Applications/Xcode.app/Contents/Developer
xcodebuild -project NativeWebLabs.xcodeproj \
  -scheme JavaScriptBridge \
  -destination 'platform=iOS Simulator,name=iPhone 17 Pro' \
  build
```

- [ ] **Step 6: 检查 Git 差异**

Run:

```bash
git diff --check
git status --short
```

Expected: 无空白错误；只包含本课程相关改动和用户原有未跟踪文件，不修改 `端工程师/swift 入门.md` 或 `markText todo.md`。

- [ ] **Step 7: Final commit**

```bash
git add '端工程师/原生与前端共舞'
git commit -m 'docs: complete native and web collaboration course'
```
