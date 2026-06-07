# create-ai-powered-actions

## 一、核心知识点梳理

1. **GitHub Models 的集成与使用**：
   - GitHub Models 提供了多款主流大语言模型的 API 访问能力（如 `openai/gpt-4.1-mini`），并且可以通过标准的 `https://models.github.ai/inference` 端点以 OpenAI 兼容的 SDK 进行访问。
   - 调用时需要配置特殊的读权限，例如在 Workflow 中显式声明 `permissions` 的 `models: read`。

2. **结构化输出（Structured Outputs）与 Zod**：
   - 传统的大语言模型输出为自由格式的纯文本，这在自动化脚本和工作流中难以被机器准确解析和做条件逻辑处理。
   - 结构化输出保证模型严格遵循预定义 Schema 生成 JSON。本实验中使用 Zod 库定义 `JokeRatingSchema`，指定模型必须返回包含 `is_joke` (boolean)、`score` (number)、`humor_type` (string) 和 `feedback` (string) 字段的 JSON。
   - 使用 `openai` 客户端的 `chat.completions.parse` 方法自动验证并解析响应结果。

3. **Workflow 中的 JSON 解析与条件判断**：
   - 在 Actions 工作流中，可以使用 `${{ fromJSON(steps.rate-joke.outputs.result).is_joke }}` 表达式对 Action 输出的 JSON 字符串进行即时解析和条件判断（如 `if: fromJSON(...).is_joke == true`），使工作流具备智能拦截或差异化处理逻辑的能力。

## 二、学习中的难点

1. **结构化数据格式解析**：
   - 必须确保大模型严格生成结构化 JSON 数据，如果返回的数据格式与 Zod Schema 不符会引发运行期解析异常。
   - 自定义 Action 中通过 `core.setOutput("result", rating)` 导出结果时，需要了解 `@actions/core` 能够自动将非字符串类型（如对象/数组）调用 `JSON.stringify` 转化为 JSON 字符串，以便后续的 `fromJSON` 表达式解析。

2. **多进程并发与冲突的处理**：
   - 该课程使用了多个并行工作流，例如在 push 时触发 Step 逻辑，而 Step 逻辑执行时又会自动提交 commits 回仓库，若本地没有先 pull 很容易在下一次 push 时遭遇 remote 拒绝冲突。

## 三、实践操作与收获

1. **操作流程**：
   - 新建并克隆课程模板仓库 `skills-create-ai-powered-actions`，开启动作运行写权限。
   - 安装 `openai` 依赖。编写 `action.yml` 元数据，声明接收 `joke` 和 `token` 输入，输出 `result`。
   - 创建 `src/rateJoke.js`，初始为普通的 GPT API 调用并打印纯文本评估。
   - 编写 `src/main.js` 和 `src/index.js`，执行打包脚本。
   - 编写 `.github/workflows/rate-joke.yml` 工作流文件并推送到远端。
   - 在 issue 留言笑话，成功触发 AI 对该笑话评分为 "7/10"。
   - 引入 `zod` 依赖，重构 `src/rateJoke.js` 采用结构化输出。
   - 更新 workflow 文件，在 `Update comment` 步骤加上 `if` 条件，仅当 `is_joke` 为真时才进行更新。
   - 测试再次发一条笑话以及一条普通对话，确认笑话能够输出包含 Humor Type, Score, Feedback 的结构化格式，而普通对话则被自动过滤不发反馈。

2. **遇到的困难及解决方法（反复做的地方）**：
   - **Git Push 拒绝问题**：在完成 Step 1 和 Step 2 推送时，由于 GitHub 自动构建的 bot 触发了新的 commits 修改了项目仓库，导致本地直接 `git push` 报错冲突。解决办法是分步运行 `git pull --rebase` 和 `git push`，保证历史记录的线性并顺利合并提交。

## 四、学习遇到的困惑与疑问

1. **Zod 和 OpenAI SDK 结合时的错误处理**：
   - 如果用户输入的数据导致大模型拒绝回答（Refusal），或者在网络波动时出现超时，如何优雅地捕获并输出友好的降级信息（例如 `is_joke` 为 `false` 且其他字段为 `null`），而非直接造成 Workflow 报错中止？这是生产环境中需要设计和补充的重要部分。
2. **GitHub Models 的速率限制（Rate Limits）**：
   - GitHub Models 提供免费的测试額度，但也有每天/每小时的限制。如果不限制调用频率或被滥用，很容易触发 429 报错导致工作流阻塞，实际商业化落地的系统中通常需要额外的 Token 缓存或备用 API。
