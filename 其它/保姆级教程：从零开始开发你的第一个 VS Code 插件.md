**当学习成为了习惯，知识也就变成了常识。** 感谢各位的 **关注**、**点赞**、**收藏**和**评论**。

新视频和文章会第一时间在微信公众号发送，欢迎关注：[李永宁lyn](https://gitee.com/liyongning/typora-image-bed/raw/master/202202171742614.jpg)

文章已收录到 [github 仓库 liyongning/blog](https://github.com/liyongning/blog)，欢迎 Watch 和 Star。

# 保姆级教程：从零开始开发你的第一个 VS Code 插件

> 👨‍💻 **目标读者**：如果你对 VS Code 插件开发感兴趣，但完全零基础，这篇教程就是为你准备的。
>
> ⏱ **预计耗时**：30 分钟
>
> 🎯 **最终成果**：一个能弹出 Hello World 提示框，且能随编辑器自动启动的完整插件。

---

## 🚀 1. 为什么开发 VS Code 插件？

VS Code 是目前最流行的编辑器之一，它的强大之处在于**丰富的生态系统**。作为开发者，你一定用过各种插件来提高效率。

**但你有没有想过：**
*   "如果有个工具能自动帮我生成这段重复代码就好了..."
*   "要是能把这个内部工具集成到编辑器里该多方便..."

其实，开发一个 VS Code 插件比你想象的要简单得多！今天我们就通过一个最简单的 Hello World 例子，带你走完**开发 -> 调试 -> 打包**的全流程。

### ⚡️ 特别篇：Cursor、Trae 也能用吗？

你可能正在使用 **Cursor**、**Trae** 或其他新兴的 AI 编辑器。你可能会问：“我学的这个是 VS Code 插件开发，能在这些编辑器里用吗？”

**答案是肯定的！而且非常丝滑。**

1.  **同源架构**：Cursor 和 Trae 等编辑器，底层很大程度上是基于 VS Code 的开源核心 (Code OSS) 构建的。这意味着它们天生就拥有相同的“基因”。
2.  **一次开发，到处运行**：绝大多数标准的 VS Code API 在这些编辑器中是完全通用的。你编写的 VS Code 插件，99% 的情况下可以直接安装在 Cursor 或 Trae 中，无需修改任何代码。
3.  **如何安装**：
    *   **开发时**：你可以直接用 Cursor 打开插件项目，按 F5 调试，体验和 VS Code 一模一样（甚至更好，因为有 AI 辅助）。
    *   **使用时**：如果你打包出了 `.vsix` 文件，直接将其拖入 Cursor/Trae 的扩展侧边栏即可安装。很多编辑器甚至直接共用了 VS Code 的插件市场。

所以，**学会了 VS Code 插件开发，你就自动解锁了为所有类 VS Code 编辑器开发插件的技能！** 这是一笔性价比极高的技术投资。

---

## 🛠 2. 环境准备

在开始之前，请确保你已经安装了以下工具：

1.  **Node.js** (建议 v18 LTS 或 v20 LTS 版本，与现代 VS Code 环境一致)
    *   检查方法：在终端运行 `node -v`
2.  **Git** (用于版本控制，可选)
3.  **编辑器**：当然是 VS Code 或者 Cursor！

---

## 🏗 3. Step 1: 搭建项目骨架

虽然官方提供了 `yo code` 脚手架，但为了让你理解每个文件的作用，我们采用**手动搭建**的方式。

请创建一个文件夹 `vscode-plugin-demo`，并在其中创建以下文件结构：

```text
vscode-plugin-demo/
├── package.json        // 核心配置文件
├── tsconfig.json       // TypeScript 配置
├── .gitignore          // Git 忽略文件
├── src/
│   └── extension.ts    // 插件代码入口
└── .vscode/
    ├── launch.json     // 调试配置
    └── tasks.json      // 构建任务
```

### 3.1 核心配置 `package.json`

这是插件的“身份证”。把以下代码复制到你的 `package.json` 中：

```json
{
  "name": "vscode-plugin-demo",
  "displayName": "VS Code Plugin Demo",
  "description": "一个用于入门学习的 VS Code 插件 Demo",
  "version": "0.0.1",
  "publisher": "yourname",  // 发布时需要修改为你的发布者ID
  "engines": {
    "vscode": "^1.85.0"     // 依赖的 VS Code 最低版本
  },
  "categories": ["Other"],
  
  // 🟢 关键点 1：激活事件
  // VS Code 1.74+ 支持“隐式激活”（只要注册了命令就会自动激活）。
  // 这里保留 "onStartupFinished" 是为了演示“编辑器启动后立即自动执行”的效果。
  "activationEvents": [
    "onStartupFinished"
  ],
  
  "main": "./out/extension.js",
  
  // 🟢 关键点 2：贡献点
  // 告诉 VS Code 这个插件提供了什么功能
  "contributes": {
    "commands": [
      {
        "command": "demo.helloWorld",  // 命令 ID，代码中会用到
        "title": "Demo: Hello World"   // 命令面板中显示的标题
      }
    ]
  },
  
  "scripts": {
    "vscode:prepublish": "npm run compile",
    "compile": "tsc -p ./",
    "watch": "tsc -watch -p ./"
  },
  "devDependencies": {
    "@types/vscode": "^1.85.0",
    "@types/node": "^18.14.6",
    "typescript": "^5.3.3"
  }
}
```

### 3.2 编译配置 `tsconfig.json`

告诉 TypeScript 怎么编译代码：

```json
{
  "compilerOptions": {
    "module": "commonjs",
    "target": "es2020",    // 推荐 es2020+，产物更简洁
    "outDir": "out",
    "lib": ["es6"],
    "sourceMap": true,
    "rootDir": "src",
    "strict": true
  },
  "exclude": ["node_modules"]
}
```

### 3.3 打包瘦身 `.vscodeignore` (关键！)

**这是一个新手最容易忽略的文件**。如果不配置它，你的源码、Git 记录、测试文件全都会被打包进插件安装包里，导致体积巨大且泄露源码。

创建 `.vscodeignore` 文件，填入以下内容：

```text
.vscode/**
.vscode-test/**
src/**
.gitignore
tsconfig.json
webpack.config.js
node_modules/**
out/test/**
**/*.map
```

*(注意：`node_modules` 必须忽略，因为打包工具会自动分析依赖并将需要的代码打包进最终文件，不需要把整个文件夹带上)*

---

## 💻 4. Step 2: 编写核心代码

现在，让我们来写插件的逻辑。在 `src/extension.ts` 中写入：

```typescript
// 引入 VS Code 模块
import * as vscode from 'vscode';

// 🔴 activate 函数：插件的入口
// 当满足 package.json 中 activationEvents 的条件时，会调用此函数
export function activate(context: vscode.ExtensionContext) {

    // 1. 打印日志（调试控制台可见）
    console.log(`[${new Date().toLocaleTimeString()}] 插件已激活！`);

    // 2. 注册命令
    // 'demo.helloWorld' 必须与 package.json 中的 command 字段一致
    let disposable = vscode.commands.registerCommand('demo.helloWorld', () => {
        
        // 3. 执行逻辑：弹出一个提示框
        vscode.window.showInformationMessage('Hello World from VS Code Plugin Demo!');
    });

    // 4. 注册资源释放
    // 将命令放入 subscriptions 中，确保插件卸载时自动清理资源
    context.subscriptions.push(disposable);
}

// deactivate 函数：插件停用时调用（通常留空）
export function deactivate() {}
```

---

## 🐞 5. Step 3: 一键调试 (F5)

为了能像开发普通 Web 项目一样按 F5 调试，我们需要配置 `.vscode` 目录下的两个文件。

### 5.1 `launch.json` (启动配置)

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Run Extension",
            "type": "extensionHost",
            "request": "launch",
            "args": ["--extensionDevelopmentPath=${workspaceFolder}"],
            "outFiles": ["${workspaceFolder}/out/**/*.js"],
            "preLaunchTask": "${defaultBuildTask}" // 启动前先运行构建任务
        }
    ]
}
```

### 5.2 `tasks.json` (构建任务)

这个配置会在调试前自动运行 `npm run watch`，实现代码修改后实时编译。

```json
{
	"version": "2.0.0",
	"tasks": [
		{
			"type": "npm",
			"script": "watch",
			"problemMatcher": "$tsc-watch",
			"isBackground": true,
			"presentation": { "reveal": "never" },
			"group": { "kind": "build", "isDefault": true },
			"label": "npm: watch"
		}
	]
}
```

---

## ▶️ 6. Step 4: 运行起来！

1.  **安装依赖**：
    在终端运行 `npm install`。

2.  **启动调试**：
    按下 **`F5`** 键。此时会自动编译代码，并打开一个新的 VS Code 窗口（称为“扩展开发宿主”）。

3.  **验证效果 1 (自动激活)**：
    在新窗口底部的 "Debug Console" (调试控制台) 中，你应该能看到我们写的日志：
    `[10:30:00] 插件已激活！`
    *说明：这是因为我们配置了 `onStartupFinished`。*

4.  **验证效果 2 (手动执行命令)**：
    *   在新窗口中按下 `Cmd+Shift+P` (Mac) 或 `Ctrl+Shift+P` (Win)。
    *   输入 `Demo: Hello World` 并回车。
    *   🎉 **Bingo!** 右下角弹出了提示框。

---

## 📦 7. Step 5: 打包与发布

开发完成后，怎么给别人用呢？

### 7.1 打包为 .vsix 文件 (私有分享)

如果你只是想发给同事，或者自己用：

1.  打包命令（推荐使用 npx，无需全局安装）：
    ```bash
    npx @vscode/vsce package
    ```
2.  这会生成一个 `.vsix` 文件。你可以直接把它拖到 VS Code 的扩展栏里进行安装。

### 7.2 发布到插件市场 (公开)

如果你想让全世界都能搜到你的插件：

1.  去 [VS Code Marketplace](https://marketplace.visualstudio.com/manage) 注册账号。
2.  获取 Personal Access Token (PAT)。
3.  运行 `npx @vscode/vsce login <publisher-id>`。
4.  运行 `npx @vscode/vsce publish`。

*(详细发布流程建议参考官方文档)*

---

## 📚 8. 更多学习资源

恭喜你！你已经掌握了 VS Code 插件开发的核心流程。

想要深入学习，推荐以下资源：
*   **[VS Code API 官方文档](https://code.visualstudio.com/api)**：最权威的指南。
*   **[Extension Samples](https://github.com/microsoft/vscode-extension-samples)**：微软官方提供的几十个示例代码，想实现什么功能先去这里找找。
*   **[VS Code 插件开发指南 (中文)](https://liiked.github.io/VS-Code-Extension-Doc-ZH/)**：社区维护的中文文档。

祝你在插件开发的道路上越走越远！🚀

---

**当学习成为了习惯，知识也就变成了常识。** 感谢各位的 **关注**、**点赞**、**收藏**和**评论**。

新视频和文章会第一时间在微信公众号发送，欢迎关注：[李永宁lyn](https://gitee.com/liyongning/typora-image-bed/raw/master/202202171742614.jpg)

文章已收录到 [github 仓库 liyongning/blog](https://github.com/liyongning/blog)，欢迎 Watch 和 Star。
