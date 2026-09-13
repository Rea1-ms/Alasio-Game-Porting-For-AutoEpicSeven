---
name: alasio-game-porting-for-autoepicseven
description: >
  Load when the user asks to migrate, develop, review, debug, package, or update AutoEpicSeven/E7 on Alasio, including the config/worker bridge, dependencies and Python upgrades, frontend/webapp integration, Windows portable releases, or ALAS tasks, state loops, pages, assets, OCR, and emulator tests. Also load for explicit $alasio-game-porting-for-autoepicseven requests. Do not use for Epic Seven gameplay advice, generic Python/Electron work, unrelated automation projects, or other ALAS games.
---

# AutoEpicSeven Alasio Game Porting

Version: 0.2 (2026-09-13)

把 AutoEpicSeven 当作已经成熟的 E7 自动化项目，在保留游戏能力的前提下迁移到 Alasio。先读当前仓库、Alasio 依赖版本、git 差异和本 skill 的对应参考资料。

## Always Read

- `references/autoepicseven-rules.md`：仓库硬约束、资产、OCR、页面、分服、测试与调度规则。
- 本文件的任务路由和状态循环底线。

## Task Routing

| 当前工作 | 还要读取 |
| --- | --- |
| 新增或重写任务、交互流程 | `references/asset-task-conventions.md`、`references/task-checklist.md`、`references/state-loop-bible.md`、`references/development-lessons.md` |
| 审核状态循环、死循环、点击时序 | `references/state-loop-bible.md`、`references/development-lessons.md` |
| 新增页面、修改路由、按服务器注册页面 | `references/wiki-guidance.md` 的 Page/UI 与多服务器章节、`references/autoepicseven-rules.md` 的页面与分服章节 |
| 新增或调整 assets、命名、目录或颜色/模板识别 | `references/asset-task-conventions.md`、`references/wiki-guidance.md` 的识别章节、`references/development-lessons.md` 的识别经验 |
| 定位任务入口、实现文件或 asset 来源 | `references/asset-task-conventions.md` 的入口检索与反向定位章节 |
| OCR、资源栏、语言问题 | `references/autoepicseven-rules.md` 的 OCR 章节、`references/development-lessons.md` 的 OCR 经验 |
| 新增任务配置、GUI、调度入口 | `references/task-checklist.md` 的配置章节、`references/wiki-guidance.md` 的配置章节 |
| 写测试、复现实机日志问题 | `references/autoepicseven-rules.md` 的测试章节、`references/development-lessons.md` |
| Alasio 架构、配置桥、worker、上游同步 | `references/alasio-migration-workflow.md`、`references/alasio-migration-gotchas.md` |
| Python 或依赖升级、OCR 模型评估 | `references/alasio-migration-workflow.md` 的依赖阶段、`references/alasio-migration-gotchas.md` 的运行时与 OCR 章节 |
| frontend、webapp、桌面端界面或打包 | `references/alasio-migration-workflow.md` 的前端与发行阶段、`references/alasio-migration-gotchas.md` 的桌面端章节 |
| Windows 发行组装、更新或验收 | `references/alasio-migration-workflow.md`、`references/alasio-migration-gotchas.md` 的发行章节 |
| 维护或验收本 skill | `references/skill-evals.md` |
| 其他 AutoEpicSeven 开发 | 扫描所有参考文件标题，只读取与当前任务直接相关的章节 |

## Execution Defaults

1. 确认当前目录、worktree、git common dir、分支和 remote。业务代码只改当前 worktree；不要把原版 master 当作迁移分支。
2. 先读当前实现、Alasio 锁定提交和 git 差异，再界定迁移范围。默认保持混合桥接：保留 `tasks/`、设备层、OCR、UI 寻路和 assets 管线，只迁移已有替代能力的框架部分。
3. 依赖变更使用 uv 命令生成或更新锁文件，不手改 `uv.lock`；uv 只管理依赖，不把它提供的 Python 当作发行运行时。
4. 游戏新功能仍在当前 worktree 的 `tasks/<task>/` 下建立同名实现；测试只放原始文件夹下被 git 忽略的 `test/`。
5. 实现交互时使用截图优先的单层状态循环；顺序固定为退出状态、点击或动作状态、通用处理器。
6. 资产缺失就停止并提醒用户放置源图。不要用 `try/except` 隐藏缺省资产，不修改生成的 assets wrapper。
7. 配置只改源 YAML 和允许手工维护的 i18n；运行任何 config updater 前必须请求用户批准，不手改生成索引和模型。
8. 每完成一个可独立验证的迁移批次就执行定向检查；依赖、配置、桌面端和发行更新分别验证，不用单一冒烟结果代替全链路。

## State Loop Baseline

