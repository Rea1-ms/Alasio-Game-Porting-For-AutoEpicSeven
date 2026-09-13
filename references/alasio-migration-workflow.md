# AutoEpicSeven → Alasio 标准迁移流程

本流程整理自 2026 年 7 月至 9 月已经完成的配置桥、worker、依赖、桌面端和 Windows 发行迁移。它规定默认顺序和验收边界；具体异常见 `alasio-migration-gotchas.md`。

## 1. 建立基线

开始修改前依次确认：

1. 当前目录、仓库根目录、worktree、git common dir、当前分支和工作区差异。
2. AutoEpicSeven 与 Alasio 两个仓库的 remote、目标分支和锁定提交。上游信息在 Alasio 仓库内检查，不根据两个目录名猜测。
3. `pyproject.toml`、`uv.lock` 和 `[tool.uv.sources]` 中 Alasio 的真实来源。
4. 当前可运行入口、活动配置名、Python 版本，以及已有 `deploy.yaml`、`aes.db`、`gui.db` 是否存在。

原版 `master` 保持上游基线，迁移代码只进入迁移 worktree。发现用户改动、未跟踪配置数据库或 IDE 文件时先保留，不擅自还原、删除或提交。

## 2. 划定迁移边界

默认采用混合桥接：

- 保留 `tasks/`、`module/device`、OCR、页面寻路、assets 和状态循环。
- 迁移配置模型、持久化、调度入口、worker 生命周期、GUI、依赖与发行流程。
- 只有 Alasio 已经提供等价能力并通过真实任务验证时，才移除旧实现。

每批工作先写明输入、输出、兼容边界和验收方式，再归入 P0、P1、P2 或 P10。P0 优先保证能安装、能启动、能保存配置、能执行任务和能安全更新；界面易用性与低频工具按实际影响排序。

## 3. 同步上游与依赖

1. 比较当前锁定的 Alasio 提交与目标上游，先确认上游是否已经解决本地补丁对应的问题。
2. 依赖声明只改 `pyproject.toml` 或依赖源文件。
3. 使用 uv 更新锁文件，例如 `uv lock --upgrade-package alasio`；不要手工改 `uv.lock`。
4. 锁文件确定后使用 `uv sync --frozen` 验证开发环境。
5. 检查 `import alasio`、后端入口、`adbutils`、`uiautomator2` 和 OCR 运行库的实际导入。

更新旧依赖时先列兼容表，至少包含：当前版本、目标版本、旧私有接口、替代方法、Python wheel 状态、代码改动、离线验证和实机验证。没有直接 import 的依赖先确认是否可删除，不为了追求版本号继续保留。

Python 大版本验证必须在新目录和独立环境中进行，不改变现有可运行环境。先验证能否解析并安装全部二进制依赖，再做导入、离线 OCR 和真实任务；任何阶段失败都不能切换正式发行基线。

## 4. 配置与 worker 迁移

配置迁移遵循以下顺序：

1. 在 `module/config_alasio/<nav>/` 修改 args、tasks 等 YAML 源文件。
2. 同步调度优先级和入口；运行生成器前直接请求用户批准。
3. 检查生成的 model、config、index 和 i18n 差异；生成物不手改，i18n 仅按项目约定维护翻译。
4. 通过 adapter 保持旧代码的扁平配置访问、`task_delay`、`task_call`、`multi_set` 和 stored 行为。
5. 用临时项目根和临时数据库验证读写往返、字段映射、时区、严格类型和调度。
6. 只迁移用户指定的当前配置，不扫描和批量迁移其他旧配置。

`deploy.yaml` 包含 backend 配置、端口和用户设置的密码，必须和活动数据库一起迁移。正式发行包不携带实际配置，只携带模板；更新已有安装时保护实际配置。

worker 桥至少验证：启动、日志转发、停止请求、任务边界读取配置、调度时间写回，以及一个真实任务的完整运行。一个任务通过只代表桥接主链可用，不代表所有识别流程已经回归。

## 5. 开发入口验证

正式开发预期是 AutoEpicSeven 环境通过 uv 安装锁定的 Alasio 包，然后在 AutoEpicSeven 根目录启动：

```powershell
./.venv/Scripts/python.exe gui.py
```

`PYTHONPATH` 只可用于定位“包未安装还是代码不可导入”，不能作为正式启动方式。若 `uv sync --frozen` 后仍无法导入，检查锁文件是否包含正确 Alasio 提交、wheel 是否包含命名空间与资源，以及当前命令是否使用了正确虚拟环境。

开发入口通过后验证：

- GUI 能加载当前配置，不生成意外的新端口或密码。
- 修改配置后数据库写入成功，重启后仍存在。
- 启动、停止和调度状态与旧入口一致。
- 模拟器连接、截图、点击和一个代表任务运行正常。

## 6. 构建 frontend 与 webapp

在 Alasio 的 `frontend` 目录执行：

```powershell
pnpm install --frozen-lockfile
pnpm check
pnpm build
```

产物必须包含 `frontend/build/index.html`、`frontend/build/_app` 和至少一个 JavaScript bundle。

在 Alasio 的 `webapp` 目录执行：

```powershell
pnpm install --frozen-lockfile
pnpm exec tsx --test main/*.test.ts
pnpm check
pnpm package
```

