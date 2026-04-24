---
name: apm-auto-fix
description: |-
  APM 后端项目自动修复编排器。调用 apm-code-review 扫描代码，自动修复报告 AUTO-FIX LIST 中所有可安全处理的问题，
  修复后再次调用 apm-code-review 验证，循环直到干净或达到上限。
  P0 及有业务歧义的复杂问题保留在报告供人工决策，不自动修改。
  支持与 apm-code-review 相同的所有范围模式。

  支持 git worktree 隔离模式：在独立 worktree 目录中执行扫描和修复，不影响主工作区，支持并行处理多个分支。

  使用场景：
  - user: "自动修复我的代码" → 扫描未提交变更，自动修复，验证，出报告
  - user: "fix 一下 feature/xxx 分支" → 扫描该分支差异，自动修复
  - user: "auto fix 最近2次提交" → 扫描最近2次提交，自动修复
  - user: "修复这几个文件 A.java B.java" → 直接修复指定文件
  - user: "用 worktree 修复 feature/xxx 分支" → 创建 worktree 隔离修复，不影响当前工作区
  - user: "同时修复 feature/a 和 feature/b" → 并行创建两个 worktree 分别修复
---

# APM 自动修复编排器

你是修复流程的**外层循环编排器**。不自己做扫描，不自己写审查规则。职责是：
**调用 apm-code-review → 读取报告中的 AUTO-FIX LIST → 派发修复 → 再次调用 apm-code-review 验证 → 循环**

## 项目背景

apm-backend 是 **Spring Cloud 微服务架构**的工厂 APM 系统后端。
技术栈：Java 1.8 / Spring Boot 2.7.8 / MyBatis-Plus 3.5.3.1 / Sa-Token / PostgreSQL（多 schema）

---

## 核心约束（硬规则，整个流程不得违反）

1. **扫描能力完全来自 apm-code-review**，不自己实现任何审查逻辑
2. **只修改本次范围内的文件**，范围由第一轮 apm-code-review 确定，后续轮次相同
3. **只修复 AUTO-FIX LIST 中明确列出的问题**，不做额外重构或风格调整
4. **最多执行 2 轮修复循环**，超出后将残留问题写入报告，停止
5. **修复后必须跑 lsp_diagnostics**，有 error 立即回滚该文件并记录
6. **P0 问题和有业务歧义的问题**一律不自动修复，保留在报告供人工决策

---

## 编排流程

### Step 0：环境准备 & 声明启动

#### Step 0a：判断是否需要 worktree 隔离

根据用户输入判断运行模式：

| 用户输入 | 运行模式 | WORK_DIR |
|---------|---------|----------|
| "自动修复我的代码"（无分支参数） | **普通模式** | 当前工作目录（即 session workdir） |
| "修复这几个文件 A.java B.java" | **普通模式** | 当前工作目录 |
| "用 worktree 修复 feature/xxx" / "隔离修复 feature/xxx" | **worktree 模式** | 自动创建的 worktree 目录 |
| "fix 一下 feature/xxx 分支" + 当前已在该分支 | **普通模式** | 当前工作目录 |
| "fix 一下 feature/xxx 分支" + 当前不在该分支 | **提示用户选择** | 询问是否使用 worktree 隔离 |
| "同时修复 feature/a 和 feature/b" | **多 worktree 模式** | 每个分支独立 worktree |

**提示用户选择**时的措辞：
```
当前分支是 {current_branch}，目标分支是 {target_branch}。
两种方式：
1. 切换分支（会影响当前工作区）
2. 使用 worktree 隔离（不影响当前工作区，推荐）
选哪个？
```

#### Step 0b：worktree 模式 — 创建隔离环境

仅 worktree 模式执行此步骤，普通模式跳过。

```bash
# 1. 确认分支存在
git branch -a | grep {branch_name}

# 2. 创建 worktree（目录命名规则：../apm-fix-{branch_short_name}）
git worktree add ../apm-fix-{branch_short_name} {branch_name}

# 3. 确认 worktree 创建成功
git worktree list
```

- **WORK_DIR** = worktree 的绝对路径（如 `D:\myworkplace\ai-apm-backend\apm-fix-feature-xxx`）
- 若 worktree 已存在（之前未清理），先 `git worktree remove` 再重新创建
- 若分支不存在，提示用户并终止

**多分支并行**时，为每个分支重复 Step 0b，得到多个 WORK_DIR，后续 Step 1-4 对每个 WORK_DIR **并行**执行。

#### Step 0c：确认并声明

输出启动信息：

普通模式：
```
正在启动自动修复流程，将调用 apm-code-review 进行扫描...
工作目录：{WORK_DIR}
```

