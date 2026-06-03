# 学习报告：实验四 course1 (Hello GitHub Actions)

本项目是 GitHub Skills 课程中的 **Hello GitHub Actions**（在我们的沟通中命名为 **实验四 course1**），旨在引导开发者迈出 GitHub Actions 持续集成与自动化工作流的第一步，理解 Workflow、Job、Step 和 Event 的基础概念与协同逻辑。

## 一、核心知识点梳理

1. **工作流（Workflow）与配置文件**
   - 工作流是存储在项目仓库中 `.github/workflows/` 目录下的 YAML 格式文件。
   - 它定义了自动化的执行流程，通过声明事件触发器（Events）、执行环境（Runners）以及具体要执行的任务步骤（Steps）。

2. **触发器事件（Trigger Events）**
   - 决定工作流在什么情况下被激活启动。常用的事件包括 `push`（代码推送）、`pull_request`（拉取请求）等。
   - 在本实践中，我们使用了 `pull_request: types: [opened]`，即仅在创建新拉取请求时才会触发欢迎工作流。

3. **任务（Jobs）与步骤（Steps）**
   - **Job（任务）**：代表运行在同一个物理机或虚拟机（Runner）上的一组步骤。默认情况下，工作流中的多个 Job 是并行执行的。本实践中我们指定了 `runs-on: ubuntu-latest`，在最新的 Ubuntu 系统环境中执行任务。
   - **Step（步骤）**：是 Job 内按顺序依次执行的最小任务单元。可以运行脚本指令（`run`）或者调用现成的 GitHub Actions（`uses`）。

4. **安全上下文与 Token 管理**
   - 工作流通过声明 `permissions` 块来控制安全权限范围，例如本课程中需要对 PR 发表评论，声明了 `permissions: pull-requests: write`。
   - 通过系统自动注入的临时凭证 `${{ secrets.GITHUB_TOKEN }}` 进行 GitHub API 认证，以安全地调用 GitHub CLI（`gh`）执行评论操作。

---

## 二、学习中的难点

1. **GitHub Actions 默认权限限制导致 API 操作失败**
   - 在首次运行自动欢迎工作流时，若直接使用 GitHub CLI 执行 `gh pr comment`，极易因仓库默认权限设置为“只读（Read-only）”而导致报错：`Resource not accessible by integration`。
   - **解决方案：** 必须使用 GitHub CLI 或在网页端设置中将 Workflow 默认写入权限开启。我们在克隆仓库后主动执行了 API 权限设置命令，从而规避了该权限拦截问题：
     ```bash
     gh api --method PUT repos/Josephine-LiMin/skills-hello-github-actions/actions/permissions/workflow -F default_workflow_permissions=write -F can_approve_pull_request_reviews=true
     ```

2. **不完整的 YAML 文件导致的工作流 lint 错误**
   - 在 Step 1 阶段，课程要求我们先编写一个不包含任何 `jobs` 定义的 YAML 骨架，并将其推送上去。这会导致 GitHub Actions 验证器自动报出 YAML 结构不完整的 lint 失败（Workflow validation failed）。
   - **解决方案：** 这是课程设计中的一步，用于循序渐进地构建任务。了解这是由于暂未配置具体的 `jobs` 所致，在 Step 2 补充完整的任务与环境配置后，错误会自动消除并验证成功。

3. **YAML 语法缩进敏感性**
   - YAML 配置文件对空格缩进具有极高敏感性，容易因为 `steps` 与 `jobs` 的层级关系没有完全对齐而造成解析失败。
   - **解决方案：** 使用代码编辑器（如 VS Code）内置的 YAML 格式校验工具，严格遵循缩进规范。

---

## 三、实践操作与收获

1. **完整操作链路**
   - 创建分支 `welcome-workflow`。
   - 编写不完整的配置文件 `.github/workflows/welcome.yml` 并推送，完成 Step 1 验证。
   - 为配置文件添加 `jobs` 声明，指定运行环境为 `ubuntu-latest` 并推送，完成 Step 2 验证。
   - 在 Job 下方声明 `steps` 执行步骤，调用环境变量 `PR_URL` 与临时授权 Token，执行 shell 命令 `gh pr comment "$PR_URL" --body "Welcome to the repository!"`，完成 Step 3 验证。
   - 利用 GitHub CLI 创建从 `welcome-workflow` 到 `main` 分支的 Pull Request，触发 `Post welcome comment`工作流运行，成功看到机器人在拉取请求下方自动追加了一条欢迎留言，完成 Step 4 验证。
   - 将 PR 合并至主分支，使工作流全局生效，关闭 Issue，完成课程。

2. **核心收获**
   - 熟悉了 GitHub 仓库自动化开发流水线的建立过程，掌握了使用 YAML 定义简单任务的格式。
   - 学会了在工作流中利用 GitHub CLI 进行 API 调用，能够灵活地获取上下文属性（如 `${{ github.event.pull_request.html_url }}`）。
   - 验证了分支合并与工作流全局生效的关系。

---

## 四、学习遇到的困惑与疑问

1. **`secrets.GITHUB_TOKEN` 的生命周期与权限分配**
   - 在执行中，系统注入的 `GITHUB_TOKEN` 到底是由谁签发的，其权限与个人 Access Token 有何区别？
   - *思考/解决方向：* `GITHUB_TOKEN` 是由 GitHub App 自动生成的临时密钥，其权限上限受到工作流中 `permissions` 声明以及仓库设置的交集控制。一旦工作流运行结束，该 Token 即失效，大大降低了凭证泄露的安全风险。

2. **如何调试在 Actions 虚拟环境中运行失败的步骤**
   - 如果 `gh pr comment` 失败且日志信息不足，该如何本地调试工作流？
   - *思考/解决方向：* 可以引入一些测试工具（如 `act`），它们允许在 Docker 本地容器内模拟 GitHub Actions 执行环境，从而在本地进行断点或日志调试，节省大量在线推代码重试的成本。
