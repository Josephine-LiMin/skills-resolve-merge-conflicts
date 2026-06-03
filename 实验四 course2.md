# 学习报告：实验四 course2 (Test with GitHub Actions)

本项目是 GitHub Skills 课程中的 **Test with GitHub Actions**（在我们的沟通中命名为 **实验四 course2**），旨在帮助开发者理解和应用持续集成（Continuous Integration, CI）的最佳实践，包括自动化测试、测试覆盖率分析、分支保护规则（Rulesets）以及如何通过 CI 保护主分支代码质量。

## 一、核心知识点梳理

1. **持续集成（CI）的核心价值**
   - 通过在代码提交或拉取请求（PR）时自动触发运行测试集，CI 能够提供快速的代码质量反馈，防止错误被带入 `main` 分支。

2. **自动化测试与覆盖率监控（Pytest & Coverage）**
   - 学习了如何在本地及 Actions 环境中使用 `pytest` 运行单元测试。
   - 使用 `coverage` 工具对代码测试覆盖率进行审计。覆盖率代表了测试代码执行了源程序中多少比例的行数，是衡量测试完备度的重要指标。

3. **第三方 Action 与市场（Marketplace）的使用**
   - 引入了社区优秀的预制 Action：
     - `actions/setup-python`：快速在 Runner 虚拟机中搭建指定版本的 Python 运行环境。
     - `py-cov-action/python-coverage-comment-action`：自动运行单元测试并在 PR 的讨论区中发表详细的测试覆盖率报告评论。

4. **分支规则集（Branch Rulesets）与状态检查拦截**
   - 通过在仓库级别配置 Branch Ruleset（分支规则集），对 `main` 分支开启保护。
   - 规定必须通过特定的状态检查（例如 `python-coverage` 工作流运行成功）才允许执行 PR 合并，从而形成开发红线，杜绝未通过测试或覆盖率不达标的代码并入生产环境。

---

## 二、学习中的难点

1. **文档路径拼写不一致问题**
   - 在课程的第三步指南中，多处将测试文件拼写为 `tests/calculation_tests.py`，然而仓库中实际存在的文件名是 `tests/calculations_test.py`。
   - **解决方案：** 在进行代码操作前，主动通过 `dir` 列出 `tests` 目录的内容发现这一细节，从而避免了因路径引用错误导致的代码替换失败。

2. **Push 冲突与非快进式（Non-fast-forward）拒绝**
   - 在我们本地配置好工作流并尝试推送到远程 `main` 分支时，Git 提示推送失败。这是由于当第一步的校验逻辑启动时，GitHub 校验工作流自动修改了远程的 `README.md` 与步骤文档，导致本地与远程基线不一致。
   - **解决方案：** 优先执行 `git pull` 命令，在本地通过 Git 的自动三方合并策略合并远程的文档变更后，再重新将工作流配置文件推送至远程仓库。

3. **故意设置的单元测试失败与逻辑修复**
   - 课程在 `tests/calculations_test.py` 中故意包含了一个错误的测试断言——声称斐波那契数列第 10 项为 89（`assert result == 89`），导致测试流水线大面积飘红。
   - **解决方案：** 经过数学验证与代码分析，斐波那契数列第 10 项的正确数值应为 55。修改断言为 `assert result == 55` 后成功使测试运行通过。

4. **测试覆盖率不足 90% 的门槛限制**
   - 测试通过后，由于 `src/calculations.py` 内部对于负数半径及负数 n 抛出 `ValueError` 的边界逻辑没有被现有的测试用例覆盖，导致整体覆盖率低于课程要求的 90% 阈值，流水线依然被判定为失败。
   - **解决方案：** 深入分析源程序的分支逻辑，在测试文件中编写并补齐了 `test_area_of_circle_negative_radius` 和 `test_get_nth_fibonacci_negative` 两个抛出异常的专用单元测试，将覆盖率提升至 100% 并最终解锁了 PR 合并按钮。

---

## 三、实践操作与收获

1. **完整操作链路**
   - **步骤 1**：在 Issue 评论区发表指定关键字评论以唤醒工作流。
   - **步骤 2**：在本地创建两个关键工作流：
     - `.github/workflows/python-package.yml`：针对多版本 Python 执行 pytest 自动编译测试。
     - `.github/workflows/python-coverage.yml`：执行代码覆盖率生成，并通过 `coverage report --fail-under=90` 与 `py-cov-action` 对低于 90% 覆盖率的 PR 进行拦截和评论。
   - **步骤 3**：新建 `reenable-unit-test` 分支，开启此前被注释掉的第 10 项斐波那契测试，提交并创建 PR。
   - **步骤 4**：调用 GitHub API 自动为仓库配置了分支规则集 `Protect main`，强制要求 `python-coverage` 状态检查必须通过。修复了错误的断言并将缺失的负数异常捕获测试补充完整，成功把覆盖率拉满至 100%。测试全部通过后，成功将 PR 合并入 `main`。

2. **核心收获**
   - 完整走过了一次“代码开发 -> 自动测试失败反馈 -> 本地修复 -> 覆盖率达标 -> 突破分支保护规则合并”的闭环 CI 研发流程。
   - 熟悉了 GitHub 规则集（Ruleset）的 API 编写与配置，实现了分支自动拦截保护。
   - 掌握了在 Python CI 中使用 pytest 进行覆盖率分析、以及断言异常（`pytest.raises(ValueError)`）的测试编写方法。

---

## 四、学习遇到的困惑与疑问

1. **规则集中 required_status_checks 的更新同步**
   - 如果一个工作流的 Job 名字或者工作流文件名发生了变更，规则集是否能够自适应感知？
   - *思考/解决方向：* 规则集匹配是基于字面量状态上下文（Status Context）名称进行的。如果 YAML 文件中的 `jobs.job_id.name` 发生改变，规则集将由于找不到原本名字的状态而导致 PR 永久处于“等待状态检查中”的挂起状态。因此，重构工作流名字时，必须同步使用 GitHub API 或后台界面更新对应的分支保护规则定义。

2. **在单次 PR 中防止覆盖率计算被恶意篡改**
   - 开发者可以通过直接调小 `python-coverage.yml` 中的 `--fail-under` 参数来绕过覆盖率门槛。如何防止这种“篡改规则”的提交被并入主分支？
   - *思考/解决方向：* 应该在规则集中加入对工作流配置文件目录 `.github/workflows/` 的文件保护。例如，可以使用 CODEOWNERS 机制，规定任何对该目录下 YAML 文件的修改必须经过运维或架构师团队的特别审批。
