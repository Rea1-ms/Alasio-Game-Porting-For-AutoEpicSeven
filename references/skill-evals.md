# Skill Evaluation v0.2

本文件用于验证 skill 的触发边界、路由和关键规则是否生效。维护 skill 时先运行这些思维测试，再改 description 或正文。

## Should Trigger

1. “用 ALAS 状态机给 AutoEpicSeven 新增竞技场奖励领取任务，补页面、assets 和测试。”
   - 原因：E7 任务、状态循环、页面、assets、测试全部命中。
2. “审核一下 `run_free_gacha`，为什么日志只算一抽，实际点了三次继续免费召唤？”
   - 原因：E7 状态计数与召唤实机排错命中。
3. “按服务器给 E7 的限时活动页面分流，只支持国际服中文，并接入调度配置。”
   - 原因：页面条件注册、语言 assets、任务配置命中。
4. “把 AutoEpicSeven 的配置和 worker 迁到最新版 Alasio，并保持现有任务不变。”
   - 原因：配置桥、worker 生命周期和 Alasio 上游同步命中。
5. “Python 3.14 能不能用于新版 OCR？请新建隔离环境验证十连识别。”
   - 原因：AutoEpicSeven 的 Python 迁移、OCR 模型和专项验证命中。
6. “组装 Windows 便携发行包，并确认更新不会覆盖密码和任务配置。”
   - 原因：Alasio frontend、webapp、便携 Python 和更新验收命中。

## Should Not Trigger

1. “Epic Seven 新手应该先练哪个角色？”
   - 原因：游戏攻略，不是自动化开发。
2. “给普通 Python 项目写一个调用在线 OCR 的命令行工具。”
   - 原因：与 AutoEpicSeven 和 ALAS 框架无关。
3. “给 Vite 页面加一个语言切换菜单。”
   - 原因：前端开发，与本 skill 无关。
4. “给普通 Electron 应用修复 CSP 白屏。”
   - 原因：没有 AutoEpicSeven 或 Alasio 迁移上下文。
5. “把另一个 ALAS 游戏项目升级到 Python 3.14。”
   - 原因：不是 AutoEpicSeven。
6. “解释 uv 和 pip 的区别。”
   - 原因：通用依赖管理问题，不涉及本项目迁移。

## Real Prompt Routes

### Prompt A: New Task

“新增一个 E7 每日签到任务，从主界面进入签到页，领取后回主界面。”

Expected reads:

- `autoepicseven-rules.md`
- `asset-task-conventions.md`
- `task-checklist.md`
- `state-loop-bible.md`
- `development-lessons.md`
- `wiki-guidance.md` 的 Page/UI 章节

Expected behavior:

- 先确认页面五类信息与 assets 是否存在。
- 在任务目录新增同名实现和 test。
- 状态循环以正面结束状态退出，不写 sleep。
- 接入 YAML、`config_manual.py` 和 `aes.py`；运行 updater 前请求批准。

### Prompt B: OCR Bug

“资源栏把 `19521/336` 改成了 `1952/336`，帮我查原因并修。”

Expected reads:

- `autoepicseven-rules.md` 的 OCR 章节
- `development-lessons.md` 的 OCR 与资源采集章节

Expected behavior:

- 不加入 `value <= total`。
- 检查前导位截断和按字符切分 fallback。
- 保留原始 OCR 文本，使用图标锚定布局验证。
- 加离线截图测试覆盖超恢复上限数值。

### Prompt C: Server-only Page

“这个活动页只有国际服中文，为什么国服模块 import 就崩了？”

Expected reads:

- `autoepicseven-rules.md` 的 Assets、页面、分服章节
- `wiki-guidance.md` 的多服务器和 Page/UI 章节

Expected behavior:

- 在 import 前完成服务器/语言分发。
- 条件注册 Page 和路由边。
- 不给国服创建虚假占位页面或捕获缺失 assets。
- 非支持组合正常跳过并调度到下一周期。

### Prompt D: Free Summon Count

“每天送五连，但用户可能先抽过；`SUMMON_SKIP` 和新角色提示也不一定出现，怎么计数？”

Expected reads:

- `development-lessons.md` 的召唤与活动流程
- `state-loop-bible.md`

Expected behavior:

- 不维护本地固定五次上限或动画计数。
- 出现继续免费召唤就继续，只剩返回就结束。
- 保存结果使用当前结果页标记，继续按钮实际点击后重置。

### Prompt E: Config Update

“给 SpecialActivity 增加三个开关并更新 GUI。”

Expected reads:

- `autoepicseven-rules.md` 的任务配置与入口章节
- `task-checklist.md` 的配置章节
- `wiki-guidance.md` 的 GUI 与 Config 章节
- `alasio-migration-workflow.md` 的配置与 worker 迁移章节

Expected behavior:

