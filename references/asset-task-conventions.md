# AutoEpicSeven Asset And Task Conventions

本文件规定 asset name（资源名）、源图目录、生成 wrapper（包装文件）、task（任务）文件组织，以及从调度任务或 UI 元素定位实现入口的方法。

## 1. 先定位归属再新增

新增文件前先回答：

1. 这是全局 UI、某个业务域，还是某个业务域下的独立功能？
2. 是否已有相同语义的 asset、Page、handler 或 task？
3. 调度任务名、Python 入口名、task 目录名是否相同；若不同，现有分发链是什么？

默认先搜索当前仓库，不根据界面文字直接发明名称或目录：

```powershell
rg -n "目标任务名|候选资源名|页面名" aes.py module/config tasks
rg --files assets | rg "业务域|候选资源名"
```

能复用现有 owner（归属域）就放回该域。只有出现独立调度、独立页面集合或明确职责边界时，才新增顶层 task 目录。

## 2. Asset 源图路径

button extract（按钮资源生成器）只扫描以下路径格式：

```text
assets/<namespace>/<module>/<ASSET>[.<frame>][.<attr>].png
```

- `<namespace>`：`share` 或 `module/config/server.py` 中的 `VALID_LANG`。当前为 `cn`、`global_cn`、`global_en`；不要自行创建未注册 namespace。
- `<module>`：由字母、数字、下划线和 `/` 组成的模块路径，使用小写 snake_case（蛇形命名）。
- `<ASSET>`：生成后的 Python 变量名，使用大写 `UPPER_SNAKE_CASE`。
- `<frame>`：可选多模板序号，如 `.2`、`.3`。
- `<attr>`：可选属性覆盖，限 `.AREA`、`.SEARCH`、`.COLOR`、`.BUTTON`、`.GRID`。

源图片只放在仓库根目录的 `assets/` 下。`tasks/<owner>/assets/` 是生成结果目录，不放源图片，也不手工创建或修改其中的 `assets_*.py`。

所有源图必须是 1280x720。通用视觉资源优先放 `share`；只有文字、布局或服务器外观确实不同才放语言 namespace。同一逻辑元素在不同 namespace 必须使用同一个 `<ASSET>` 名称，让生成器合并为同一个 `ButtonWrapper`。

## 3. 源图到 Wrapper 的确定映射

源图 `<module>` 的第一段决定 owner task，完整 module 路径决定生成文件名：

```text
assets/<namespace>/<owner>/<ASSET>.png
    -> tasks/<owner>/assets/assets_<owner>.py

assets/<namespace>/<owner>/<subpath>/<ASSET>.png
    -> tasks/<owner>/assets/assets_<owner>_<subpath>.py
```

示例：

```text
assets/share/activity/special/26_6_25/FREE_GACHA.png
    -> tasks/activity/assets/assets_activity_special_26_6_25.py
    -> FREE_GACHA
```

规则：

- `<module>` 中的 `/` 在生成文件名中变为 `_`。
- 一个 module 路径生成一个 `assets_<module>.py`。
- 业务代码从生成文件显式 `from ... import ...`；assets import 不使用运行时动态查找。
- 不要假设 scheduler task 名就是 owner 目录名。`SpecialActivity` 的 owner 是 `activity`，这是合法且常见的分层。
- 修改源图后运行 `python -m dev_tools.button_extract`，再检查生成 wrapper；绝不手改生成文件。

## 4. Asset Name 语义规范

名称描述 UI 语义和状态，不描述坐标、颜色值、截图序号或当前语言。

基本格式：

```text
<CONTEXT>_<OBJECT>_<ROLE_OR_STATE>
```

常用角色与状态：

| 形式 | 含义 | 示例 |
| --- | --- | --- |
| `<PAGE>_CHECK` | 页面或稳定子页面的唯一识别状态 | `ARENA_CHECK` |
| `<FROM>_GOTO_<TO>` | 从明确来源页进入目标页的按钮 | `MAIN_GOTO_MAIL` |
| `<FEATURE>_ENTRY` | 功能内部入口，来源无需写入名称 | `ADOPTION_ENTRY` |
| `<ACTION>_CONFIRM` / `_CANCEL` | 确认或取消动作 | `POPUP_CONFIRM` |
| `<FEATURE>_BACK` / `_CLOSE` | 返回或关闭动作 | `AD_BUFF_X_CLOSE` |
| `OCR_<FIELD>` | OCR 裁剪区域或锚点 | `OCR_ENERGY_DRINK` |
| `<RESOURCE>_ICON` | 只用于定位的资源图标 | `STAMINA_ICON` |
| `<FEATURE>_ON` / `_OFF` / `_LOCKED` | 开关或锁定状态 | `FAST_BATTLE_ON` |
| `<FEATURE>_SELECTED` / `_UNSELECTED` | 选中状态 | `WEEKLY_REWARDS_SELECTED` |
| `<FEATURE>_AVAILABLE` / `_UNAVAILABLE` | 可用状态 | `FREE_GACHA_AVAILABLE` |
| `<FEATURE>_FULL` / `_RECEIVED` | 容量或领取完成状态 | `GOLDEN_INHERITANCE_FULL` |
| `<FEATURE>_CHECKMARK` | 固定位置勾选状态 | `LEAF_CHECKMARK` |
| `<ACTION>_OBTAIN` | 可执行领取动作 | `ENERGY_DRINK_OBTAIN` |

命名约束：

