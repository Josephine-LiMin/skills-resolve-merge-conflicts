# publish-docker-images

## 一、核心知识点梳理

1. **GitHub Packages 与 GitHub Container Registry (GHCR)**：
   - GitHub Packages 是 GitHub 提供的软件包和容器镜像托管服务，支持 Docker、npm、NuGet 等多种包格式。
   - GitHub Container Registry (GHCR) 的域名为 `ghcr.io`。GHCR 的主要特点是包级别生命周期独立于项目仓库，且支持灵活的访问控制。

2. **GHCR 的身份验证与权限**：
   - 在 GitHub Actions 工作流中，可以使用内置的 `${{ secrets.GITHUB_TOKEN }}` 通过 `docker/login-action` 对 `ghcr.io` 进行身份认证。
   - 需要在工作流的 `permissions` 块中显式指定 `packages: write` 以允许向 Container Registry 推送镜像。

3. **专用的 Docker Actions 工具集**：
   - `docker/setup-qemu-action`：用于设置 QEMU 模拟器，支持跨架构（如 ARM64）构建镜像。
   - `docker/setup-buildx-action`：用于配置 Docker Buildx，启用先进的 BuildKit 功能，如多平台并行构建和构建缓存。
   - `docker/build-push-action`：集成化的镜像构建与推送 Action，支持声明构建平台（`platforms`）和通过 `tags` 指定目标镜像标签。

4. **利用 `docker/metadata-action` 进行动态标签提取**：
   - 能够基于 Git 上下文自动计算生成 Docker 标签。支持对分支（如 `main`）、Pull Request（生成 `pr-X` 临时测试标签）以及 Release 标签（如发布 `v1.0.0` 和 `latest`）生成规范的镜像版本标识。

## 二、学习中的难点

1. **多平台镜像构建的编译耗时**：
   - 声明 `platforms: linux/amd64,linux/arm64` 时，Buildx 会在后台利用 QEMU 虚拟机对不同 CPU 架构进行交叉编译，这会显著延长构建执行时间。
2. **Docker 镜像命名规范限制**：
   - GHCR 规范要求镜像包名及标签必须全部小写。如果在路径中包含任何大写字母（如用户的 GitHub 账号含有大写），Docker 客户端构建或推送时会直接抛出非法命名格式的异常，因此在提取元数据和指定 tag 时必须注意格式转化。

## 三、实践操作与收获

1. **操作流程**：
   - 创建并克隆 `skills-publish-docker-images` 仓库，并在设置中开启动作的写权限。
   - 编写初版 `.github/workflows/docker-publish.yml`，使用传统的 `docker build` & `docker push` 命令，验证了向 `ghcr.io` 的登录和基本发布流程。
   - 本地拉取已发布的 Docker 镜像 `ghcr.io/josephine-limin/...` 并启动容器，在本地 `8080` 端口试玩了 **Stackoverflown** 拼图游戏。
   - 升级工作流，引入 `setup-qemu`、`setup-buildx` 和 `build-push-action`，实现多架构（`amd64`, `arm64`）自动打包推送。
   - 引入 `docker/metadata-action@v5` 动态提取 Git 元数据，同时更新了 `on` 触发器，支持 push、pull_request 和 release 等多生命周期事件。
   - 新建并切换到 `feature/add-high-score` 分支，修改游戏首页以支持高分榜展示。
   - 提交分支代码并创建 PR，触发了 PR 构建并自动打上 `pr-2` 标签。
   - 合并 PR 到 `main` 分支，随后创建并发布 `v1.0.0` 正式 Release，成功触发发布工作流，将正式版镜像 `v1.0.0` 推送到了 GHCR。

2. **遇到的困难及解决方法（反复做的地方）**：
   - **分支合并及标签冲突**：由于在测试多个生命周期触发时，频繁修改了分支配置，合并 PR 以及发布 Release 在 GitHub 端由系统 bot 触发了一些同步行为，导致本地在合并和提交阶段多次需要执行 `git pull --rebase` 来规避推送冲突。
   - **镜像大写字符转换**：如果将 tag 误写成含有大写 `Josephine-LiMin`，工作流会遭遇校验报错。通过在镜像名中使用小写形式 `ghcr.io/josephine-limin/...` 并结合 `docker/metadata-action` 对分支名的规范清洗，成功完成了推送。

## 四、学习遇到的困惑与疑问

1. **构建缓存的优化与复用**：
   - 随着项目的增长，交叉编译（跨架构编译）的耗时会急剧增加。在实际开发中，如何配合 `actions/cache` 或者使用 GHCR 自身的 `cache-to=gha` 和 `cache-from=gha` 缓存层，来缩短后续流水线的重复构建时间？
2. **多账号/组织级的包发布**：
   - GitHub Packages 的包是挂载在用户账号下而非单纯挂载在单个 Repo 下。在大型团队开发中，如何通过配置共享组织级的 packages 权限，以使得其他不隶属于该代码仓但具有包访问权的项目可以安全地使用这些镜像？
