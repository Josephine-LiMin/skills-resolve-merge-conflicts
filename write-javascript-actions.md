# write-javascript-actions

## 一、核心知识点梳理

1. **GitHub 自定义 JavaScript Action 的基本结构**：
   - 自定义 JavaScript Action 主要由 JavaScript 源码、依赖库配置（`package.json`）和元数据定义文件（`action.yml`）组成。
   - 元数据文件 `action.yml` 规定了 Action 的基本信息（名称、描述）、输入参数（`inputs`）、输出参数（`outputs`）以及运行方式（`runs`），其中 JavaScript Action 的 `runs` 字段中需要指定 `using` (如 `node24`) 和 `main` 入口文件路径（通常指向打包后的代码 `dist/index.js`）。

2. **GitHub Actions Toolkit**：
   - 核心库 `@actions/core` 提供了与 GitHub Actions 运行时交互的方法，例如通过 `core.setOutput(name, value)` 设置输出参数，让工作流的后续步骤能够读取并消费该输出。

3. **构建与打包工具 `@vercel/ncc`**：
   - 为避免在 GitHub 仓库中提交庞大且臃肿的 `node_modules` 文件夹，自定义 JavaScript Action 必须将所有依赖项与核心逻辑编译打包成单个文件（通常是 `dist/index.js`）。
   - `@vercel/ncc` 是官方推荐的单文件 Node.js 编译打包工具。在 `package.json` 中配置 `"build": "ncc build src/main.js -o dist"` 脚本，能将代码及依赖合并输出到 `dist` 目录。

4. **自定义 Action 的调用与授权**：
   - 可以在同一仓库的 `.github/workflows/` 下编写 workflow，通过 `uses: ./.` 相对路径来调用当前仓库 of 自定义 Action。
   - 使用 `issue_comment` 事件触发时，需要在 workflow 中显式声明 `permissions`（如 `issues: write` 和 `contents: read`）以确保 workflow 拥有向 issue 自动追加评论的权限。

## 二、学习中的难点

1. **依赖的离线构建与打包（ncc）**：
   - 自定义 Action 必须将第三方模块（如 `request-promise`）和打包产物一同推送到远端。如果不运行 `npm run build`，或者没有将生成的 `dist/index.js` 提交至 Git 仓库，Workflow 执行时会因为找不到对应的运行入口而报错。

2. **本地测试与线上运行的差异**：
   - 在本地使用 `node src/main.js` 测试时，`@actions/core` 的 `core.setOutput` 会在终端输出类似 `::set-output name=joke::...` 的特殊格式日志，需要理解这是 GitHub 运行时的特定交互协议。

## 三、实践操作与收获

1. **操作流程**：
   - 初始化 npm 模块并安装依赖（`@actions/core`, `@vercel/ncc`, `request-promise`, `request`）。
   - 在 `.gitignore` 中配置 `node_modules/`，防止依赖污染仓库。
   - 编写 `src/joke.js` 以调用外部 API 随机获取 Dad Joke。
   - 编写 `src/main.js` 作为 Action 入口，使用 `core.setOutput` 导出 joke 变量。
   - 配置 `package.json` 编译打包命令并运行 `npm run build` 生成 `dist/index.js`。
   - 编写元数据 `action.yml` 并创建工作流文件 `.github/workflows/joke-action.yml`。
   - 提交代码后，在 issue 下方评论 `/joke` 成功触发 Workflow 并在 issue 中得到机器人自动生成的幽默回复。

2. **遇到的困难及解决方法（反复做的地方）**：
   - **Shell 命令行兼容性问题**：在本地 Windows PowerShell 环境下直接运行官方提供的 `npm init -y && npm pkg set type=module` 时，PowerShell 报错提示不兼容 `&&` 连接符。我将语句拆开，分步执行 `npm init -y` 和 `npm pkg set type=module`，成功解决了环境兼容性问题。
   - **Git Push 冲突问题**：在推送项目初始化修改时，由于 GitHub 远端有一些初始化 commits 本地没有同步，推送遇到了冲突拒绝。通过在本地执行 `git pull --rebase` 合并远端 commit 后，重新 push 成功解决。

## 四、学习遇到的困惑与疑问

1. **废弃依赖警告**：
   - 在运行 `npm install` 时，控制台输出了关于 `request` 和 `request-promise` 包已被废弃并停止维护的警告（Deprecated warning）。虽然这是课程推荐的依赖库，但在实际生产中建议将其迁移到现代的 `fetch` API 或 `axios` 等替代方案，以避免安全漏洞和兼容性风险。
2. **Node 版本的演进**：
   - `action.yml` 中配置了 `using: node24`。这表明 GitHub Actions 对 Node.js 的新版支持在不断更新。我们需要密切关注不同 GitHub 环境下 Node.js 版本的兼容性与支持周期。
