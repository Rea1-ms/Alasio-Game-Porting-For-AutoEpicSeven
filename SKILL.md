---
name: alas-game-porting-for-autoepicseven
description: >
  Load when the user asks to develop, port, review, debug, or refactor AutoEpicSeven/E7 automation, including tasks, ALAS state loops, pages/routes, assets/button extraction, OCR, server/language dispatch, scheduler/config wiring, or emulator screenshot tests. Also load for explicit $alas-game-porting-for-autoepicseven requests. Do not use for Epic Seven gameplay advice, unrelated automation projects, or generic Python work outside AutoEpicSeven.
---

# AutoEpicSeven ALAS Game Porting

Version: 0.1 (2026-07-18)

把 AutoEpicSeven 当作已经成熟的 E7 自动化项目直接开发。先读当前仓库代码、git 差异和本 skill 的参考资料。

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
| 维护或验收本 skill | `references/skill-evals.md` |
| 其他 AutoEpicSeven 开发 | 扫描所有参考文件标题，只读取与当前任务直接相关的章节 |

## Execution Defaults

1. 确认当前目录和 git worktree。代码只改当前 worktree，不向原始文件夹写业务代码。
2. 先读当前实现、相关 assets 调用和 git 差异，再定义输入页面、输出页面、成功状态、异常状态和服务器范围。
3. 新功能在当前 worktree 的 `tasks/<task>/` 下建立同名实现文件；测试只放在原始文件夹下的 `test/`，不得写入当前 worktree 的 `test/`，并保持整个测试目录被 git 忽略。
4. 实现交互时使用截图优先的单层状态循环；顺序固定为退出状态、点击/动作状态、通用处理器。
5. 资产缺失就停止并提醒用户放置源图。不要用 `try/except` 隐藏缺省资产，不要修改生成的 assets wrapper。
6. 配置只改 YAML 源文件和 `config_manual.py`。运行 config updater 前必须请求用户批准。
7. 每完成一个模块就补对应测试；先离线截图验证，再按风险决定是否请求批准运行模拟器测试。
8. 修改后运行定向静态检查和差异格式检查。不能运行的验证要明确说明。

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

## Validation Loop

1. 运行针对改动文件的 ruff 检查；有错误就先修第一个真实问题，再重跑。
2. 运行 `git diff --check`，确认 LF、空白和补丁格式正常。
3. 运行目标测试；测试需要权限时直接请求批准，不尝试 uv 或 pytest。
4. 页面或分服注册有改动时，至少检查目标服务器和非目标服务器的导入、页面扫描与寻路行为。
5. 最多连续修复三轮同类失败；仍失败时保留日志和截图，说明阻塞状态，不用宽泛 fallback 掩盖错误。

## Completion Checklist

- [ ] 当前 worktree、服务器范围、页面进出和成功条件已确认
- [ ] 没有删除文件，没有手改生成文件，没有隐藏缺失 assets
- [ ] 状态循环没有固定 sleep、负面退出或带副作用退出
- [ ] OCR 显式使用规范化语言，assets 仍由 server.lang 选择
- [ ] 页面节点、路由、返回关系和入场后弹窗状态已覆盖
- [ ] 配置源、调度顺序、aes.py 入口和 i18n 按任务需要完成
- [ ] 对应测试已放入原始文件夹下被忽略的 `test/`，并完成可执行验证
- [ ] 日志足以解释失败位置，脆弱状态有详尽英文注释
