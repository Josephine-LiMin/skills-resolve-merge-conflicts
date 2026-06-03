# 学习报告：course 3 (AI in Actions)

本项目是 GitHub Skills 课程中的 **AI in Actions**（在我们的沟通中命名为 **course 3**），主要介绍如何使用 **GitHub Models** 服务将 AI 大语言模型无缝集成到 GitHub Actions 自动化工作流中，以实现智能化的任务分析与反馈。

## 一、核心知识点梳理

1. **GitHub Models 服务与推理 API**
   - GitHub Models 是一个集成了多家领先提供商（如 Mistral, Meta Llama 等）AI 模型的托管服务。
   - 提供了统一的推理端点（Inference API），允许开发者通过 Actions 运行环境安全地调用 AI 模型，无需管理第三方云服务的 API 密钥。

2. **内置免密认证与最小化权限原则**
   - 权限机制：通过在 YAML 配置文件中明确声明 `permissions: models: read`，授权内置的 `${{ secrets.GITHUB_TOKEN }}` 能够以只读权限访问 GitHub Models 推理 API。
   - 这避免了配置外部 Secrets 的繁琐步骤，提升了开发安全性。

3. **工作流编排三部曲（Workflow Composition Pattern）**
   - AI 工作流的最佳实践通常包含以下三个连续步骤：
     - **🔍 上下文收集（Context Gathering）**：从触发事件（如 Issue、PR 详情）、代码文件或 API 输出中收集原始数据。
     - **🤖 AI 模型处理（AI Processing）**：将数据输入 `actions/ai-inference@v2` 动作，结合特定的 System Prompt 设定角色和输出格式要求，控制生成内容。
     - **🚀 应用处理结果（Applying Results）**：提取 AI 的输出（`response`），利用其他动作（如评论、更新文件、标签分类）产生实际的仓库变更。

---

## 二、学习中的难点

1. **`models: read` 权限缺失导致推理 API 鉴权失败**
   - 初次配置 AI Inference 动作时，若忘记声明 `permissions: models: read`，工作流运行后会抛出 API 401/403 未授权错误。
   - **解决方案：** 必须在 YAML 顶层或 Job 层级的 `permissions` 块中添加 `models: read`。如果工作流还需要发表 Issue 评论，还需同步声明 `issues: write`。

2. **Git Push 冲突与本地/远程基线对齐**
   - 在将我们新写好的 `ask-ai.yml` 提交并推送到远程 `main` 分支时，被 GitHub 拒绝。这是由于当第一步的校验逻辑启动时，Mona 教学机器人自动提交了文件并修改了 `README.md` 的内容。
   - **解决方案：** 运行 `git pull` 命令以合并远程变更，在本地处理合并后再次执行推送。

3. **YAML 中大括号表达式的转义处理**
   - 在编写带有 Actions 上下文（如 `${{ github.event.issue.body }}`）的 prompt 时，如果是通过自动化模板渲染，有时需要用 `{% raw %}` 与 `{% endraw %}` 包裹表达式以防止其被静态解析器提前求值。
   - **解决方案：** 了解 GitHub Actions 语法解析树，确保在最终的 `.yml` 源码中表达式是能够正常被 Actions 引擎调用的原生 `${{ ... }}` 结构。

---

## 三、实践操作与收获

1. **完整操作链路**
   - **手动测试（Step 1）**：创建 `.github/workflows/ask-ai.yml`，设置触发为 `workflow_dispatch`（手动触发）。调用 `actions/ai-inference@v2` 并使用默认模型获取一条关于编程的笑话，最终将其追加写入当前运行的 `$GITHUB_STEP_SUMMARY`（工作流摘要）中。通过 GitHub CLI 触发运行，成功看到了渲染的 markdown 摘要。
   - **事件驱动测试（Step 2）**：创建 `.github/workflows/issue-completeness.yml`，设定触发条件为新 Issue 创建。当 Issue 开启时，捕获其标题和 Body 上下文，传递给 AI 进行完整性分析（要求 AI 作为助手给出补充信息建议，并以 Markdown 结构回复），最后调用 `peter-evans/create-or-update-comment@v4` 动作，自动在 Issue 下方生成分析结果。
   - **闭环验证**：创建了一个模拟的 "Login form throwing 500 errors on mobile" 的测试 Issue，成功触发工作流运行，在 Issue #2 下方收到了 AI 助手对排查步骤和所需日志的智能化建议，课程主 Issue 成功关闭。

2. **核心收获**
   - 掌握了在 GitHub Actions 中引入轻量化大模型进行文本分析、分类和辅助审查的方法。
   - 熟悉了 GitHub Models 的权限要求，学会了如何安全地处理 Token 和推理结果。
   - 验证了“上下文收集 -> AI 分析 -> API 应用结果”这一黄金编排模式在实际项目中的自动化威力。

---

## 四、学习遇到的困惑与疑问

1. **推理模型的速率限制（Rate Limits）与生产环境承载力**
   - GitHub Actions 内置的 Models API 在高并发或者超大项目频繁触发时（例如大量 PR 提交），是否会因为触发频率过高（Rate Limits）而导致自动化任务挂起或失败？
   - *思考/解决方向：* 教学版 API 有严格的每日额度限制，如果需要在生产流水线中常态化使用，应在 Action 中切换配置，将 `token` 参数由 GITHUB_TOKEN 替换为具备企业级付费或 Azure OpenAI 自备密钥的 Secret Token，以获得更高的配额和稳定性。

2. **AI 输出内容的不确定性与安全性控制**
   - AI 在自动生成 Issue 评论或代码分析时，可能会产生幻觉，或者由于模型生成包含不安全的链接、代码段而导致风险。如何对 AI 步骤的 outputs 进行净化？
   - *思考/解决方向：* 可以在 Workflow 中引入后续的自动化安全检查步骤，过滤 AI 返回值中的非安全字符，或者在 System Prompt 层面强化防护规则，限制其生成除文本和合规建议以外的实体。
