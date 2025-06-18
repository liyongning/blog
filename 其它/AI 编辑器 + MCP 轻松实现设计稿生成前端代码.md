**当学习成为了习惯，知识也就变成了常识。** 感谢各位的 **关注**、**点赞**、**收藏**和**评论**。

新视频和文章会第一时间在微信公众号发送，欢迎关注：[李永宁lyn](https://gitee.com/liyongning/typora-image-bed/raw/master/202202171742614.jpg)

文章已收录到 [github 仓库 liyongning/blog](https://github.com/liyongning/blog)，欢迎 Watch 和 Star。

**[更多精彩内容](https://github.com/liyongning/blog/blob/main/README.md)**

<hr />

# 简介

今天给大家介绍一款前端生产力提效工具：MasterGo Magic MCP，它是一项独立的 MCP 服务，旨在将 MasterGo 设计工具与 AI 模型链接起来。它使 AI 驱动的工具能够直接从 MasterGo 文件中检索、处理和使用设计数据，从而弥合设计资产和代码生成之间的差距

# 使用效果

第一张图是设计稿，第二张是转换后的效果，还原度是非常高的。

![](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202506172157207.png)![image-20250617215550790](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202506172155838.png)

# 基本使用

在正式开始配置之前，需要从 [MasterGo](https://mastergo.com/files/account?tab=security) 获取个人访问令牌（token），因为后面的 MCP Server 配置中需要用到。

![image-20250617083756512](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202506170837544.png)

接下来配置 AI 编辑器，这里以字节的 Trae 为例，其它编辑器（包括插件）可以查看后面的 **其它工具** 章节。

## Trae

以 v1.4.3 版本为例

* 点击右上角的头像 -> AI 功能管理 -> MCP

  ![image-20250617084959549](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202506170849597.png)

* 由于 Trae 的插件市场还没有该插件，需要点击 “添加” -> “手动添加”

* 在弹框中粘贴以下内容：

  ```json
  {
    "mcpServers": {
      "mastergo-magic-mcp": {
        "command": "npx",
        "args": [
          "-y",
          "@mastergo/magic-mcp",
          "--token=mg_31e0294a883044cda42be218a0887477",
          "--url=https://mastergo.com",
          "--rule=Use TypeScript for all components with strict type definitions",
          "--rule=Use the method of SCSS combined with CSS modules for style design",
          "--rule=Follow Ant Design component patterns and design tokens",
          "--rule=Generate components with proper ESLint and Prettier formatting",
          "--rule=Create reusable, single-responsibility components",
          "--rule=Include proper TypeScript interfaces for all props",
          "--rule=Replace icons and images with placeholders. There is no need to implement it with code"
        ],
        "env": {}
      }
    }
  }
  ```

> 这里添加了很多自定义规则（--rule），是为了约束 AI，生成符合项目技术栈的代码。这些规则会和 MCP 内部的规则做合并，一起提供给 AI。大家可以根据自己的情况灵活调配。

这是添加成功后的界面，需要注意这个提示信息，意思是只有在智能体模式下才能使用，所以在使用时需要先在智能体模式中选中 `Builder with MCP` 。

![image-20250617085217326](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202506170852363.png)

![image-20250617085318523](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202506170853579.png)

接下来打开设计稿选中对应的图层，然后复制地址栏中的 url 到聊天框，比如：

![image-20250617204016300](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202506172040410.png)

![image-20250618083106239](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202506180831769.png)

**重点：** 这里选择图层的时候，和日常开发一样，一定要以组件为单位进行选择，这么做有两个目的：

* 组件化的方式，AI 生成的代码还原度会更高
* 生成的前端代码可维护性也更好，二次改动的复杂度也更低

![image-20250617215411830](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202506172154872.png)

## 其它工具

其它工具可参考 [官方配置](https://mastergo.com/file/155675508499265?page_id=12549%3A4448&devMode=true)，像主流的 cursor、vscode + Lingma、vscode + cline 都支持。但使用时需要注意关于 agent 模式的提示，比如 vscode + Lingma 的方案。

# 深入理解

这是 MasterGo Magic MCP 的架构图，分为上中下三部分。最上面的是 AI 模型的宿主环境，比如 Trae、Cursor 等，中间的则是 MCP 本身，负责串联 AI 模型和 MasterGo 平台，最下面的则是 MasterGo 平台服务，负责为 MCP 提供数据。

![image-20250616083846431](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202506160849467.png)

MCP 目前提供了四个核心的工具，分别是：GetDSLTool、GetMetaTool、GetComponentLinkTool、GetComponentWorkflowTool。

## GetDSLTool

DSL 是 MasterGo Magic MCP（模型上下文协议）系统的核心组件，用于从 MasterGo 设计文件中检索设计规范语言（DSL）数据。该工具充当 MasterGo 设计元素和代码生成过程之间的桥梁，使 AI 工具能够理解和解释设计结构。

DSL 工具提供对详细设计信息（包括组件层次结构、属性、样式和关系）的访问。这些数据对于以下方面至关重要：

* 分析 MasterGo 设计的结构
* 了解组件层次结构和关系
* 提取设计属性和约束
* 促进基于设计规范的代码生成

该工具还返回在根据检索到的 DSL 数据生成代码时必须遵循的规则。

## GetMetaTool

元数据工具是 MasterGo Magic MCP 系统中的一个专用组件，用于从 MasterGo 设计文件中检索高级站点配置信息。该工具通过提供有关设计的结构化元数据来充当设计规范和开发之间的桥梁，这些元数据可用于为 AI 辅助代码生成提供信息。

元数据工具旨在从 MasterGo 设计文件中检索高级站点配置信息和元数据。它在以下情况下特别有用：

- 从设计规范构建完整的网站
- 获取高级站点配置信息
- 了解设计元素的结构和属性
- 检索实施过程中应遵循的规则

该工具返回元数据结果和一组规则（如 markdown），这些规则指导应如何解释和使用元数据。

## GetComponentLinkTool

是一个专用的 MCP 工具，用于从额外的 URL 获取组件文档，以帮助 AI 模型生成准确的前端代码实现。

如果当前 UI 在设计时使用了组件库，比如 Ant Design，那这个 UI 的 DSL 数据中就可能会包含非空的 `componentDocumentLinks` 数组，这个数组中存放的是依赖的组件的文档链接，AI 随后会使用 `mcp__getComponentLink` 工具循环获取这些文档内容，基于获取的组件文档，AI 能够理解如何正确使用这些组件并生成符合规范的代码。

这个机制确保了从设计到代码的转换过程中，AI 能够获得准确的组件使用指南，从而生成符合组件库规范的前端代码。

## GetComponentWorkflowTool

该工具是 MasterGo Magic MCP 系统中专门用于组件开发场景的工具，与其他工具（如 `GetDslTool`、`GetComponentLinkTool`）协同工作，形成完整的设计到代码转换流程。

这个工具不直接生成组件代码，而是为组件开发提供"脚手架"和"指导手册"，确保开发者（AI）按照标准化流程开发出符合设计规范的组件。所以，`GetComponentWorkflowTool` 的核心作用就是：**在将设计稿中的指定 UI 的 DSL 数据生成前端组件代码时，提供组件开发的规范、流程以及处理设计资源，即提供完整的组件开发工作流**。

# 总结

本文讲解了 MasterGo Magic MCP 的基础使用和核心工具的介绍，有了该 MCP 前端 UI 研发效率至少提升一倍以上。

Magic MCP 实现设计稿生成前端代码的原理简单总结就是：通过图层的 layerId 和 fileId 去 MasterGo 平台获取到对应的 DSL 数据，并将数据交给模型，然后模型根据数据以及数据中的约束生成符合要求的前端代码。

<hr />

**当学习成为了习惯，知识也就变成了常识。** 感谢各位的 **关注**、**点赞**、**收藏**和**评论**。

新视频和文章会第一时间在微信公众号发送，欢迎关注：[李永宁lyn](https://gitee.com/liyongning/typora-image-bed/raw/master/202202171742614.jpg)

文章已收录到 [github 仓库 liyongning/blog](https://github.com/liyongning/blog)，欢迎 Watch 和 Star。

**[更多精彩内容](https://github.com/liyongning/blog/blob/main/README.md)**