pnpm 12 使用 `pnpm-workspace.yaml` 中的 `allowBuilds`。依赖构建脚本被拦截时，先用 `pnpm ignored-builds` 核对，再只批准明确需要的包；不要批准全部。

桌面端必须使用 `pnpm package` 或保持相同顺序：先构建 renderer 并修正 CSP hash，再运行 electron-builder。只运行 `vite build` 或直接运行 electron-builder 都不能替代这条链。

## 7. 组装 Windows 发行目录

### 当前发行路线的性质

当前采用“完整发行目录 + 发行清单 + 覆盖程序文件”的更新方式，这是 AutoEpicSeven 迁移项目的
发行选择，不是 Alasio 上游已经完成的方案，也不是新版 Git 强制要求。

选择这条路线是因为一次发行需要组合两个仓库的源码、完整便携 Python、锁定依赖、独立 frontend
和 webapp，同时必须排除并保护用户的 `deploy.yaml`、`aes.db`、`gui.db`。发行清单记录两个仓库的
提交号，是为了让二进制产物可追溯；要求 Alasio 提交已经推送，也是“正式发行必须能重新取得源码”
的项目规则。Git 本身允许从未推送甚至未提交的工作区构建。

Git 的 `safe.directory` 只处理仓库所有者与运行账户不一致；linked worktree 的 `index.lock` 权限错误
来自当前沙箱无法写外部 git common dir。这两类限制都可以在 Git 更新路线中单独处理，不构成改用
完整发行包的原因。Git 3.0 计划收紧的 `safe.bareRepository` 也只针对隐式发现的 bare repository，
普通工作树不受影响。

Alasio 源码中的 `deploy_dev` 已有按 Git 提交生成版本历史、全量包和增量包的方向，但尚未形成可直接
用于本项目的完整分发链。以后上游更新器成熟时可以重新评估；不要把当前完整目录方案描述成唯一道路
或上游既定设计。

组装前检查：

- AutoEpicSeven 所有需要进入发行包的 tracked change 已提交。
- `frontend/build` 和 `webapp/release/win-unpacked` 完整。
- `PortablePythonPath` 指向完整 64 位 CPython 3.10.x，包含标准库和动态库，不含旧 `WebApp`。
- 实际 `deploy.yaml`、`aes.db`、`gui.db` 没有被跟踪或暂存。

随后直接请求用户批准，再运行：

```powershell
pwsh -File ./deploy/Windows/build_release.ps1 `
  -WebAppPath <Alasio>/webapp/release/win-unpacked `
  -FrontendPath <Alasio>/frontend/build `
  -PortablePythonPath <portable-python>/toolkit `
  -ReleaseName AutoEpicSeven-<commit>-validation-<n>
```

该脚本可能需要读取 WinGet 的 `uv.exe` 链接等沙箱外资源，因此不要先无权限试跑。失败产物不会自动删除；重试使用新的 `ReleaseName`，并提醒用户稍后清理旧目录。

成功标准：发行清单存在，Python、uv、锁文件、frontend、webapp 和 `app.asar` hash 完整；清空 `PYTHONPATH` 后仍能导入 `alasio`、`adbutils`、`uiautomator2` 和后端入口。

## 8. 更新已有安装并验收

测试目录应复制一个真实发行安装，而不是复制开发仓库。它应包含 `toolkit`、`toolkit/WebApp` 和现有用户配置；是否带 `.git` 不影响更新逻辑，但开发仓库副本不能证明发行更新可用。

关闭已有后端和桌面端后执行：

```powershell
pwsh -File ./deploy/Windows/update_release.ps1 `
  -InstallPath <existing-install-copy> `
  -PackagePath <new-release-directory>
```

更新脚本应先验证发行清单和 hash，再覆盖程序文件，并确认以下文件内容、ACL 和属性不变：

- `config/deploy.yaml`
- `config/aes.db`
- `config/gui.db`

更新完成后只运行：

```powershell
./toolkit/WebApp/Alasio.exe
```

不要先运行 `toolkit/python.exe gui.py`。最终人工检查端口、backend 密码、当前配置、语言和主题、启动与停止、日志显示，并在模拟器中完整运行一个代表任务。

## 9. 分批提交

每批只包含一个可描述、可回滚、已验证的主题。提交前检查 diff、未跟踪文件和秘密信息，不提交 `.env`、数据库、实际部署配置、构建产物或 IDE 文件。

当前 linked worktree 的 git common dir 可能位于沙箱外。执行 `git add` 或 `git commit` 前直接请求用户批准，不先触发 `index.lock` 权限错误。权限只覆盖本次明确的暂存与提交，不包含 push。

## 10. 完成判定

- [ ] 上游来源、分支和锁定提交明确
- [ ] 依赖由 uv 更新和冻结验证，锁文件未手改
- [ ] 配置读写、worker 启停和真实任务通过
- [ ] frontend 与 webapp 按正确顺序构建
- [ ] 发行目录在无 `PYTHONPATH` 情况下导入通过
- [ ] 更新没有改变三份用户配置的内容、权限和属性
- [ ] 桌面端从唯一入口启动，端口、密码和当前配置保持不变
- [ ] 已知未解决问题明确记录，没有用 fallback 隐藏
