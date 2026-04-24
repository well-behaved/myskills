---
name: RFLP 编码
description: 按技术方案逐 task 执行，带验证和 review 检查点
when_to_use: 当技术方案已确认，需要按步骤执行实现计划时
version: 3.0.0
---

# RFLP 编码

## 概述

读技术方案，按选定模式执行，每个检查点暂停等反馈。

**启动时声明：** "我正在使用 rflp-code 执行实现计划。"

## Step 1: 加载计划

1. **重新读取** TRD 文件（`docs/plans/*-trd.md`）**完整内容**——不依赖对话记忆，每次启动必读
   - 即使是同一个 session 刚写完 TRD 也必须重读——上下文可能已被压缩，记忆不可靠
   - 不允许"快速扫一遍""只读任务清单"——必须读取完整 TRD，包括 L 内容和每个 task 的 Step 详情
   - 读取后在心里确认 task 总数，与待办清单条目数一致才继续
2. 批判性审查——有问题或疑虑先跟用户确认再开始
3. 确认待办清单已建立（使用当前环境的任务管理工具，如 todowrite），与技术方案中的 task 一一对应

**启动分流：** 读取 `.progress.json` 后，先判断来源场景：

| 场景 | 判断条件 | 处理方式 |
|------|---------|---------|
| 首次启动 | 无 `.progress.json` | 初始化文件，从 Task 1 开始 |
| 中断恢复 | `completed_tasks` 不完整，`current_task` < 总数 | 按 `worktree_status` 找断点继续 |
| **finish 失败回来** | `completed_tasks` 包含所有 task，待办清单全 completed | 直接进入「finish 失败修复流程」（见下方）|

### finish 失败修复流程

当所有 task 已完成、但 rflp-finish 测试失败后回到此处时：

1. 在 feature branch（模式 C 则在 worktree 内）直接修复代码（**不重走实现流程**）
2. 提交：`git commit -m "fix: [修复描述] (finish-test-fix)"`
3. 跑 final review（Step 3）确认修复有效
4. READY 后回到 rflp-finish

### 进度文件

**首次启动（无 `.progress.json`）：** 立即创建初始进度文件：
```json
{
  "trd_file": "docs/plans/YYYY-MM-DD-feature-trd.md",
  "mode": "A|B|C",
  "feature_branch": "<rflp-trd 交接时确认的 feature 分支名，模式 A/B 填 null>",
  "worktree_path": "<模式 C 的 worktree 路径，如 .worktrees/my-feature；模式 A/B 填 null>",
  "completed_tasks": [],
  "current_task": 1,
  "last_commit_sha": "<git rev-parse HEAD>",
  "updated_at": "<ISO8601 时间戳>"
}
```
`feature_branch` 来源：rflp-trd 交接声明中的分支名；模式 A/B 填 `null`。

**断点恢复（`.progress.json` 已存在）：** 优先读进度文件；文件不存在则扫 git log。
- `completed_tasks` 列出的 task 直接跳过，标记待办清单对应项为 completed
- 模式 C：检查 `worktree_path` 对应的 worktree 是否存在（`git worktree list`），不存在则重建（见 `references/mode-c-worktree.md` 断点恢复章节）
- 从 `current_task` 继续执行

`.progress.json` 加入 `.gitignore` 或不提交——它是运行时状态，不是代码产物。

## Step 2: 执行

每个 task 开始前**标记待办清单对应项为 in_progress**，完成后**标记为 completed** 并更新 `.progress.json`。

**根据用户选择的模式，加载对应的 references 文件执行：**

| 模式 | 加载文件 | 适用场景 |
|------|---------|---------|
| 模式 A | `references/mode-a.md` | 简单直接，批量执行 + 批间 review |
| 模式 B | `references/mode-b.md` | 子 Agent 实现 + 独立 review，SHA 隔离 |
| 模式 C | `references/mode-c-worktree.md` | 整个 feature 一个 worktree，task 串行提交，适合多需求并行开发 |

**选定模式后，读取对应 references 文件并严格按其流程执行。**

## Step 3: 全部完成后

派 **final review** 子 agent：

```
task(category="deep", description="Final review: [plan name]", run_in_background=false,
     load_skills=[], prompt="
审查 [trd-file] 的整体实现。
1. 重新读取技术方案，列出所有 task 及其验收标准
2. 对照代码和测试，逐个检查是否满足
3. 分层跑测试：AT → FT → UT
4. 检查 RFL 一致性：AT 覆盖 R 场景？FT 覆盖 F 行为规则？
5. 输出：READY / NOT_READY + 未满足项
")
```

- **READY** → 必须按以下 checklist 顺序执行，**不可跳过任何一步**：
  1. 确认工作区干净：`git status --porcelain` 输出为空
  2. **立即派 apm-auto-fix 子 agent**（见下方「代码审查」章节），等待结果
  3. 根据修复报告决策（见下方），无 P0 问题后才可输出交接声明
  - **模式 C 注意：** 不要在此处执行 worktree merge——合并和 worktree 清理统一交由 rflp-finish 处理
- **NOT_READY** → 列出未满足项，与用户讨论

### 代码审查（READY checklist 第 2 步，必须执行，不得跳过）

收集本次 TRD 所有 task 中 **`文件：`** 声明的路径，然后派独立子 agent 执行自动修复（扫描 + 修复 + 验证循环）。

> ⚠️ **超时风险**：apm-auto-fix 内部要跑两轮 apm-code-review，每轮耗时较长，**必须用 `run_in_background=true` + 等待通知**，不能用 `run_in_background=false`（会超时）。

