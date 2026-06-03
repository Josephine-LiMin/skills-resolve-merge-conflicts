# 学习报告：Agent Orchestration: Build Your AI Dream Team

本项目是 GitHub Skills 课程中的 **Agent Orchestration: Build Your AI Dream Team**，旨在帮助开发者理解和掌握如何通过 GitHub Copilot CLI 协同和编排一组专业化分工的自定义 AI 代理（Agents），共同协作构建项目面板（Project Pulse Dashboard）的完整研发链路。

## 一、核心知识点梳理

1. **AI 智能代理编排（Agent Orchestration）设计思想**
   - 相比于单一 Prompt 直接生成整个系统，智能代理编排更适用于复杂项目的生命周期管理。
   - 通过角色化和专业化拆分，利用不同特长和参数大小的 LLM 来担任特定角色，能显著提高复杂任务的生成成功率，减少幻觉和长上下文损耗。

2. **四大 AI 代理团队角色与模型配置**
   - **Orchestrator（协调者）**：负责项目统筹、任务分解、代理派发、集成验证以及编写最终报告。搭载模型：`Opus 4.7`。
   - **Planner（规划者）**：分析代码资产、定义实施阶段、梳理文件依赖关系、识别可并行项并设计验证指标。搭载模型：`Opus 4.7`。
   - **Designer（设计者）**：关注 UI/UX、交互体验、页面布局（如 Flexbox/Grid 响应式设计）与精细样式（如阴影和圆角）。搭载模型：`Gemini 3.1 Pro`。
   - **Coder（编码者）**：负责编写具体的 HTML 模板逻辑、处理 JSON 数据加载和解析，以及创建符合规范的 VS Code 本地开发运行配置文件。搭载模型：`GPT-5.5`。

3. **项目交付与验证标准规范**
   - 严格的静态校验规则：对代码文件中的语法、关键字（如 `status`, `recentActivity`, `priority`）、特定的 CSS 选择器（如 `.dashboard`, `.project-card` 及其属性 `border-radius`, `box-shadow`）以及严格不含注释的 `.vscode/launch.json` 文件进行自动测试。
   - 最终交付物：包括项目看板前端 `app/index.html`、样式表 `app/styles.css`、数据源 `app/project-data.json`、启动配置文件 `.vscode/launch.json`，以及最终编排总结 `docs/final-handoff.md`。

---

## 二、学习中的难点

1. **严格的 JSON 规范与校验**
   - 在配置 `.vscode/launch.json` 时，习惯于写入 JSON 注释以供解释，但 GitHub Actions 自动校验器使用严格的 JSON 解析器进行检查，包含注释或尾随逗号会导致 Step 3 失败。
   - **解决方案：** 确保 `.vscode/launch.json` 采用绝对干净的 Standard JSON 格式，去除所有注释、保持严格的双引号包围，并且把 launch 的工作目录指向 `/app` 文件夹。

2. **自动化校验中特定文本字面量的缺失匹配**
   - Step 3 的 Action 检查器（如 `skills/action-keyphrase-checker`）会对 HTML、CSS 和 JSON 文件进行极严格的关键字词检测。例如，HTML 文件中必须硬编码匹配到 `status`、`recentActivity` 和 `priority`；CSS 必须包含 `.dashboard` 与 `.project-card` 选择器，且圆角和阴影属性名称必须分毫不差。
   - **解决方案：** 在开发阶段不要使用过于复杂的组件库或缩写别名，必须在编写 prompt 和代码时精准融入上述单词，避免因语义近义词替换（如使用 `activity` 代替 `recentActivity`）导致 Action 跑不通。

3. **终期交付文档 `docs/final-handoff.md` 的关键字覆盖**
   - Step 4 工作流要求最终的交付总结中必须同时涵盖所有参与代理名称、所修改的文件路径、VS Code 的配置名称 "Run Project Pulse Dashboard"，以及至少一个包含 `validation` 的二级标题和包含 `handoff` 的二级标题。
   - **解决方案：** 在生成 `docs/final-handoff.md` 前，仔细审查了 `.github/workflows/4-step.yml` 中的正则表达式和关键字断言，确保生成的 Markdown 标题和内容完整无缺地覆盖了所有检查项。

---

## 三、实践操作与收获

1. **完整操作链路**
   - **第一阶段（Meet the team）**：审查 `.github/agents/` 下各代理的 markdown 定义，将代理的分工和所使用的不同模型（`Opus 4.7`, `GPT-5.5`, `Gemini 3.1 Pro`）记录在 `docs/agent-team.md` 中。
   - **第二阶段（Plan the dashboard）**：利用 Orchestrator 和 Planner 生成项目开发路线图，并在 `docs/project-pulse-plan.md` 中明确依赖项、分工和可并行逻辑。
   - **第三阶段（Build the dashboard）**：编排 Designer 与 Coder 完成前端面板，包含项目卡片样式渲染（`.project-card`），数据源解析（`projects` 结构体），并利用 Python 3 简易服务器（`python3 -m http.server 5500`）配置本地运行预览端口。
   - **第四阶段（Validate & Handoff）**：由 Orchestrator 对项目进行了代码 and 设计验证，并撰写包含代理成果、使用限制和后续工作计划的 `docs/final-handoff.md`，推送到 GitHub 触发自动化验收，成功使主 Issue #1 关闭。

2. **核心收获**
   - 实践了多 Agent 团队的协作模式（Multi-Agent System Orchestration）。
   - 了解了针对不同的业务场景（逻辑推理、图形界面、纯编码实现）选择性价比和专业性最适用的 LLM 基础模型。
   - 掌握了如何利用 VS Code 启动配置（`launch.json`）和简易 HTTP 服务器来进行 Web 项目的敏捷联调。

---

## 四、学习遇到的困惑与疑问

1. **复杂依赖项的多轮迭代与死锁问题**
   - 当 Designer 和 Coder 需要同时修改同一个文件时（例如，Coder 需要修改 HTML 的交互逻辑，Designer 需要修改 HTML 的结构以融入特定的 CSS 选择器），如何协调两个并发代理的代码修改而避免出现代码冲突或覆盖？
   - *思考/解决方向：* 需要设计更加细粒度的文件拥有权划分，或者让 Orchestrator 采用串行而不是并行的方式来调度冲突阶段的编码工作，以维护分支/代码树的整洁度。

2. **跨模型上下文对齐的损失**
   - 当我们在编排中使用不同的 LLM 模型（如 Opus 传递到 Gemini，再传递到 GPT）时，由于它们的 Prompt 遵循偏好和上下文表示格式不完全相同，信息在多次中转中是否会出现语义丢失或偏差？
   - *思考/解决方向：* Orchestrator 作为中枢核心，需要承担将输出转换并格式化为标准输入格式（如 JSON）的职责，从而为每个 specialist agent 提供统一清晰的上下文结构。