- 禁止用 `click + sleep` 推进游戏流程。
- 循环首部必须获取新截图，并用 `skip_first_screenshot` 复用调用方已有截图。
- 独立测试或直接运行前显式执行一次截图。
- 退出只使用正面、无副作用的状态判断；不要用负面条件、点击语句或 handler 作为退出判据。
- 点击动作通常设置 2/3/5 秒 interval；纯退出识别不设置 interval。
- `handle_*` 只返回 bool；仅当确实执行了会改变画面的操作时返回 True。
- 可预期的短子状态使用局部 Timer 或重试上限；长动画和战斗主循环不强加短超时。
- 默认保持一个顶层状态循环。确需嵌套时，子循环必须退出到父循环能够识别的状态，并写清原因。
- 反复修复、容易回归的状态变量和时序判断必须写详尽英文注释，说明不变量及失败模式。

完整原文与反例见 `references/state-loop-bible.md`。

## Known Gotchas

- assets 语言与 OCR 模型语言是两套系统；`global_cn` 不能传给 OCR。
  -> `references/autoepicseven-rules.md#ocr-语言`
- 固定位置只判亮灰优先 `match_color()`；先定位再判色使用 `match_template_color()`。
  -> `references/autoepicseven-rules.md#颜色与模板匹配`
- 稀有中间状态不能用于业务计数。免费召唤应由“继续免费召唤/返回/已用完”等游戏状态推进。
  -> `references/development-lessons.md#召唤与活动流程`
- E7 的计数器当前值可以合法超过恢复上限，不要套用 `value <= total`。
  -> `references/development-lessons.md#ocr-与资源采集`
- 跨模块页面跳转优先 `ui_goto(Page)`，不要复用依赖隐含页面上下文的私有入口方法。
  -> `references/development-lessons.md#页面与跨模块跳转`
- 修改函数名、变量名或类名时，不执行全仓批量替换；汇总重命名映射并交给用户用 IDE 处理。
  -> `references/autoepicseven-rules.md#编辑边界`
- `ModuleNotFoundError: alasio` 不应长期用 `PYTHONPATH` 掩盖；正式预期是 uv 锁定并安装完整的 Alasio 包。
  -> `references/alasio-migration-gotchas.md#alasio-导入与安装`
- 发行运行时当前固定为完整 64 位便携 CPython 3.10.x；Python 3.14 只在隔离目录验证。
  -> `references/alasio-migration-gotchas.md#python-与依赖`
- 桌面端白屏、`Not Found` 和后端提前退出是三类问题，按前端产物、CSP 和启动入口分别检查。
  -> `references/alasio-migration-gotchas.md#桌面端与前端`
- 实际配置不进入发行包；更新必须保留 `deploy.yaml`、`aes.db` 和 `gui.db` 的内容与权限。
  -> `references/alasio-migration-gotchas.md#发行与更新`

## Validation Loop

1. 运行针对改动文件的 ruff 检查；有错误就先修第一个真实问题，再重跑。
2. 运行 `git diff --check`，确认 LF、空白和补丁格式正常。
3. 运行目标测试；测试需要权限时直接请求批准，不尝试 uv 或 pytest。
4. 在 linked worktree 中执行 `git add` 或 `git commit` 前直接请求权限批准；git common dir 可能位于沙箱外，否则会因无法创建 `index.lock` 失败。
5. 执行 `deploy/Windows/build_release.ps1` 前直接请求权限批准；脚本可能需要读取 WinGet 的 `uv.exe` 链接等沙箱外资源。
6. 页面或分服注册有改动时，至少检查目标服务器和非目标服务器的导入、页面扫描与寻路行为。
7. 最多连续修复三轮同类失败；仍失败时保留日志和截图，说明阻塞状态，不用宽泛 fallback 掩盖错误。

## Completion Checklist

- [ ] 当前 worktree、服务器范围、页面进出和成功条件已确认
- [ ] 没有删除文件，没有手改生成文件，没有隐藏缺失 assets
- [ ] 状态循环没有固定 sleep、负面退出或带副作用退出
- [ ] OCR 显式使用规范化语言，assets 仍由 server.lang 选择
- [ ] 页面节点、路由、返回关系和入场后弹窗状态已覆盖
- [ ] 配置源、调度顺序、aes.py 入口和 i18n 按任务需要完成
- [ ] 对应测试已放入原始文件夹下被忽略的 `test/`，并完成可执行验证
- [ ] 日志足以解释失败位置，脆弱状态有详尽英文注释
- [ ] Alasio 依赖由 uv 正常安装，不依赖临时 `PYTHONPATH`
- [ ] 发行包包含实际 frontend、webapp 和完整便携 Python，且不包含用户配置
- [ ] 更新验证确认端口、密码、任务配置和三份受保护文件保持不变
