# AutoEpicSeven Task Checklist

新增或重写一个任务时按此顺序执行。不要先搭大骨架；每次只完成一个用户可见能力。

## 1. Scope

- [ ] 任务名称、服务器和语言范围已确定
- [ ] 输入页面、输出页面和成功状态已确定
- [ ] 正常完成、今日已完成、资源不足、背包已满和网络异常已区分
- [ ] 所需 assets、OCR 字段、配置项和调度周期已列出
- [ ] 已阅读当前任务目录、相似 E7 任务和相关 git diff
- [ ] scheduler task、snake_case 入口、owner 目录和实际实现链已追踪

## 2. File Placement

- [ ] 在 `tasks/<owner>/` 下创建与功能同名的 `<feature>.py`
- [ ] `entry.py` 只在现有 owner 需要页面进入边界、入口 mixin 或实现分发时使用，不创建空入口层
- [ ] `run()` 只做编排，具体交互放在可独立测试的方法
- [ ] assets import 使用 `from ... import ...`
- [ ] 不修改生成的 `assets_*.py`
- [ ] 在原始文件夹下的 `test/` 新建对应 `test_*`，不写入当前 worktree 的 `test/`

## 3. Assets And Detection

- [ ] 截图分辨率为 1280x720
- [ ] asset name 使用有语义的 `UPPER_SNAKE_CASE`，未包含语言、坐标或截图序号
- [ ] 通用资源优先放 `share`，差异资源放对应服务器/语言目录
- [ ] 源图只放在根目录 `assets/`，预期生成 wrapper 路径已按 module 映射确认
- [ ] 多模板使用 `.2` / `.3`，属性覆盖使用合法 attr 后缀且第一帧存在
- [ ] 固定位置亮灰判定优先 `match_color()`
- [ ] 先定位再判色使用 `match_template_color()`
- [ ] 只判断存在使用 `match_template()` 或 `match_template_luma()`
- [ ] 搜索区域足够小，模板具有区分度
- [ ] 修改源图后运行 button extract，并只检查生成结果
- [ ] 缺少源图时停止实现该识别分支并提醒用户，不写静默 fallback

## 4. Page Contract

- [ ] Page 有唯一 `check_button`
- [ ] 页面类型是正式页、popup 还是 overlay 已明确
- [ ] 国服/国际服存在性和 assets namespace 已明确
- [ ] 入口、BACK、关闭键、页内按钮的去向已连边
- [ ] `dynamic_return_button`、`shared_toolbar`、后台托管限制已决定
- [ ] 入场后二段点击、周奖励、结算、网络弹窗有独立状态处理
- [ ] 仅单服务器存在的页面按服务器条件注册和连边
- [ ] 目标服务器和非目标服务器都完成 import 与寻路回归检查

## 5. State Loop

状态循环使用以下骨架：

```python
def _execute_feature(self, skip_first_screenshot=True):
    while 1:
        if skip_first_screenshot:
            skip_first_screenshot = False
        else:
            self.device.screenshot()

        # End: positive, side-effect-free states only.
        if self.appear(FEATURE_COMPLETE):
            return True
        if self.appear(FEATURE_ALREADY_DONE):
            return False

        # Actions: every click must be retryable.
        if self.appear_then_click(FEATURE_ENTRY, interval=2):
            continue
        if self.appear_then_click(FEATURE_CONFIRM, interval=2):
            continue

        # Shared handlers run after feature-specific states.
        if self.handle_popup_confirm():
            continue
        if self.handle_network_error():
            continue
```

- [ ] 循环顶部截图，不在尾部截图
- [ ] 独立调用前已有一张显式截图
- [ ] 退出条件在点击条件之前
- [ ] 退出只使用正面状态，不使用 `not appear(...)`
- [ ] 不使用 `appear_then_click()` 或 handler 直接退出
- [ ] 点击有合理 interval，纯退出识别没有 interval
- [ ] 没有用 sleep 推进状态
- [ ] `handle_*` 仅在执行操作时返回 True
- [ ] 状态变量在所有状态变化动作后正确重置
- [ ] 脆弱门控和状态变量有详尽英文注释

## 6. OCR

- [ ] 从 `Emulator_GameLanguage` 读取 OCR 语言，auto/空值回退 cn
- [ ] 每个 OCR 类显式传 `lang=lang`
- [ ] 没有把 `server.lang` 或 `global_cn` 传给 OCR
- [ ] 原始 OCR 文本保留在调试日志
- [ ] 格式、范围和 E7 业务语义都有校验
- [ ] 没有用会制造“格式合法、语义错误”数据的宽泛 fallback
- [ ] Counter 允许当前值合法超过恢复上限，除非该字段明确是硬上限

## 7. Config And Scheduler

- [ ] `task.yaml` 定义任务与选项组，并包含 `Scheduler`
- [ ] `argument.yaml` 定义选项、默认值和校验
- [ ] `gui.yaml` 定义额外 GUI 文本
- [ ] `default.yaml` / `override.yaml` 按需要更新
- [ ] `config_manual.py` 中调度优先级合理
- [ ] 当前任务入口加到 `aes.py`
- [ ] 已确认 TaskName 经 `inflection.underscore()` 后与 `aes.py` 方法名一致
- [ ] 配置属性使用 `Group_Argument` 扁平名称
- [ ] 所有支持语言翻译完整，没有占位路径
- [ ] 运行 config updater 前已请求用户批准

调度顺序默认遵循：重启、短时收取、每日奖励、日常战斗、常规战斗、纯消磨时间。活动收益很高不等于可以绕过页面、服务器和资源安全检查；优先级变化要结合活动有效期和失败影响决定。

## 8. Failure Policy

- [ ] 可预期子状态有局部 Timer 或重试上限
- [ ] 战斗和长动画没有不合理的短超时
- [ ] 人工处理状态抛 `RequestHumanTakeover`
- [ ] 不可恢复业务矛盾抛项目现有业务异常
- [ ] 框架 stuck / too-many-click 保护仍生效
- [ ] “今天已完成”与“脚本失败”没有混为同一个返回值或调度结果

## 9. Tests

- [ ] 离线截图覆盖主要识别和 OCR 分支
- [ ] 状态循环测试前显式截图
- [ ] 覆盖首次进入、今日已完成、按钮点击未生效和弹窗打断
- [ ] 分服页面覆盖支持与不支持组合
- [ ] 稀有中间状态没有成为唯一完成或计数信号
- [ ] 使用 `.venv/Scripts/python.exe`，没有尝试 pytest 或沙箱 uv
- [ ] 权限失败时请求批准后重跑

## 10. Verification

1. 对改动 Python 文件运行定向 ruff。
2. 运行对应离线测试。
3. 运行 `git diff --check`。
4. 检查没有删除文件、没有手改生成文件、没有提交 test 或秘密信息。
5. 检查每个新 asset 的源图、生成 wrapper 和 import 路径能够互相追溯。
6. 需要实机时，从受支持页面和任务调度入口各验证一次，检查日志与结果截图。

完成标准不是“理想路径点通”，而是按钮偶尔点击失败、截图偶尔损坏、设备变慢时，流程仍能通过下一帧状态继续推进或明确失败。
