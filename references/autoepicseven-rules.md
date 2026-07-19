# AutoEpicSeven Repository Rules

本文件是 AutoEpicSeven 开发的强制约束。执行任何代码修改前都要读取；与一般 ALAS 习惯冲突时，以本文件和用户当前指令为准。

## 运行环境

- Windows 11、PowerShell 7。
- 所有文本使用 UTF-8。
- 所有换行使用 LF (`\n`)。
- 禁止删除文件，只允许修改或新增。
- 不使用破坏性 git 命令，不覆盖用户已有改动。

## Worktree 与目录

- “原始文件夹”指创建当前 worktree 的主检出目录，也就是当前仓库 git common dir 所在 `.git` 目录的父目录。先解析 `git rev-parse --git-common-dir` 的结果再定位，不在 skill 中硬编码盘符、用户名或机器路径；当前仓库不是 linked worktree 时，原始文件夹就是当前仓库根目录。
- 代码修改必须发生在当前 worktree，不得写回原始文件夹的业务代码。
- 用户指定测试统一放在原始文件夹下的 `test/`，这是原始文件夹唯一允许的开发写入例外；整个 `test/` 保持在 `.gitignore` 中，不单独提交测试文件。
- 开始前检查 `git status --short` 和相关 diff；遇到用户已有改动时在其基础上继续，不还原。

## 编辑边界

- 不手改生成文件：
  - `module/config/argument/args.json`
  - `module/config/argument/menu.json`
  - `module/config/config_generated.py`
  - `tasks/*/assets/assets_*.py`
- assets import 一律使用 `from ... import ...`，与项目现有风格一致。
- 修改函数名、变量名或类名时，不自行执行全仓搜索替换。先完成可独立修改的代码，再汇总旧名到新名的映射，交给用户通过 IDE 批量重命名。
- 新增功能时，先在对应 `tasks/<task>/` 下建立与功能同名的实现文件，不把新逻辑继续堆入无关旧模块。
- 对反复修复、容易回归的状态变量、计数、时序门控和 fallback 必须写详尽英文注释。注释要说明：状态代表什么、何时重置、为什么不能按直觉简化、简化后会怎样失败。
- 页面跨越方法使用 Google 风格 docstring，并写 `Pages:` 的输入和输出状态。
- 不为了顺手整洁而重构当前任务之外的代码。

## 任务配置与入口

新增或修改任务时检查以下源文件：

- `module/config/argument/task.yaml`：任务包含哪些选项组，任务必须包含 `Scheduler`。
- `module/config/argument/argument.yaml`：选项组、选项、默认值、类型和校验。
- `module/config/argument/gui.yaml`：GUI 上额外出现的文本。
- `module/config/argument/default.yaml`：项目实际需要的默认任务配置。
- `module/config/argument/override.yaml`：锁定、隐藏或按服务器覆盖的值。
- `module/config/config_manual.py`：`SCHEDULER_PRIORITY` 调度顺序，修改后立即生效。
- `aes.py`：当前任务入口。`src.py` 是保留旧文件，不再作为新增 E7 任务的默认入口。

完成 YAML 修改后需要运行：

```powershell
.\.venv\Scripts\python.exe -m module.config.config_updater
```

运行这条命令前必须请求用户批准；不要先试跑，因为未批准时通常会出现权限错误。生成后检查所有支持语言的 i18n，不留下路径式占位文本。

配置访问使用扁平属性名：

```python
self.config.SpecialActivity_GetDailyReward
```

不要写嵌套属性：

```python
self.config.SpecialActivity.GetDailyReward
```

批量修改配置使用 `self.config.multi_set()`，任务结束使用 `task_delay(...)`，需要后续任务使用 `task_call(...)`。

## 测试

- 每写完一个模块，就在原始文件夹下的 `test/` 增加对应 `test_*`；不要写入当前 worktree 下的 `test/`。
- 不使用 pytest；项目环境没有 pytest。
- 不尝试在沙箱中使用 uv；本项目测试直接使用 `.venv/Scripts/python.exe`。
- 单独运行状态循环或任务前，显式执行一次 `self.device.screenshot()`，否则复用首帧时可能没有可用截图，导致跳过或报错。
- 离线识别优先通过 `image_file` 注入截图，避免消耗游戏内一次性资源。
- 测试命令遇到权限错误时，请求用户批准后重跑，不改用不等价的测试方式。
- 修改 `tasks/base/page.py` 的服务器分支或条件注册后，要检查非目标服务器导入、全局页面扫描、寻路和返回路径，避免条件名称不存在或错误页面被注册。
- 测试留在 git 忽略目录，不用 `git add -f` 强行提交。

## Assets

- asset name、源图目录到 wrapper 的映射、frame/attr 后缀和入口定位以 `asset-task-conventions.md` 为唯一详细来源。
- assets 的唯一来源是 button extract 流程；源图片只放入仓库根目录的 `assets/`，`tasks/<owner>/assets/` 只存放生成结果。
- 生成的 `assets_*.py` 只能读取和 import，绝不能手改或放入源图片。
- 缺省 assets 不要用 `try/except`、动态 getattr 或静默 fallback 隐藏。没有就是没有，漏了就是漏了；明确提醒用户补放源图片。
- 同名元素只是不同服务器外观不同：在同一个 assets 定义中保留同名 `ButtonWrapper`，不支持的一侧使用 `None`，让页面层保持统一 import。
- 页面节点只存在于某个服务器：Page 注册、路由连边和入口逻辑都必须按服务器条件分开，不能只分 assets。
- 资源优先放 `share`。仅语言或服务器不同的图放 `cn`、`global_cn` 等对应目录。
- 短期只支持国际服中文的活动资源可放 `global_cn`；等国服同步后，再把相同资源迁回 `share` 并清理旧分支。清理涉及删除时只提醒用户，不自行删除。

