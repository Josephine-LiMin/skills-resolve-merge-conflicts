# reusable-workflows

## 一、核心知识点梳理

1. **可复用工作流（Reusable Workflows）的基本概念**：
   - 可复用工作流是可以被其他工作流调用的工作流。它的主要目的是消除重复的流水线代码，将常用的构建、测试或部署逻辑集中管理，实现标准化的 CI/CD 流程。
   - 使用 `workflow_call` 触发器使工作流具备可复用能力。在 `workflow_call` 下可以定义 `inputs`（输入参数）和 `outputs`（输出结果），以便调用方传递定制数据并接收反馈。

2. **调用语法与引用方式**：
   - 可以在同仓库中通过相对路径引用：`uses: ./.github/workflows/reusable-node-quality.yml`。
   - 也可以在不同仓库中通过绝对路径和版本标识引用：`uses: owner/repo/.github/workflows/workflow.yml@ref`，其中 `ref` 可以是分支名、Release 标签或 Commit SHA，出于安全性考虑推荐使用 Commit SHA 或固定标签。

3. **权限继承与覆盖机制**：
   - 默认情况下，被调用的可复用工作流会继承调用方工作流的 Token 权限，但无法获取超出调用方上限的权限。
   - 调用方工作流（Caller Workflow）可以通过在作业中配置 `permissions` 块，为具体的被调用作业显式扩大或缩小权限限额（例如 `pages: write` 或 `id-token: write`）。

4. **作业依赖（needs）与多重作业组合**：
   - 通过 `needs` 关键字建立多个可复用作业的先后执行次序。例如，必须先执行并通过 `quality`（质量检测）后，才能执行 `github-pages`（编译部署）。
   - 通过作业级别的 `if: always()` 条件逻辑，无论依赖的前序作业成功与否，都可以确保执行收尾通知（如向 PR 发送状态报告）。

## 二、学习中的难点

1. **环境依赖引发的编译挂起（Playwright Hangs）**：
   - 在较新的 Node.js 版本（如 Node 24.x）下，使用 Playwright 下载安装浏览器二进制及其系统依赖 `npx playwright install --with-deps chromium` 存在已知的不兼容 Bug，会导致在 `extract-zip` 压缩包解压阶段产生死锁并无限挂起（运行超过十几分钟不中断）。
   - 必须通过强制回退到稳定的 Node 20.x 或 22.x LTS 环境，才能彻底解决 Playwright 浏览器的安装死锁问题。

2. **嵌套工作流的排查与日志审计**：
   - 可复用工作流运行时，GitHub Actions 界面会将所有的被调用作业平铺展开。在排查报错日志时，需要理清究竟是调用方文件语法配置错误，还是被调用模板文件内部逻辑出错。

## 三、实践操作与收获

1. **操作流程**：
   - 实例化 `skills-reusable-workflows` 仓库，并在本地建立 `reusable-workflows` 分支。
   - 创建可复用工作流 `.github/workflows/reusable-node-quality.yml`，定义 `workflow_call` 接收 Node.js 版本号，并封装 `lint`、`tests`、`e2e` 三个日常 CI 任务。
   - 编写调用工作流 `.github/workflows/ci.yml`，触发器设为 `pull_request`，调用封装好的质量检测流。提交并开启 PR，顺利验证继承流的功能。
   - 配置 GitHub Pages 的部署源为 GitHub Actions 方式。
   - 在 `ci.yml` 中追加 `github-pages` 部署作业与 `comment` 作业，使用 `Peter-Evans` 的 PR comment 组件，在部署完成后自动将 Pages 预览地址回复至 Pull Request 下方。

2. **遇到的困难及解决方法（反复做的地方）**：
   - **Playwright 依赖死锁（重点复做）**：在 Step 3 执行时，由于在 `ci.yml` 中将 Node.js 版本指定为最新的 `24`，导致 Playwright 下载浏览器依赖时在控制台无限挂起，首个测试运行了 17 分钟仍处于 Pending 状态。通过查询技术文档得知该版本存在解压冲突，我通过 `gh run cancel` 手动取消了所有阻塞的 Runs，将 `ci.yml` 中的 `node-version` 修改为稳定的 `"20"` 后重新 Push，流水线在 1 分多钟内即顺利走通，检查项全部 Pass。

## 四、学习遇到的困惑与疑问

1. **可复用流的局部参数硬编码**：
   - 可复用工作流中如果存在强依赖某些本地路径（如 `actions/checkout` 默认 checkout 调用方仓库，而非模板本身仓库），需要小心参数与路径的作用域混淆。
2. **跨仓库复用流的密钥安全**：
   - 被调用的工作流无法直接访问调用方仓库的 Secrets，若需访问，必须通过 `secrets: inherit` 隐式传递，或在 inputs/secrets 块中显式传递。如何在保障企业密钥安全的前提下实现跨项目复用是运维设计的重要课题。
