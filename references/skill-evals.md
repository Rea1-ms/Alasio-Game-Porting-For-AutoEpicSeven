# Skill Evaluation v0.1

本文件用于验证 skill 的触发边界、路由和关键规则是否生效。维护 skill 时先运行这些思维测试，再改 description 或正文。

## Should Trigger

1. “用 ALAS 状态机给 AutoEpicSeven 新增竞技场奖励领取任务，补页面、assets 和测试。”
   - 原因：E7 任务、状态循环、页面、assets、测试全部命中。
2. “审核一下 `run_free_gacha`，为什么日志只算一抽，实际点了三次继续免费召唤？”
   - 原因：E7 状态计数与召唤实机排错命中。
3. “按服务器给 E7 的限时活动页面分流，只支持国际服中文，并接入调度配置。”
   - 原因：页面条件注册、语言 assets、任务配置命中。

## Should Not Trigger

1. “Epic Seven 新手应该先练哪个角色？”
   - 原因：游戏攻略，不是自动化开发。
2. “给普通 Python 项目写一个调用在线 OCR 的命令行工具。”
   - 原因：与 AutoEpicSeven 和 ALAS 框架无关。
3. “给 Vite 页面加一个语言切换菜单。”
   - 原因：前端开发，与本 skill 无关。

## Real Prompt Routes

### Prompt A: New Task

“新增一个 E7 每日签到任务，从主界面进入签到页，领取后回主界面。”

Expected reads:

- `autoepicseven-rules.md`
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

Expected behavior:

- 修改 task/argument/gui YAML，必要时 default/override。
- 使用 `SpecialActivity_Option` 扁平配置名。
- updater 命令先请求用户批准。
- 当前任务入口检查 `aes.py`，不引用旧项目。

## Edge Cases

- 当前目录不是 worktree：确认真实仓库后再改，不自行创建替代项目。
- assets 缺失：停止该识别分支，列出需要用户放置的源图。
- 测试没有截图：先显式 screenshot 或注入 image_file，不让状态循环复用空帧。
- 测试权限错误：请求批准，不改用 uv/pytest。
- 用户要求删除代码：可以删除代码行或状态变量，但仍不删除文件；需要删除文件时说明限制。

## Quality Gates

- [ ] `SKILL.md` 少于 500 行，主要承担路由而不是百科全文
- [ ] 所有参考文件都由 `SKILL.md` 直接链接，没有多层引用链
- [ ] description 同时包含触发请求、用户关键词和反触发范围
- [ ] 至少三个 should-trigger 和三个 should-not-trigger 样例
- [ ] 每个关键决策都有默认选项
- [ ] 其他游戏项目的名称、路径和源码依赖没有出现在实现指导中
- [ ] 开发圣经全文位于 `state-loop-bible.md`
- [ ] AutoEpicSeven 开头约束完整位于 `autoepicseven-rules.md`
