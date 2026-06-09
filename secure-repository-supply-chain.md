## 一、核心知识点梳理

1. **依赖供应链安全 (Supply Chain Security)**：
   - 当前软件开发高度依赖开源第三方包。如果其中某个包存在安全漏洞，可能会让软件受到供应链攻击（Supply Chain Attack）。
   - GitHub 提供了依赖图（Dependency Graph）、依赖评审（Dependency Review）、Dependabot alerts 和 Dependabot updates 等全套机制，自动识别、警告并修补脆弱依赖。
2. **依赖图 (Dependency Graph)**：
   - 依赖图是对于存储在仓库中的 manifest 和 lock 文件（例如 `package-lock.json`）的依赖汇总。
   - 它展示了当前项目所有的直接依赖和间接（传递性）依赖。
3. **Dependabot 警报 (Dependabot Alerts)**：
   - 当项目依赖的包在 GitHub 安全公告数据库 (GitHub Advisory Database) 中被标记为包含已知漏洞或恶意软件时，GitHub 就会自动向项目维护者发送 Dependabot 警报。
4. **Dependabot 安全更新 (Dependabot Security Updates)**：
   - 启用后，当触发 Dependabot 警报且有可用补丁版本时，GitHub 将自动创建 Pull Request（PR）尝试将有漏洞的包升级到安全版本。
5. **Dependabot 版本更新 (Dependabot Version Updates)**：
   - 维持依赖包处于最新版本是防止安全问题的最佳实践。
   - 通过在仓库根目录中配置 `.github/dependabot.yml` 文件，能够自定义扫描周期（如每日、每周、每月）对指定的生态系统（如 `github-actions`, `nuget`, `npm` 等）进行版本更新检测，并由 Dependabot 自动提 PR。

## 二、学习中的难点

1. **手动与自动修复带来的 PR 冲突**：
   - 本次实验在手动将包升级至 1.2.8 后，Dependabot 为修复 Axios 漏洞所提的 PR 分支与主分支 `main` 发生了 `package-lock.json` 冲突（`CONFLICTING`）。
   - 难点在于如何在不破坏原本包依赖关系的前提下解决锁文件冲突。
   - **解决办法**：在本地检出 Dependabot PR 的分支，执行 `git merge origin/main`。针对 `package-lock.json` 冲突，我们使用 `git checkout --ours` 将基准恢复为本分支，然后在此基础上在项目目录运行 `npm install` 自动让 npm 工具重新分析依赖并产生合并后的新锁定文件。最后提交并推送回 PR 分支，使 GitHub 端冲突得以完美消解。
2. **REST API 与 GraphQL 的网络环境重试**：
   - 检查任务进度时经常会因为 GraphQL TLS 握手超时出错，可以通过换用 REST API 接口（例如 `GET /repos/{owner}/{repo}/issues/1/comments`）或增设合理的重试逻辑以越过偶然的连接错误。

## 三、实践操作与收获

1. **完成的四个主要步骤**：
   - **Step 1**：在项目 `package-lock.json` 里添加 `follow-redirects`（漏洞版本 1.14.1），成功建立依赖图数据。
   - **Step 2**：在安全设置中开启了 Dependabot alerts 并手动通过 npm 将 `minimist` 升级到安全的 `1.2.8` 版本推送解决警报。
   - **Step 3**：开启 Dependabot security updates，针对系统检测出的 Axios 高危漏洞 PR 进行冲突解决与合并。
   - **Step 4**：向 `.github/dependabot.yml` 添加了 `nuget` 生态监测配置，并推送触发扫描成功。
2. **实践收获**：
   - 理解了 Dependabot 的运行原理及其自动合并和废弃多余 PR 的智能清理策略（若升级某个包顺带连同其它包的问题也修复了，Dependabot 会自动关闭并删除废弃的 PR）。
   - 掌握了处理多人协作或多任务同时修改 package 文件导致的冲突处理模型。

## 四、学习遇到的困惑与疑问

1. **组合安全更新 PR 的需求**：
   - 如果一个历史遗留项目首次开启 Dependabot，会一次性生成数十个安全修复 PR，逐个审查并测试会耗费大量精力。GitHub 目前是否提供了更好的“组合安全 PR（Grouped Security Updates）”功能，使得在保证流水线不挂的前提下将关联依赖一次性更新？
2. **私有包管理器的支持**：
   - 很多大型企业在内网部署私有 NuGet 或 npm 镜像源，这种不公开的依赖包源应该如何在 `.github/dependabot.yml` 中配置凭证，以便 Dependabot 能够正确抓取版本更新？