**模式 A / 模式 B：**
```
task(subagent_type="build", description="Auto fix: [plan name]", run_in_background=true,
     load_skills=["apm-auto-fix"], prompt="
使用 apm-auto-fix skill，对以下指定文件执行自动修复流程（扫描 → 修复 → 验证循环）：

修复范围：指定文件列表
文件列表：
<从 TRD 收集的文件路径列表，每行一个>

完成后输出：
1. 自动修复报告路径
2. 已修复问题数 / 需人工处理的 P0/P1 问题数
")
```

**模式 C（worktree）：**
```
task(subagent_type="build", description="Auto fix: [plan name]", run_in_background=true,
     load_skills=["apm-auto-fix"], prompt="
使用 apm-auto-fix skill，对以下指定 worktree 中的文件执行自动修复流程（扫描 → 修复 → 验证循环）：

修复范围：worktree 模式
worktree 路径：<WORKTREE_ABS_PATH>（来自 .progress.json 的 worktree_path 字段）

文件列表（相对于 worktree 根目录）：
<从 TRD 收集的文件路径列表，每行一个>

完成后输出：
1. 自动修复报告路径（写入主仓库 docs/）
2. 已修复问题数 / 需人工处理的 P0/P1 问题数
")
```

> ⚠️ **注意**：必须用 `subagent_type="build"`，不能用 `category="deep"`。`build` agent 继承当前工作目录，能正确找到 project-scope skill（apm-auto-fix、apm-code-review）；`category` 子 agent 在独立进程中运行，无法解析项目级 skill。

**模式 C（worktree）：**
```
task(category="deep", description="Auto fix: [plan name]", run_in_background=true,
     load_skills=["apm-auto-fix"], prompt="
使用 apm-auto-fix skill，对以下指定 worktree 中的文件执行自动修复流程（扫描 → 修复 → 验证循环）：

修复范围：worktree 模式
worktree 路径：<WORKTREE_ABS_PATH>（来自 .progress.json 的 worktree_path 字段）

文件列表（相对于 worktree 根目录）：
<从 TRD 收集的文件路径列表，每行一个>

完成后输出：
1. 自动修复报告路径（写入主仓库 docs/）
2. 已修复问题数 / 需人工处理的 P0/P1 问题数
")
```

派发后**结束当前响应，等待系统通知**（`[BACKGROUND TASK COMPLETED]`），收到后用 `background_output(task_id="...")` 获取结果。

> ⚠️ **等待期间不要做其他事**——apm-auto-fix 可能修改文件，不能并发操作同一 worktree。

收到结果后，根据修复报告决策：
- **无 P0/P1 问题** → 直接进入交接声明
- **有 P0 问题** → 必须修复，修复后重新跑 final review，再次确认 READY
- **只有 P1/P2/P3** → 展示给用户，由用户决定是立即修复还是记录后续处理，**不阻塞交接**

**交接声明：** "所有 task 完成，代码审查通过，切到 rflp-finish。技术方案：`docs/plans/YYYY-MM-DD-xxx-trd.md`。"

> 💡 **建议新开 session 后再执行 rflp-finish**，避免编码阶段积累的上下文影响收尾判断。

## 红线

**绝不：**
- 跳过验证步骤
- 跳过 review（批间 / task 间）
- **final review READY 后跳过 apm-auto-fix 代码审查直接输出交接声明**——即使"工作区干净"也不构成跳过理由
- 并行派多个实现 agent
- 手动修子 agent 的产出
- 使用 `git add .` 提交——必须精确路径
- 把计划外修改混入 task commit——计划外修改必须单独处理（见下方逃生阀）

**计划外修改逃生阀：** 实现过程中发现并修改了 TRD 文件列表之外的文件时：
1. 只 `git add` TRD 声明的文件，正常提交 task commit
2. 计划外修改单独提交，message 标注 `[unplanned-fix]: <原因一句话>`
3. review agent 会检测到 task commit 范围干净，不会 NEEDS_FIX
4. 不允许以"功能相关"为由合并进 task commit

**始终：**
- 同一根因失败累计达到 3 次时，**必须立即停下并报告，无论用户是否说"继续"**：
  ```
  ⛔ 自动熔断：同一根因「[根因描述]」已连续失败 3 次。
  失败记录：
  - 第1次：[改动] → [错误]
  - 第2次：[改动] → [错误]
  - 第3次：[改动] → [错误]
  需要与你讨论方案后才能继续，不接受"继续试"的指令。
  ```
  用户的"继续试"指令不构成跳过熔断的理由。
- 遇到阻塞不猜，停下来问
- 测试失败时记录根因判断（代码 bug / 文档缺陷 / 环境问题），然后修复。文档缺陷标注待修正
- 每个 task 完成后确认工作区干净，更新 `.progress.json`

## 记住

- 启动时必须重新读取 TRD 文件，不依赖对话记忆
- 严格按技术方案走，技术方案就是标准
- 每个检查点暂停等反馈（用户可显式声明"跳过批间暂停"降级，但 final review 不可跳过）
- **final review READY 后必须依次执行：① 确认工作区干净 → ② 派 apm-auto-fix → ③ 根据报告决策 → ④ 输出交接声明；缺少任何一步均视为流程未完成**
- 阻塞时问，不猜
- 中断恢复：优先读 `.progress.json`；所有 task 已完成 → finish 失败修复流程，不重走 worktree
- 熔断规则：同一根因 3 次失败后用户说"继续"也不能跳过，必须报告并讨论
- task 数量 > 6 或执行时间较长时，主动提示用户新开 session