- 检测到 `module/config_alasio/` 后修改对应 nav 的 args/tasks YAML，不继续把旧 YAML 当作真相源。
- 同步 Alasio const 与 `config_manual.py` 的调度优先级。
- 使用 `SpecialActivity_Option` 扁平配置名。
- 对应生成器命令先请求用户批准，不手改生成模型和索引。
- 当前任务入口检查 `aes.py`，不引用旧项目。

### Prompt F: Asset Naming And Entry Lookup

“我要给国际服中文的活动免费召唤补图，asset 应该放哪里、叫什么，生成后又从哪里 import？”

Expected reads:

- `autoepicseven-rules.md` 的 Assets 章节
- `asset-task-conventions.md`

Expected behavior:

- 先搜索现有 `FREE_GACHA` 语义和 owner，不新建重复名字。
- 按语言差异选择 `share` 或 `global_cn`，源图只放根目录 `assets/`。
- 使用 `UPPER_SNAKE_CASE` 和 `.2` / `.SEARCH` 等合法后缀。
- 根据 module 路径推导 `tasks/<owner>/assets/assets_<module>.py`，生成后显式 from import。
- 能从 task.yaml、`aes.py`、可选 `entry.py` 继续追到实际状态循环。

### Prompt G: Python 3.14 And OCR

“保持当前 3.10 环境不动，在新目录验证 Python 3.14 和新 OCR 模型是否值得迁移，重点看十连结果。”

Expected reads:

- `alasio-migration-workflow.md` 的依赖阶段
- `alasio-migration-gotchas.md` 的 Python 与依赖、OCR 模型与十连识别章节

Expected behavior:

- 新建隔离目录，不修改现有 `.venv` 和发行基线。
- 先重新检查 cp314 wheel，不把 2026-09-12 的状态当永久事实。
- 把框选定位、十槽切分和模型识别分别评估。
- 使用同一批真实截图比较准确率、速度和包体，再判断是否迁移。

### Prompt H: Desktop Blank Or Not Found

“Alasio.exe 能打开，但先是白屏，补了一些文件后又显示 Not Found，帮我定位。”

Expected reads:

- `alasio-migration-workflow.md` 的 frontend 与 webapp 阶段
- `alasio-migration-gotchas.md` 的桌面端与前端章节

Expected behavior:

- 不把白屏和 `Not Found` 视为同一问题。
- 白屏检查 CSP hash 与 webapp 构建顺序；`Not Found` 检查独立 frontend 产物。
- 使用 `pnpm package`，不直接跳到 electron-builder。
- 检查唯一桌面端入口和残留后端进程。

### Prompt I: Release Update

“用新发行包更新我复制出来的安装目录，确认原端口、密码和任务配置都保留。”

Expected reads:

- `alasio-migration-workflow.md` 的发行组装、更新和分批提交章节
- `alasio-migration-gotchas.md` 的发行与更新章节

Expected behavior:

- 确认测试源是真实安装副本，不是开发仓库副本。
- 发行组装前直接申请权限，使用完整便携 CPython 和新 ReleaseName。
- 检查发行包不含实际三份配置，更新前后比较内容、ACL 和属性。
- 只从 `toolkit/WebApp/Alasio.exe` 启动并跑一个真实任务。

## Edge Cases

- 当前目录不是 worktree：确认真实仓库后再改，不自行创建替代项目。
- assets 缺失：停止该识别分支，列出需要用户放置的源图。
- 测试没有截图：先显式 screenshot 或注入 image_file，不让状态循环复用空帧。
- 测试权限错误：请求批准，不改用 uv/pytest。
- linked worktree 暂存或提交：命令执行前直接请求批准，不先制造 `index.lock` 权限错误。
- Windows 发行组装：命令执行前直接请求批准；失败半成品不覆盖，改用新的发行名。
- 发行更新目录缺少现有配置：只算首次安装验证，不能宣称用户配置得到保留。
- 用户要求删除代码：可以删除代码行或状态变量，但仍不删除文件；需要删除文件时说明限制。

## Quality Gates

- [ ] `SKILL.md` 少于 500 行，主要承担路由而不是百科全文
- [ ] 所有参考文件都由 `SKILL.md` 直接链接，没有多层引用链
- [ ] asset 命名、目录映射和入口定位只有 `asset-task-conventions.md` 保存详细规则
- [ ] description 同时包含触发请求、用户关键词和反触发范围
- [ ] 至少三个 should-trigger 和三个 should-not-trigger 样例
- [ ] 每个关键决策都有默认选项
- [ ] 其他游戏项目的名称、路径和源码依赖没有出现在实现指导中
- [ ] 开发圣经全文位于 `state-loop-bible.md`
- [ ] AutoEpicSeven 开头约束完整位于 `autoepicseven-rules.md`
- [ ] Alasio 标准流程和踩坑分别位于两个直接引用的 reference，没有复制过时 roadmap
- [ ] 迁移配置任务会选择 `module/config_alasio/`，旧配置链仅用于没有该目录的旧分支
- [ ] `git add` / `git commit` 和发行组装的权限申请发生在首次执行之前