## 颜色与模板匹配

按问题性质选择识别方式：

1. 固定位置、按钮不会漂、状态主要靠整体变灰或变亮：使用 `match_color()`。
2. 位置可能偏移，或同类元素很多，需要先确认“就是这个按钮”再判亮灰：使用 `match_template_color()`。
3. 只判断元素是否存在，不关心亮灰：使用 `match_template()` 或 `match_template_luma()`。

默认规则：

- 能用 `match_color()` 稳定解决，就不要上 `match_template_color()`。
- 只要存在“先定位再判色”，直接使用 `match_template_color()`。
- 模板要缩小到稳定且有区分度的区域；小而重复的模板容易产生密集多匹配。
- 多目标匹配后必须按可靠的界面几何关系配对，例如同一商品行的 Y 中心，而不是固定取第一个结果。

主界面与遮罩的双重判断示例：

```python
def is_in_main(self, interval=0):
    """
    Check whether the main page is visible without a popup overlay.

    The home-only star identifies the page. The fixed menu color then
    distinguishes a usable page from the same page dimmed by an overlay.
    """
    self.device.stuck_record_add(WHITE_STAR)

    if interval and not self.interval_is_reached(WHITE_STAR, interval=interval):
        return False

    appear = False
    if WHITE_STAR.match_template_luma(self.device.image):
        if POPUP_OVERLAY.match_color(self.device.image, threshold=30):
            appear = True

    if appear and interval:
        self.interval_reset(WHITE_STAR, interval=interval)

    return appear
```

这里模板负责确认主页，颜色负责确认没有遮罩；不要把复杂按钮的平均颜色判断误写成“定位按钮”的方案。

## OCR 语言

`server.lang` 是 assets 的命名空间，常见值为 `global_cn`、`global_en`；OCR 模型只接受项目规范化语言，如 `cn`、`en`、`jp`、`tw`，再映射到模型语言。两者不能混用。

统一流程：

1. 读取 `self.config.Emulator_GameLanguage`。
2. `auto`、空值或无效自动值回退到 `cn`。
3. 所有 `Ocr`、`Digit`、`DigitCounter`、`Duration` 都显式传入 `lang=lang`。
4. assets 继续由 `server.lang` 选择，不把 OCR 语言反向用于 assets。
5. OCR 结果必须做格式、范围和界面语义校验；保留原始 OCR 文本到调试日志。

最小示例：

```python
lang = self.config.Emulator_GameLanguage
if not lang or lang == 'auto':
    lang = 'cn'
ocr = Digit(OCR_VALUE, name='OCR_VALUE', lang=lang)
```

不要这样写：

```python
Digit(OCR_VALUE, lang=server.lang)
```

## 超时与异常

- 可预期、范围明确的子状态增加局部 Timer 或重试上限，例如“无免费次数”的短判定、确认弹窗出现窗口、滚动稳定检测。
- 长时间动画、战斗主循环不强制短超时。
- 最终由框架的 stuck 和 too-many-click 保护处理无法识别的死循环。
- 不可恢复的业务矛盾使用项目现有异常；需要人工处理的游戏状态使用 `RequestHumanTakeover`。

背包已满示例：

```python
if self.appear(PACKAGE_FULL, interval=0):
    logger.critical(
        'Combat: package full detected after COMBAT_START. '
        'Please clear inventory before starting combat again.'
    )
    raise RequestHumanTakeover
```

不要把资产缺失、程序错误或业务矛盾降级成“正常完成”。

## 页面要求

新增页面前必须明确至少五类信息：

1. 页面身份：页面名、唯一 `check_button`，以及正式页、popup 或 overlay 类型。
2. 服务器范围：国服是否存在、国际服是否存在；资源放 `share`、`cn` 还是 `global_cn`。
3. 路由关系：从哪些页面通过哪些入口进入；BACK、关闭键和页内按钮分别返回哪里。
4. 页面属性：是否需要 `dynamic_return_button`、`shared_toolbar`，是否无法进入后台托管战斗。
5. 入场状态：到达页面后是否还要点击一步；是否会被周奖励、结算、网络弹窗等打断。存在干扰就写独立状态循环，不能只依赖 `ui_goto`。

每个 Page 使用唯一识别状态。跨模块跳转优先 `ui_goto(page_xxx)`；只有调用者已经满足业务模块私有入口的页面前置条件时，才复用私有 enter 方法。

## 不同服务器分流

采用“公共流程 + 服务器分发”：

- 公共业务逻辑保持在一起，短期差异放在分服 assets 和少量分支方法。
- 判断服务器使用 `module/config/server.py` 的 `is_cn_server`、`is_oversea_server`，上层通常传 `self.config.Emulator_PackageName`。
- 服务器状态必须在依赖它的模块 import 之前完成初始化；条件页面注册尤其如此。
- 只有某服务器存在的 Page，必须按服务器条件注册和连边，否则另一侧的页面扫描与寻路会把不存在页面算进去。
- 模块入口按服务器和语言分发时，不支持的组合应正常记录并调度到下一周期，不应在 import 阶段崩溃。
- 不为了尚未支持的语言创建虚假页面占位或空 assets；支持范围以当前任务明确声明为准。

## 日志

- 日志需要让人能还原状态变化：任务阶段、关键配置、识别结果、计时器退出原因和最终状态。
- `logger.hr` 按任务、阶段、子阶段分级使用，避免每件事都强调。
- `logger.warning` 只用于确实需要关注但流程仍可继续的异常，不用于普通分支。
- 排查 OCR 时记录原始文本、候选区域和解析结果；排查模板时记录匹配数量和区域。
- 对一次性资源操作，失败时优先保存截图和上下文，不盲目重试完整流程。