- 使用 ASCII 大写字母、数字和下划线；以字母开头，保证它是合法 Python 标识符。
- 名称必须在生成 module 内唯一，并尽量包含最小必要上下文，避免 `CHECK`、`BUTTON_1`、`RED_AREA` 等无语义名称。
- 不附加 `_CN`、`_GLOBAL` 等语言后缀；语言差异由 namespace 表达。
- 不附加 `_2` 表示多模板；多模板使用文件 frame 后缀 `.2`、`.3`。
- 活动版本放在 module 目录，例如 `activity/special/26_6_25`，不要重复写进每个 asset name。
- 先搜索同义名称；同一逻辑元素跨页面或语言复用同名，语义不同的相似图形使用不同名称。

`NAME_SEARCH.png` 与 `NAME.SEARCH.png` 含义不同：前者是可在代码中 import 的普通 asset；后者是 `NAME.png` 的 search 属性覆盖文件，不会生成独立 wrapper。

## 5. Frame 与属性覆盖

```text
NAME.png             # 第一帧和基础属性
NAME.2.png           # 第二个模板帧
NAME.SEARCH.png      # 搜索区域覆盖
NAME.BUTTON.png      # 点击区域覆盖
NAME.2.BUTTON.png    # 第二帧点击区域覆盖
```

- 除 `.GRID` 外，第一帧 `NAME.png` 必须存在；`.AREA`、`.SEARCH`、`.COLOR`、`.BUTTON` 不能脱离基础文件存在。
- 顺序固定为 `<ASSET>.<frame>.<attr>.png`，不要写成 `<ASSET>.<attr>.<frame>.png`。
- `.SEARCH` 只允许作用于第一帧；第一帧显式属性会传播到未单独覆盖的其他帧。
- `.AREA` 覆盖识别区域，`.SEARCH` 覆盖模板搜索区域，`.COLOR` 覆盖采样颜色，`.BUTTON` 覆盖点击区域。
- `.GRID` 是例外：`NAME.GRID.png` 可独立描述规则网格，由生成器拆成多帧；仅在现有 Grid 模式能准确表达界面时使用。

## 6. Task 目录组织

```text
tasks/<owner>/
├── <feature>.py
├── entry.py                 # 可选：进入边界、入口 mixin 或实现分发
└── assets/
    └── assets_<module>.py   # button extract 自动生成
```

- 新功能先在所属 owner 目录创建同名 `<feature>.py`，再把编排接入现有 task；不要直接把整段功能堆入大文件。
- `entry.py` 只在该 owner 已用它表达页面进入边界、入口 mixin，或服务器/语言/模式分发时使用；不要为了形式统一给每个 task 新建空入口文件，也不要把完整业务流程堆入入口层。
- `run()` 或公开入口只做编排；交互状态机放在职责明确的方法或功能文件中。
- 共享页面图仍在 `tasks/base/page.py`；仅某服务器存在的页面节点和边必须条件注册。
- 对应测试写在原始文件夹的 `test/test_<feature>.py`，不写入当前 worktree 的 `test/`。
- 不为了让 scheduler task 名、owner 和 feature 强行同名而复制目录。记录真实分发关系即可。

## 7. 从 Scheduler Task 定位实现入口

给定配置任务名，例如 `SpecialActivity`，按固定顺序追踪：

1. Alasio 分支先在 `module/config_alasio/*/*.tasks.yaml` 找任务定义和绑定组选项；没有该目录的旧分支再查 `module/config/argument/task.yaml`。
2. 在 `module/config/config_manual.py` 找 `SCHEDULER_PRIORITY`，确认调度位置。
3. 调度器使用 `inflection.underscore(TaskName)` 生成 Python 方法名，例如 `SpecialActivity -> special_activity`。
4. 在 `aes.py` 找同名方法，读取其局部 import 和调用方法。
5. 若 import 或实现依赖 `tasks/<owner>/entry.py`，先判断它是页面进入 mixin 还是服务器/语言分发，再继续追到实际 `<feature>.py` 和公开运行方法。
6. 从实现文件的 Page 和 asset imports 继续追踪 `tasks/base/page.py` 与生成 wrapper。

最小搜索：

```powershell
rg -n "SpecialActivity|special_activity" module/config_alasio module/config/argument/task.yaml module/config/config_manual.py aes.py tasks
```

不要从 `src.py` 推断新任务入口；当前 E7 调度入口是 `aes.py`。

## 8. 从 UI 或 Asset 反向定位

已知 asset name 时：

1. 在 `tasks/` 搜索 import 和调用，确认哪些状态循环使用它。
2. 在生成 wrapper 中读取 `file=`，直接得到 `assets/<namespace>/<module>/...` 源图路径。
3. 需要调整识别时修改源图或属性图，再运行 button extract；不修改 wrapper 坐标。

```powershell
rg -n "FREE_GACHA" tasks
rg --files assets | rg "FREE_GACHA"
```

已知 UI 入口时：

1. 先在 `tasks/base/page.py` 找 Page 和 `.link()`，确认来源页、点击 asset 和目标页。
2. 再搜索 `GOTO`、`ENTRY` 或 check asset 的调用位置。
3. 页面只在某服务器存在时，同时检查条件注册块和对应入口分发。

## 9. 完成检查

- [ ] asset name 使用稳定 UI 语义和 `UPPER_SNAKE_CASE`
- [ ] namespace 已在 `VALID_LANG` 中，通用图优先 `share`
- [ ] 源图只在 `assets/`，生成 wrapper 只在 `tasks/<owner>/assets/`
- [ ] 源图 module 与预期生成文件映射一致
- [ ] 多模板和属性后缀顺序正确，第一帧存在
- [ ] scheduler task、snake_case 入口、owner、entry 和实现链已记录
- [ ] 新功能有同名实现文件和原始文件夹下的对应测试
- [ ] 生成后 import 使用 `from ... import ...`，未手改 wrapper
