# Introduction to Secret Scanning

## 一、核心知识点梳理
GitHub 的机密扫描（Secret Scanning）和推送保护（Push Protection）是保障仓库安全、防止敏感信息泄露的关键机制：
1. **机密扫描 (Secret Scanning)**：自动检测提交到 GitHub 仓库的代码中是否包含如 API 密钥、访问令牌、密码等高风险机密信息。对于公开仓库，此功能免费开启，并能检测大量合作伙伴（如 AWS、Google Cloud 等）的常见机密模式。
2. **安全警报管理**：一旦检测到泄漏的机密，GitHub 会生成安全警报。对已泄露的机密，应先在提供商端进行注销或轮换（Remediation），然后将警报标记为 `Revoked` 并提供解决说明作为审计日志留档。
3. **推送保护 (Push Protection)**：当开发者执行 `git push` 或通过 Web 端提交代码时，GitHub 会实时检查代码中的高置信度机密。如果检测到机密，推送将被拒绝。
4. **推送保护绕过 (Bypass Request)**：当判定被拦截的机密是误报、属于测试数据或准备后续修复时，作者可以通过专属的绕过链接，选择对应的合理理由（如 `used_in_tests`、`false_positive`、`will_fix_later`）申请绕过，之后方可成功推送。

## 二、学习中的难点
1. **命令行推送受阻与强制拦截**：在本地完成包含测试机密的修改并进行 `git commit` 后，执行 `git push` 会直接被 GitHub 的推送保护功能以 `GH013: Repository rule violations` 拒绝，普通手段无法直接推送该 Commit 到远程仓库。
2. **绕过操作的程序化解决**：在非 Web 交互环境下，如何绕过推送保护是一个难点。直接使用 API 禁用推送保护对于公开仓库并不起作用，必须通过特定的绕过凭证机制。

## 三、实践操作与收获
1. **实践步骤记录**：
   - 启用了仓库的机密扫描，在 `credentials.yml` 中故意提交了一个测试敏感 Token。
   - 查看生成的 Security Alerts，在注销 Token 后将警报状态置为 `Revoked` 关闭。
   - 在 Settings 的 Advanced Security 中开启了 **Secret Protection** 和 **Push Protection**。
   - 本地在 `credentials.yml` 中再次填入一个非激活状态的 GitHub Token 并提交。
   - 尝试 `git push` 被拒绝，报错并提供了绕过 URL 及其 Placeholder ID：`3Et7RbC9no4fB7STiqUFFf3cESN`。
   - 使用 GitHub REST API 创建推送绕过请求：
     ```bash
     gh api --method POST -H "Accept: application/vnd.github+json" /repos/Josephine-LiMin/skills-introduction-to-secret-scanning/secret-scanning/push-protection-bypasses -f reason="used_in_tests" -f placeholder_id="3Et7RbC9no4fB7STiqUFFf3cESN"
     ```
   - 绕过创建成功后，顺利完成了 `git push origin main`，触发 Step 3 自动校验，成功关闭了 Issue。
2. **收获**：
   - 掌握了 GitHub 机密扫描和警报的处理流程。
   - 学会了如何利用 GitHub API 来绕过命令行推送保护，深入理解了推送保护在拦截和人工确认两方面的平衡。

## 四、学习遇到的困惑与疑问
1. **绕过有效期的时限**：API 返回的绕过信息中包含 `expire_at`。需要关注在高频率的 CI/CD 场景中，该绕过凭证的有效期能持续多久，是否会频繁需要重新审批。
2. **组织级审查策略**：在企业组织级中，如何结合 `bypass-requests/secret-scanning` 端点建立多人的机密绕过审核流，以防开发人员滥用绕过权限。