worktree 模式：
```
正在启动自动修复流程（worktree 隔离模式）...
目标分支：{branch_name}
Worktree 目录：{WORK_DIR}
当前工作区不受影响，你可以继续在主目录工作。
```

---

### Step 1：第一轮扫描（调用 apm-code-review）

以**子任务方式**调用 apm-code-review，传入用户的原始参数。

**worktree 模式下**：告知 apm-code-review 使用 worktree 路径，所有 git/Read 操作在 WORK_DIR 下执行。

```typescript
task(
  category="deep",
  load_skills=["apm-code-review"],
  run_in_background=false,        // 同步等待，需要拿到报告路径
  description="第1轮扫描",
  prompt=`
    请对以下范围执行代码审查，输出报告文件，并确保报告末尾包含 AUTO-FIX LIST 区块。

    审查范围参数：{透传用户原始输入，如"未提交变更"/"feature/xxx 分支"/"最近2次提交"/文件列表}

    ${WORK_DIR !== 当前工作目录 ? `
    ⚠️ Worktree 隔离模式：
    - worktree 路径：{WORK_DIR}
    - 所有 git 命令使用 workdir={WORK_DIR} 或 git -C {WORK_DIR}
    - 所有文件读取路径拼接 {WORK_DIR}/ 前缀
    - 报告仍写入主仓库的 docs/ 目录
    ` : ''}

    完成后告知：
    1. 报告文件的完整路径
    2. AUTO-FIX LIST 中的条目数量
  `
)
```

子任务完成后：
- 记录报告文件路径（如 `docs/code-review-20260414-xxx.md`）
- 用 Read 工具读取该报告文件，定位 `## 5. AUTO-FIX LIST` 区块，解析其中的 JSON 数组

**路径处理**：AUTO-FIX LIST 中的文件路径可能是相对路径（相对于 WORK_DIR）。后续 Step 2 读取/修改文件时，需拼接为 `{WORK_DIR}/{relative_path}` 的绝对路径。

若 AUTO-FIX LIST 为空数组 `[]`，直接跳到 Step 4（worktree 模式会在 Step 4 中清理 worktree）：
```
✅ 扫描完成，无可自动修复的问题。
报告已在：{报告路径}
需人工处理的问题请查阅报告。
```

---

### Step 2：修复循环（最多执行 2 次）

**轮次计数器**从 1 开始。每轮执行以下操作：

#### Step 2a：解析本轮待修复内容

从当前报告的 `## 5. AUTO-FIX LIST` JSON 中，按文件分组，输出摘要：
```
第 N 轮修复：
  共 M 个文件，X 个问题需修复
  - path/to/File.java：3 个问题（@Autowired→@Resource, serialVersionUID, log.error补参数）
  - path/to/Entity.java：2 个问题（LocalDateTime TypeHandler ×2）
```

#### Step 2b：读取待修复文件的当前内容

用 Read 工具读取每个有 fixes 的文件完整内容，**在内存中保存原始内容**（用于回滚）。

**路径规则**：文件路径 = `{WORK_DIR}/{AUTO-FIX LIST 中的相对路径}`。普通模式下 WORK_DIR 就是当前目录，worktree 模式下是 worktree 绝对路径。

#### Step 2c：并行派发修复执行器

对每个有 fixes 的文件，派发一个修复子任务。**文件路径传绝对路径**（已拼接 WORK_DIR）：

```typescript
task(
  category="quick",
  load_skills=[],
  run_in_background=true,         // 并行执行，多文件同时修
  description="修复 {WORK_DIR}/path/to/File.java（第N轮）",
  prompt=FIXER_PROMPT             // 见下方「修复执行器 Prompt 模板」，文件路径使用绝对路径
)
```

等待所有文件修复子任务完成。

#### Step 2d：lsp_diagnostics 验证

对每个被修改的文件运行 `lsp_diagnostics`（传入绝对路径 `{WORK_DIR}/{relative_path}`），检查 error 级别：

- **有 error** → 用 Write 工具将 Step 2b 保存的原始内容写回（回滚），记入"回滚列表"
- **无 error** → 记入"成功列表"

#### Step 2e：判断是否继续

- 轮次 < 2 → 进入 Step 3（验证扫描）
- 轮次 = 2 → 直接进入 Step 4

---

### Step 3：验证扫描（再次调用 apm-code-review）

**只扫描本轮成功修改的文件**（不含回滚文件，不扩展范围）。

**worktree 模式下**：文件路径使用 WORK_DIR 内的绝对路径，并告知 apm-code-review worktree 上下文。

