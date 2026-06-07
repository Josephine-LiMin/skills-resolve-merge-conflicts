## 一、核心知识点梳理

1. **工作流制品 (Workflow Artifacts) 的概念**：
   - GitHub Actions 每次运行的作业都在一个全新的虚拟环境（或容器）中，作业结束后临时生成的文件会随之丢失。
   - 制品（Artifacts）提供了在作业运行期间保存文件（如构建产物、测试报告、日志）的机制，使得这些文件在作业结束后依然可被下载、保留或供后续作业复用。
2. **制品的上传与下载**：
   - 使用官方的 `actions/upload-artifact` 动作进行上传。
   - 使用官方的 `actions/download-artifact` 动作进行下载。
3. **单文件直接上传与在线预览**：
   - 默认情况下制品会以 `.zip` 文件格式归档下载。
   - 在 `actions/upload-artifact` 中可以通过设置 `archive: false`，使得单个文件不经过压缩直接上传。这对于 HTML 格式的测试报告（例如 Playwright 的 `index.html` 报告）极为方便，允许开发者在 GitHub UI 中直接点击并在浏览器中预览，而不需要下载并解压。
4. **跨工作流制品共享**：
   - 可以通过 `workflow_run` 触发器，在某个工作流完成后自动运行另一个工作流。
   - 使用 `actions/download-artifact` 时，可以通过指定 `run-id`（利用触发事件的上下文数据 `${{ github.event.workflow_run.id }}`）和 `github-token`，跨越工作流运行的边界下载前置工作流生成的制品。
5. **部署环境保护 (Deployment Environments)**：
   - 学习了如何通过 GitHub 仓库设置中的 Environments 对关键环境（如 `prod`）配置“Required reviewers”审批 gate。这可以确保自动部署任务在到达生产环境前暂停，直到指定审核人员审批通过后再继续运行。

## 二、学习中的难点

1. **跨工作流制品的下载机制**：
   - 需要理解在被触发的 `deploy-prod.yml` 中，如何定位源工作流的制品。需要准确使用 `${{ github.event.workflow_run.id }}` 动态获取运行 ID，并且需赋予作业 `actions: read` 权限以读取其他工作流的运行记录。
2. **Node 24 环境下 Playwright 依赖下载挂起风险**：
   - 在之前课程的实践中，我们发现使用 `node-version: 24` 时，运行 `npx playwright install --with-deps chromium` 会因为 Node 24 的某些 zip 解压锁定机制导致安装过程无限期卡死挂起。
   - 本次实验在编写 workflow 时，主动将推荐的 `node-version: 24` 调整为了 `node-version: 20`，从而成功规避了挂起卡死问题，极大地保障了 E2E 测试步骤的稳定高效运行。

## 三、实践操作与收获

1. **顺利完成四个步骤的实操**：
   - **Step 1**：成功编写了单元测试与覆盖率收集工作流 `tests.yml`，并将 `coverage/` 目录上传为制品。
   - **Step 2**：引入 Playwright 的 E2E 测试作业，并通过 `archive: false` 配置将 HTML 报告直接上传供浏览器即时预览。
   - **Step 3**：新建 `build-deploy.yml` 实现了对项目的 Vite 构建，将构建生成的 `dist` 目录打包上传为 `octomatch` 制品，并在下游部署作业中下载并利用 `tree website` 成功验证了解压后的目录树。
   - **Step 4**：新建 `deploy-prod.yml` 实现了跨工作流触发，并安全地下载了 `Build and Deploy` 产生的构建产物模拟生产环境部署。
2. **体会到持续集成的高效管理**：
   - 熟悉了 GitHub Actions 里的数据共享链路：同一工作流内（跨作业）通过 `upload/download-artifact` 避免重复构建；不同工作流之间通过 `workflow_run` 结合 `run-id` 实现跨任务传递，使流水线设计更加解耦和模块化。

## 四、学习遇到的困惑与疑问

1. **单文件直接预览的局限性**：
   - `archive: false` 目前只能应用于**单个文件**。如果我们的 HTML 报告包含大量的外部静态资源（如分离的 CSS、JS、图片等），只上传 `index.html` 会导致预览时排版丢失和资源加载 404。在这种复杂场景下，除了打包为 `.zip` 供下载，如何实现更好的在线预览展示（例如是否能配合 Pages 发布，或者将多文件打包成单文件 HTML 形式）？
2. **跨工作流运行的权限边界**：
   - `deploy-prod` 由 `workflow_run` 触发，这属于特权触发，即使是来自 Fork 仓库的 PR 触发，在主仓库中运行的 `workflow_run` 也有写权限和读取 Secrets 的权限。在实际企业级开发中，我们需要如何对 `workflow_run` 工作流进行安全审计，以防止恶意注入？
