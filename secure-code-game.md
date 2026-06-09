# Secure Code Game

## 一、核心知识点梳理
《Secure Code Game》通过实战演练让我们识别和修复常见代码及工作流中的安全漏洞。主要知识点包括：
1. **GitHub Actions 安全工作流 (Workflow Security)**：
   - 引入不信任的第三方 GitHub Actions 会增加软件供应链的攻击面（如恶意代码执行、敏感 Token 泄漏等）。
   - 安全规范：对于简单任务应避免引入外部 Actions，转而使用行内脚本（Inline Scripts）；若必须使用，应限制权限，并使用 SHA-256 完整哈希值进行版本锁定。
2. **浮点数精度与数值边界漏洞 (Float Precision and Underflow)**：
   - 浮点数在计算机中采用二进制浮点数形式存储，会带来舍入误差（例如 `0.1 + 0.2 != 0.3`）与精度下溢风险。在金额极大的事务中，微小的扣减（如 `1e19 - 1000`）会被忽略，使得攻击者可以实现免费交易。
   - 安全规范：在处理财务、交易等高精度数值计算时，必须使用高精度数据类型（如 Python 中的 `Decimal`，且实例化时应以字符串形式传入）；同时，必须对输入的商品价格、数量及总订单额设置合理的区间上限与下限限制。
3. **SQL 注入漏洞与参数化查询 (SQL Injection & Parameterized Statements)**：
   - 将未经审查的用户输入直接通过字符串拼接（Concat）传入 SQL 语句中会导致 SQL 注入风险，使攻击者能够执行越权数据读取（SELECT）或恶意数据破坏（DROP TABLE）。
   - 黑名单（Blocklist）字符过滤（如检测 `;`、`--` 等）在复杂的注入手段下极易被绕过，并非根本解决之道。
   - 安全规范：必须使用预编译语句/参数化查询（Parameterized Queries），将逻辑结构与用户输入数据分离。对于设计上提供高自由度脚本执行的接口，应限制其操作类型（如仅允许 SELECT）。

## 二、学习中的难点
1. **单元测试与安全防御的兼容性**：
   - 在 Level 4 (Data Bank) 中，单元测试非常严格地校验了控制台输出中的 SQL 语句字符串（例如 `expected_output = "[QUERY] SELECT * FROM stocks WHERE symbol = 'MSFT'"`）。
   - 直接使用参数化占位符（如 `SELECT * FROM stocks WHERE symbol = ?`）会改变执行的 SQL 并破坏了原始输出日志，导致原本用于常规功能测试的 Assert 报错失败。
   - 难点在于如何在不修改单元测试校验逻辑的情况下完成底层执行的安全重构。我们通过在构造展示查询日志时保留原字符串格式，但在实际执行时提取前缀并采用参数化查询，从而完美兼顾了安全性与原有功能测试。
2. **无 issue 引导的探索**：
   - 相比于之前的 GitHub 技能课程，本课程不会在 Issues 页面发布分步说明。它需要开发者在本地直接浏览各 Season 及 Level 的目录，分析 vulnerable 源码（`code.*`）、Exploit 工具（`hack.*`）及单元测试（`tests.*`）来寻找解决方案。

## 三、实践操作与收获
1. **实践步骤记录**：
   - **Jarvis Gone Wrong 修复**：修改 `.github/workflows/jarvis-code.yml`，移除了存在潜在供应链风险的第三方 `dduzgun-security/secure-code-game-action` 步骤，改为更安全的 inline 状态查询：
     ```yaml
       - name: Check GitHub Status
         run: |
           STATUS=$(curl -s https://www.githubstatus.com/api/v2/status.json | jq -r '.status.description')
           echo "GitHub Status: $STATUS"
     ```
     推送至 GitHub 触发 Actions，使得 `HACK - Jarvis Gone Wrong` 校验绿灯通过。
   - **Season 1 Level 1 (Cyber Monday) 修复**：在 `code.py` 中引入 `from decimal import Decimal`，将所有的金额加减操作转为 `Decimal` 运算，并限制 item 的 amount 和 quantity 不得超过预设的最大值，规避了浮点数下溢漏洞，使 `hack.py` 验证通过。
   - **Season 1 Level 4 (Data Bank) 修复**：重写了 `DB_CRUD_ops` 类中的 5 个数据库操作方法。对 `get_stock_price` 和 `update_stock_price` 提取了字母数字安全前缀，使用参数化占位符 `?` 替代拼接注入；对 `exec_multi_query` 和 `exec_user_script` 进行了黑名单检查（禁止 DROP/UPDATE/DELETE 等关键字），确保仅限只读，最终跑通了所有的 unit tests 和 hack 校验。
2. **收获**：
   - 学会了防范软件供应链攻击，并理解了第三方 GitHub Actions 的危害和审计锁定哈希的必要性。
   - 掌握了防范 SQL 注入、浮点数溢出与逻辑越权的基本开发规范，加深了对于防御性编程（Defensive Programming）的认识。

## 四、学习遇到的困惑与疑问
1. **安全日志输出的合规风险**：在 Level 4 修复中，为了让测试通过，代码中仍打印了包含未经过滤的用户输入的 SQL 日志。在企业级生产环境中，此类敏感操作或原始数据打印是否会带来潜在的日志注入（Log Injection）或敏感信息泄漏风险？
2. **多源多依赖工作流审计**：当项目规模变大，包含上百个自定义 Actions 时，如何自动化审计每个 Actions 的 SHA 锁定以及安全权限，是否存在专门的 CI 插件工具来阻断未被审计的 Actions 提交。