```typescript
task(
  category="deep",
  load_skills=["apm-code-review"],
  run_in_background=false,
  description="第N轮验证扫描",
  prompt=`
    请对以下指定文件执行代码审查，输出报告文件，确保包含 AUTO-FIX LIST 区块。

    指定文件列表（仅审查这些文件，不扩展范围）：
    - {WORK_DIR}/path/to/File.java
    - {WORK_DIR}/path/to/Entity.java

    ${worktree模式时追加}
    ⚠️ Worktree 隔离模式：
    - worktree 路径：{WORK_DIR}
    - 所有 git 命令使用 workdir={WORK_DIR} 或 git -C {WORK_DIR}
    - 文件路径已是绝对路径，直接使用
    - 报告仍写入主仓库的 docs/ 目录

    完成后告知报告文件路径和 AUTO-FIX LIST 条目数。
  `
)
```

读取新报告的 AUTO-FIX LIST：

- **为空** → 修复完成，进入 Step 4
- **仍有条目** → 轮次 +1：
  - 轮次 ≤ 2 → 回到 Step 2，继续修复新报告中的 AUTO-FIX LIST
  - 轮次 > 2 → 将残留条目记为"超出轮次限制，需人工处理"，进入 Step 4

---

### Step 4：汇总写报告 & worktree 清理

#### Step 4a：写报告

收集所有轮次信息，写入 `docs/auto-fix-<日期>-<简短主题>.md`（见下方报告模板）。

**报告始终写入主仓库的 `docs/` 目录**（不是 worktree 内），确保所有报告集中管理。

#### Step 4b：worktree 清理（仅 worktree 模式执行）

普通模式跳过此步骤。

```bash
# 1. 检查 worktree 内是否有未提交变更
git -C {WORK_DIR} status --porcelain
```

| worktree 状态 | 处理方式 |
|--------------|---------  |
| 工作区干净（无未提交变更） | 直接清理：`git worktree remove {WORK_DIR}` |
| 有未提交的修复变更 | 先提交：`git -C {WORK_DIR} add -A && git -C {WORK_DIR} commit -m "auto-fix: 自动修复规范问题"`，然后清理 |
| remove 失败（被占用等） | 提示用户手动清理：`git worktree remove --force {WORK_DIR}` |

清理完成后输出：
```
Worktree 已清理：{WORK_DIR}
修复变更已提交到分支 {branch_name}，可通过 git log {branch_name} 查看。
```

#### Step 4c：多分支并行模式的汇总

若同时修复多个分支，每个分支独立执行 Step 1-4b 后，在此汇总：

```
=== 多分支自动修复汇总 ===

| 分支 | 修复数 | 残留数 | 报告 |
|------|--------|--------|------|
| feature/branch-a | 12 | 2 | docs/auto-fix-xxx-branch-a.md |
| feature/branch-b | 8 | 0 | docs/auto-fix-xxx-branch-b.md |

所有 worktree 已清理。
```

---

## 修复执行器 Prompt 模板

````
你是 APM 后端项目的代码修复执行器。对指定文件应用明确的修复操作。

## 硬规则（绝对不能违反）

1. **只修改本文件**：不创建新文件，不读取或修改任何其他文件
2. **只修复列出的问题**：不做额外重构、格式化、风格调整、逻辑修改
3. **保留所有原有逻辑**：只做最小化改动
4. **使用 Edit 工具逐条修改**：每个 fix 单独一次 Edit 调用，不整体覆盖文件
5. **修复完成后运行 lsp_diagnostics**：发现 error 立即在输出中标注，不继续

## 待修复文件

文件路径：{文件绝对路径}（编排器已拼接 WORK_DIR，直接使用此路径读写文件）

## 问题清单

{从 AUTO-FIX LIST 提取的该文件 fixes 数组，JSON 原文}

每条 fix 包含：
- `line`：问题行号，用于定位
- `target`：目标位置描述
- `desc`：修复意图（核心依据）
- `action`：修复动作（`replace` / `insert_before` / `insert_after` / `append_annotation`）
- `before`：修复前代码片段（精确定位用）
- `after`：修复后代码片段

## 执行原则

1. **以 `desc` + `action` 为主要依据**：`desc` 描述了修复意图，`action` 说明了操作类型
2. **以 `before`/`after` 为精确定位**：若提供了 before/after，优先用其精确匹配，不要自行猜测修改范围
3. **`line` 仅作参考**：代码可能因之前的修复而行号偏移，先用 before 匹配，找不到再用行号定位
4. **每条 fix 独立执行**：逐条处理，中间遇到编译错误立即停止并上报

