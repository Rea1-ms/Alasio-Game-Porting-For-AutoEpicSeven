# Alasio 迁移踩坑记录

只记录已经在 AutoEpicSeven → Alasio 迁移中复现、验证或明确保留的问题。时间敏感的依赖结论必须重新检查，不能当作永久事实。

## Alasio 导入与安装

| 现象 | 已确认原因 | 处理方式 |
| --- | --- | --- |
| `ModuleNotFoundError: No module named 'alasio'` | 当前虚拟环境没有安装 Alasio；`uv sync --frozen` 只验证既有锁文件，不会替错误或过期的锁自动选择新来源 | 检查 `pyproject.toml` 的 uv source 和 `uv.lock` 中的提交，使用 uv 更新锁并同步，再在清空 `PYTHONPATH` 后验证导入 |
| 临时设置 `PYTHONPATH=<Alasio>` 后可以启动 | 只能证明源码目录可导入，不能证明 wheel 包含命名空间、配置资源和 backend extra | 仅用于诊断；正式开发和发行必须安装锁定依赖 |
| 早期 wheel 安装后仍缺模块或资源 | Alasio 当时的打包发现规则没有完整包含 namespace package 和 YAML/JSON 资源 | 使用已经修复 package discovery 的 Alasio 提交，并在发行目录做无 `PYTHONPATH` 导入测试 |

当前预期行为：AutoEpicSeven 把 Alasio 声明为 uv 管理的 Git 依赖，`uv sync` 后直接运行本仓库 `gui.py`；不要求用户手工设置 `PYTHONPATH`。

## Python 与依赖

### uv 不是发行 Python

uv 负责解析、锁定和安装依赖，不等于完整的可分发 Python。曾尝试把 uv 提供的不完整 Python 当作运行时，最终改回完整 64 位便携 CPython 3.10.x。发行组装会检查 `bz2`、`ctypes`、`ensurepip`、`lzma`、`multiprocessing`、`sqlite3`、`ssl` 和 `venv`。

### Python 3.14 隔离验证结论

2026-09-12 的验证中：

- `onnxruntime==1.23.2` 没有可用的 Windows cp314 wheel，是完整安装的硬阻塞。
- `jellyfish==0.11.2` 需要源码构建，是次要阻塞，不能仅凭版本号判断运行兼容。
- Python 语法和类型注解可逐批现代化，但不等于依赖栈已经能切换到 3.14。

这些 wheel 状态会变化。再次评估时必须在新的独立目录重新解析依赖，不改变现有 3.10 环境；只有安装、离线 OCR、设备连接和真实任务都通过后才能讨论切换发行基线。

### 已处理的旧依赖

- SciPy 在项目中没有直接使用，确认后从直接依赖移除，比无目的升级更稳妥。
- `adbutils` 从 1.2.9 升到 2.12.0 时，私有 `_connect()`、全局 `adb_path` patch 等调用改为公开的 reverse、forward removal、device listing、open shell 和 `ADBUTILS_ADB_PATH`。
- `uiautomator2` 从 2.16.17 升到 3.7.0 时，私有 init、HTTP endpoint 和 atx-agent URL 调用改为公开的 connect、window size、shell 与 reset API。旧远程 HTTP 模式所需的小范围能力隔离到项目自己的兼容 client，不再绑住 uiautomator2 私有接口。

升级这两类库时先搜索私有属性和私有 endpoint，逐项替换并做模拟器测试；只通过 import 不足以证明反向端口、截图、点击、应用启动和重启服务正常。

## OCR 模型与十连识别

新 OCR 模型不自动等于十连结果更准。十连结果识别先分离三类变量：

1. 截图分辨率、窗口缩放和框选坐标是否一致。
2. 十个槽位的几何切分是否落在姓名文本区。
3. 模型识别、语言映射和后处理是否正确。

