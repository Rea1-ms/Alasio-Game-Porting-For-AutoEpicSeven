# ALAS-Game-Porting-For-AutoEpicSeven

面向 AutoEpicSeven（自动化项目）的 Codex skill（技能），用于开发、移植、审核和调试 ALAS（二代游戏自动化框架）风格的 E7 自动化功能。

当前版本：v0.1（0.1 版本）

## 适用范围

- task（任务）实现与状态循环
- Page（页面对象）注册、路由和服务器分流
- assets（识别资源）与 button extract（按钮资源生成）流程
- OCR（光学字符识别）语言、后处理和资源栏识别
- scheduler（调度器）、配置和 GUI（图形界面）接入
- 截图测试、实机日志分析和回归检查

本 skill（技能）面向 AutoEpicSeven（自动化项目）开发，不涉及 E7 游戏攻略、无关自动化项目或普通 Python（编程语言）问题。

## 设计目标

这个仓库独立维护 skill（技能）本身，不包含 AutoEpicSeven（自动化项目）的业务代码、assets（识别资源）或测试截图。

AutoEpicSeven（自动化项目）通过 git submodule（Git 子模块）引用指定版本。

skill（技能）已经包含开发所需的状态循环规范、项目约束、wiki（规则整理）和开发经验，使用时不依赖额外资料。

## 目录结构

```text
alas-game-porting-for-autoepicseven/
├── README.md
├── SKILL.md
├── agents/
│   └── openai.yaml
└── references/
    ├── autoepicseven-rules.md
    ├── development-lessons.md
    ├── skill-evals.md
    ├── state-loop-bible.md
    ├── task-checklist.md
    └── wiki-guidance.md
```

- `SKILL.md`：触发说明、任务路由、执行默认值和验收清单。
- `agents/openai.yaml`：Codex 展示名称、简介和默认调用提示。
- `references/autoepicseven-rules.md`：AutoEpicSeven（自动化项目）的强制开发约束。
- `references/state-loop-bible.md`：ALAS（自动化框架）状态循环完整规范。
- `references/task-checklist.md`：新增或重写任务时的检查清单。
- `references/development-lessons.md`：现有模块和实机排错经验。
- `references/wiki-guidance.md`：识别、页面、异常、配置等框架原则。
- `references/skill-evals.md`：触发边界和行为验收样例。

## 安装

### 用户级安装

将仓库克隆到 Codex（智能编程助手）的 skills（技能）目录：

```powershell
git clone <skill-repository-url> "$env:CODEX_HOME/skills/alas-game-porting-for-autoepicseven"
```

重新启动 Codex（智能编程助手）后即可调用。

### 作为项目子模块

在 AutoEpicSeven（自动化项目）根目录执行：

```powershell
git submodule add <skill-repository-url> .agents/skills/alas-game-porting-for-autoepicseven
git submodule update --init --recursive
```

`.agents/skills/alas-game-porting-for-autoepicseven` 是 repository skill（仓库级技能）位置。项目会记录 skill（技能）的具体提交，不会自动跟随远端变化。

克隆包含该子模块的项目时使用：

```powershell
git clone --recurse-submodules <project-repository-url>
```

已有项目副本可执行：

```powershell
git submodule update --init --recursive
```

## 使用

显式调用：

```text
使用 $alas-game-porting-for-autoepicseven，为 AutoEpicSeven 新增一个状态驱动的每日任务。
```

也可以直接描述 AutoEpicSeven（自动化项目）的 task（任务）、Page（页面对象）、assets（识别资源）、OCR（光学字符识别）、调度或测试问题，由触发描述自动加载本 skill（技能）。

## 核心约定

- Windows 11、PowerShell 7、UTF-8（统一编码）和 LF（换行）。
- 业务代码只修改当前 worktree（工作树）。
- 测试写入原始文件夹下被 git（版本控制）忽略的 `test/`，不写入当前 worktree（工作树）的同名目录。
- 禁止删除文件，不手改生成配置和生成 assets wrapper（识别资源包装文件）。
- 游戏交互使用截图驱动的状态循环，不使用固定 sleep（等待）推进流程。
- assets（识别资源）语言与 OCR（光学字符识别）模型语言分开处理。
- 配置生成器运行前需要用户批准。

详细规则以 `SKILL.md` 和 `references/` 为准。