## 通用修复规范

### replace（替换）
用 Edit 工具将 `before` 内容替换为 `after` 内容，保持缩进和上下文不变。

### insert_before（前插入）
用 Edit 工具将 `after` 内容插入到 `before` 所在行的上方，对齐缩进。

### insert_after（后插入）
用 Edit 工具将 `after` 内容插入到 `before` 所在行的下方，对齐缩进。

### append_annotation（追加注解）
在目标字段或方法声明行的**上方**插入注解，对齐现有代码缩进。

## 修复完成后输出格式

```
=== 修复结果 ===

【文件：path/to/File.java】

已完成：
- 第 12 行（字段 xxxService）：@Autowired 替换为 @Resource
- 第 67 行（catch 块）：log.error 补充异常参数 e
- 类级别：插入 serialVersionUID = 1L

lsp_diagnostics 结果：✅ 无编译错误
```

若有编译错误：
```
lsp_diagnostics 结果：❌ 编译错误
错误内容：[具体错误信息]
⚠️ 请编排器回滚此文件
```

## 文件当前内容

{文件完整代码}
````

---

## 最终报告模板

写入 `docs/auto-fix-<日期>-<简短主题>.md`：

```markdown
# 自动修复报告

> 修复时间：{日期时间}
> 修复范围：{分支/提交/文件描述}
> Worktree：{WORK_DIR}（分支：{branch_name}）  <!-- worktree模式时填写，普通模式删除此行 -->
> 执行轮次：{实际执行轮次} / 2

## 1. 修复概述

| 项目 | 数量 |
|------|------|
| 扫描文件数 | N |
| 发现可自动修复问题数 | N |
| 实际修复问题数 | N |
| 修复失败已回滚文件数 | N |
| 残留未修复问题数（超出轮次限制） | N |

## 2. 已自动修复

| 文件 | 修复内容 | 轮次 |
|------|---------|------|
| `path/to/File.java` | @Autowired→@Resource ×2, serialVersionUID ×1 | 第1轮 |
| `path/to/Entity.java` | LocalDateTime TypeHandler ×1 | 第1轮 |

## 3. 需人工处理

> 以下问题来自 apm-code-review 报告，因存在业务歧义未自动修复。
> 详细描述见关联 review 报告。

### 🔴 P0 — 阻塞（合并前必须修复）
...

### 🟠 P1 — 严重
...

### 🟡 P2 / 🔵 P3
...

## 4. 修复失败（已回滚）

| 文件 | 编译错误信息 |
|------|------------|
| `path/to/File.java` | [错误内容] |

## 5. 残留可修复问题（超出2轮限制）

（如有，列出文件和问题，建议手动处理）

## 6. 关联 Review 报告

| 轮次 | 报告文件 |
|------|---------|
| 第1轮扫描 | `docs/code-review-20260414-xxx.md` |
| 第1轮验证 | `docs/code-review-20260414-yyy.md` |
| 第2轮验证 | `docs/code-review-20260414-zzz.md` |

## 7. 总结

**自动修复后残留问题**：[通过 ✅ / 需修改后通过 ⚠️ / 需重大修改 ❌]

**优先处理**：[最应优先人工修复的 2-3 个问题]
```

---

## 注意事项

- **不自己实现审查逻辑**：扫描完全依赖 apm-code-review，只解析其输出的 AUTO-FIX LIST
- **AUTO-FIX LIST 解析失败时**：报告中找不到该区块或 JSON 解析失败，告知用户并停止，不猜测
- **回滚方式**：Step 2b 读取文件时在内存保存原始内容，用 Write 工具原样写回
- **docs/ 不存在时**：先创建目录再写文件
- **文件过多时**（>20个）：apm-code-review 本身会询问，跟随其决定即可
- **Worktree 模式注意事项**：
  - **同一分支不能被两个 worktree 同时 checkout**：若目标分支已有 worktree，先 remove 旧的再创建新的
  - **worktree 目录命名**：统一为 `../apm-fix-{branch_short_name}`（与主仓库同级），避免嵌套在主仓库内
  - **报告写入位置**：始终写入主仓库的 `docs/`，不写入 worktree 内
  - **lsp_diagnostics 兼容性**：传入 worktree 内的绝对路径，LSP 应能正常解析（若不行则跳过 diagnostics 并在报告中注明）
  - **清理时机**：Step 4b 自动清理；若流程异常中断，用户可手动执行 `git worktree remove ../apm-fix-xxx`
  - **多分支并行**：每个分支独立 worktree + 独立 Step 1-4b 流程，最后在 Step 4c 汇总