评估新模型时使用同一批真实十连截图建立基线，分别统计槽位定位、姓名识别和整体十连全对率。先修框选偏移，再比较模型；新模型只有在准确率、速度、包体和 Python wheel 均满足发行要求时才迁移。不要用一张截图或普通资源栏 OCR 结果替代十连专项验证。

## 配置模型与数据迁移

| 问题 | 处理结论 |
| --- | --- |
| 新模型字段名与旧扁平字段不一致 | adapter 双向映射 Game、Device 和 EmulatorInfo 字段，改名时同步映射并跑往返测试 |
| 新 datetime 要求带时区，旧代码使用本地 naive datetime | 只在 adapter 边界双向转换，游戏代码继续保持原语义 |
| stored/Dashboard 在新模型中是文本 | 写入时编码 JSON，读取时恢复 dict；新增同类字段要登记 |
| 新模型没有 `Scheduler.Command` | 读取时按任务名合成，写入时忽略 |
| 框架任务混入旧调度数据 | 排除 Device、RestartDevice、RestartGame 和 `_global_bind` |
| `save()` 后修改未持久化 | `_persist()` 必须发生在 `modified.clear()` 之前 |
| 严格 Literal 拒绝旧脏值 | 记录原值和拒绝原因，显式清洗；不要静默改成默认值 |
| 旧 `base:` 仍能被解析却丢失继承 | 新 args YAML 使用 `parent:`；生成后检查模型字段，而不只看命令退出码 |
| tasks 的 groups 结构变化 | 使用当前上游要求的映射结构，不照搬旧列表语法 |

上游已经自行修复过两项早期问题：批量部分写入清空同组兄弟字段，以及 `Literal[True]` 与 msgspec 不兼容。同步上游后先验证现状，不继续携带重复补丁。AutoEpicSeven 任务使用当前支持的 SchedulerUedit 变体。

用户数据方面只迁移用户指定的当前配置。旧 JSON 与新 SQLite 一旦同时运行会分叉；迁移后不能继续用旧入口修改同一配置。`config/alas.json` 曾被确认是其他项目残留，不能当作 E7 当前配置。

## 桌面端与前端

### 三种相似但不同的失败

| 表现 | 根因 | 检查 |
| --- | --- | --- |
| 主窗口白屏 | renderer 构建后没有更新 CSP inline hash，启动脚本被拦截 | webapp 使用 `pnpm package`；不要跳过 CSP 修正直接运行 electron-builder |
| 页面显示 `Not Found` | 后端找不到 `toolkit/Lib/site-packages/frontend/build` | 先构建独立 frontend，并由发行脚本复制完整 `build` |
| `Backend exited before ready (code: 0)` | 常见于先手工启动 `toolkit/python.exe gui.py`，再启动桌面端；端口、认证和生命周期由两个入口争用 | 关闭残留进程，只从 `toolkit/WebApp/Alasio.exe` 启动 |

`PLUGIN_TIMINGS` 是构建耗时分析，不是失败；只要构建最终成功并更新 `app.asar`，无需因为该段日志重跑。

### pnpm 12 构建脚本许可

pnpm 12 默认不运行未允许的依赖构建脚本。允许项保存在 `pnpm-workspace.yaml` 的 `allowBuilds`。使用 `pnpm ignored-builds` 查看被拦截项，确认用途后只批准具体包；不要使用批准全部的方式，也不要手改 lockfile 规避。

### 已解决但需要回归的桌面端问题

桌面端透明拖拽层曾覆盖整个顶部约 48 像素区域，造成语言、主题和左侧配置卡上半部分无法点击。
2026-09-13 已把拖拽范围缩到中部空白区，并完成桌面端点击回归。后续改动仍要同时验证关键控件
可以点击、中部区域可以拖动；不要只以“界面能显示”判定 GUI 验收通过。

独立把 `Alasio.exe` 拿离完整工程目录运行不属于受支持发行形态。若工程根、配置、Python 或前端缺失，长期应显示明确错误页；黑屏不是可接受提示，但该体验修复优先级低于完整发行链。

## 发行与更新

### 发行包组成

完整发行目录必须同时包含：

- 当前已提交的 AutoEpicSeven 文件。
- 完整 64 位便携 CPython 3.10.x。
- uv 导出的锁定依赖和可执行文件。
- Alasio `frontend/build`。
- Alasio `webapp/release/win-unpacked`。
- 可验证上述内容的 `release-manifest.json`。

只复制开发仓库、只复制 webapp，或让桌面端脱离工程目录单独运行，都不是完整发行测试。

### 用户配置不打包但更新时保留

实际 `config/deploy.yaml`、`config/aes.db` 和 `config/gui.db` 不进入发行包，否则会泄露密码或覆盖用户状态。发行包只带模板。更新脚本在复制前后核对这三份文件的内容 hash、Windows ACL 和属性。

如果测试安装目录本来没有 `deploy.yaml`，首次启动会从模板生成新配置，因此端口和密码会变化。这只能验证首次安装，不能验证“更新保留配置”。更新测试必须从已有真实发行安装复制，并确认三份文件原本存在。

### 发行组装权限

`build_release.ps1` 需要复制系统可见的 uv；通过 WinGet link 读取 `C:/Users/<user>/AppData/Local/Microsoft/WinGet/Links/uv.exe` 时曾出现 `Access denied`。因此运行发行组装前直接请求用户批准，不先无权限试跑。

同名发行输出存在时脚本会停止，不覆盖也不清理。权限失败后的半成品使用新 `ReleaseName` 重试；受“不能删除文件”约束时，只记录待用户清理的旧产物。

### linked worktree 提交权限

当前迁移目录是 linked worktree 时，`.git` 指向原始仓库的 git common dir。沙箱可能能写 worktree，却不能在外部 common dir 创建 `index.lock`，表现为：

```text
fatal: Unable to create '<common-dir>/worktrees/<name>/index.lock': Permission denied
```

执行 `git add` 或 `git commit` 前直接请求用户批准。不要先重试、不要改 Git 目录，也不要把失败误判为已有锁文件冲突。提交前仍需检查秘密信息，并且批准不包含 push。

## 诊断顺序

遇到启动或发行问题时固定按以下顺序缩小范围：

1. 安装来源与无 `PYTHONPATH` 导入。
2. `deploy.yaml`、端口、backend 密码和活动数据库。
3. 独立 frontend 产物。
4. webapp CSP 和 `app.asar`。
5. 完整发行清单与目录结构。
6. 唯一桌面端入口和残留进程。
7. GUI 交互区域。
8. 模拟器真实任务。

不要用修改端口、删除配置、复制随机 build 目录或长期设置 `PYTHONPATH` 作为泛化修复。

## Anti-Drift 记录

| 曾出现的判断 | 日期 | 实际后果 | 以后怎么做 |
| --- | --- | --- | --- |
| “`uv sync --frozen` 成功就表示 Alasio 已安装” | 2026-09-09 | 旧锁文件检查通过，但运行仍报 `ModuleNotFoundError` | 同时检查依赖来源、锁定提交和无 `PYTHONPATH` 导入 |
| “webapp 目录包里已经包含全部界面” | 2026-09-13 | 桌面端启动后显示 `Not Found` | 独立构建并打包 `frontend/build` |
| “直接运行 electron-builder 就等于正式打包” | 2026-09-13 | CSP hash 没更新，桌面端白屏 | 使用 `pnpm package` 的完整顺序 |
| “端口变化只是测试环境随机变化” | 2026-09-13 | 实际是测试目录没有旧 `deploy.yaml`，新配置重新生成 | 更新验证必须从带真实配置的安装副本开始 |
| “界面显示出来就算 GUI 通过” | 2026-09-13 | 顶部与左侧部分区域被拖拽层覆盖，控件不能点击 | 把关键控件点击纳入桌面端回归 |
