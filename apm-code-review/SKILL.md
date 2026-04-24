---
name: apm-code-review
description: |-
  APM 后端项目代码审查编排器。自动确定审查范围、分批派发给审查执行器、汇总输出报告文件。
  支持：当前未提交变更、指定分支 diff、最近 N 次提交、指定文件列表、git worktree 独立目录。

  使用场景：
  - user: "review 一下我刚写的代码" → 自动拉取未提交变更，执行审查
  - user: "帮我 review feature/xxx 分支" → diff 该分支与 master，执行审查
  - user: "review 最近 3 次提交" → 拉取最近 3 次提交变更，执行审查
  - user: "review 这几个文件 A.java B.java" → 直接对指定文件执行审查
  - user: "检查这个 PR 是否符合规范" → 按 checklist 逐项审查并输出报告
  - user: "review worktree /path/to/worktree 的变更" → 在指定 worktree 目录下执行审查
---

# APM 代码审查编排器

你是审查流程的**编排者**，不是审查的执行者。职责是：解析参数 → 确定范围 → 读取文件内容 → 分批派发给审查执行器 → 汇总结果 → 写入报告文件。

## 项目背景

apm-backend 是 **Spring Cloud 微服务架构**的工厂 APM 系统后端。
技术栈：Java 1.8 / Spring Boot 2.7.8 / MyBatis-Plus 3.5.3.1 / Sa-Token / PostgreSQL（多 schema）

---

## 编排流程

### Step 0：声明启动

输出：`"正在解析审查参数，确定审查范围..."`

### Step 1：解析参数，确定审查范围

从用户输入中提取审查模式，执行对应 git 命令：

#### 模式 A：未提交变更（默认，无参数时）
```bash
git diff HEAD --name-only
git diff --cached --name-only
```

#### 模式 B：指定分支与 master 的差异
```bash
git diff master...<branch> --name-only
git log master...<branch> --oneline --no-merges
```

#### 模式 C：最近 N 次提交（如"最近3次提交"）
```bash
git log -N --oneline --no-merges
git diff HEAD~N HEAD --name-only
```

#### 模式 D：用户直接指定文件列表
跳过 git 命令，直接使用用户提供的文件路径。

**过滤规则**（所有模式通用）：排除以下文件：
- `**/test/**`（测试代码）
- `*.md`、`*.yml`、`*.properties`（配置/文档）

**XML 处理规则**（所有模式通用）：`*Mapper.xml` 不单独派发，与对应 `*Mapper.java` 打包成同一批次一起审查。

收集完毕后，输出范围摘要：
```
审查模式：[模式名称]
变更文件数：N
文件列表：
  - path/to/File1.java  [Controller]
  - path/to/File2.java  [ServiceImpl]
  - path/to/File3Mapper.java + File3Mapper.xml  [Mapper]
```

#### 模式 E：git worktree 模式
当用户指定 `--worktree <path>` 或说"在 worktree xxx 中"/"review worktree xxx"时启用。
可与子模式组合，如"review worktree /path 最近2次提交" → 模式 E + 模式 C。

**Step E-1：检测 worktree 根目录**
```bash
# 列出所有 worktree，确认目标路径有效
git worktree list
```

**Step E-2：确定 worktree 路径**
- 用户可传入绝对路径（如 `/workspace/feature-xxx`）或相对路径
- 若路径不在 `git worktree list` 输出中，提示用户并终止，不要猜测路径

**Step E-2.5：⚠️ 强制确认当前分支（防止改错仓库）**

```bash
git -C <worktree_path> branch --show-current
```

输出当前所在分支，向用户明确告知：

> `当前 worktree 路径：<worktree_path>`
> `所在分支：<branch-name>`
> `后续所有文件读取均来自该 worktree，确认继续？（直接回车或说继续）`

- 若用户确认 → 继续 Step E-3
- 若用户说"不对"/"换一个" → 重新提示输入正确路径，返回 Step E-1
- **编排器本身不做任何文件写入操作**，只读取文件内容交给执行器审查，无需担心写错位置；但此确认步骤的目的是：**让用户在审查开始前明确知道审查的是哪个分支的代码**，避免看了错误分支的报告

**Step E-3：在指定 worktree 下执行 git 命令**

所有后续 git 命令使用 `workdir=<worktree_path>` 或 `-C` 前缀：

```bash
# 方式一：通过 workdir 参数（推荐）
# Bash(command="git diff HEAD --name-only", workdir="<worktree_path>")

# 方式二：显式指定 git 目录（worktree 的 .git 是文件而非目录，自动解析）
git -C <worktree_path> diff HEAD --name-only
git -C <worktree_path> diff --cached --name-only
```

子模式选择规则：
- 用户未指定子模式 → 默认模式 A（未提交变更）
- 用户指定"最近 N 次提交" → 模式 C，git 命令加 `-C <worktree_path>`
- 用户指定分支 → 模式 B，git 命令加 `-C <worktree_path>`

**Step E-4：文件路径处理**
- git 返回的路径是相对于 worktree 根目录的相对路径
- 读取文件（Step 2）时必须拼接为 `<worktree_path>/<relative_path>`
- 报告中文件路径统一显示为含 worktree 根目录的完整绝对路径

**Step E-5：报告写入位置**
- 报告仍写入**主仓库**的 `docs/` 目录（即当前 session 的 workdir）
- 执行器 prompt 中 `{模式}` 占位符填写：`worktree(<worktree_path>) + <子模式名>`，如 `worktree(/workspace/feature-xxx) + 未提交变更`

### Step 2：读取文件内容

使用 Read 工具读取每个 `.java` 文件的完整内容。
如果是 Mapper.java，同时读取同目录下的 `*Mapper.xml`（如存在）。

### Step 2.5：读取审查规范

使用 Read 工具读取以下文件，内容将在 Step 3 拼入执行器 Prompt：

```
.claude/skills/apm-code-review/references/checklist.md
```

> 此文件是规则的**唯一权威来源**，包含完整检查清单、最高频违规列表、输出格式。

### Step 3：分批派发给审查执行器

**分批规则**：每批最多 5 个文件（Mapper.java + Mapper.xml 算 1 个），并行执行。

对每批文件，使用 `task` 工具派发：

```typescript
task(
  subagent_type="explore",
  run_in_background=true,
  load_skills=[],          // 必须为空数组，避免子 agent 重新进入编排模式
  description="审查批次 N/M",
  prompt=REVIEWER_PROMPT   // 见下方「执行器 Prompt 模板」
)
```

等待所有批次完成后进入 Step 4。

---

## 执行器 Prompt 模板

每次 task() 调用时，将以下内容作为 prompt，并将 `{...}` 占位符替换为实际值：

> **编排器构建 prompt 步骤**：
> 1. 将 Step 2.5 读取到的 `checklist.md` 全文替换 `{CHECKLIST_CONTENT}`
> 2. 替换批次信息占位符
> 3. 填入文件路径列表和文件内容

````
你是 APM 后端项目的代码审查执行器。按以下规范对代码进行逐项检查，输出结构化问题列表。

{CHECKLIST_CONTENT}

---

## 本批次待审查文件

审查模式：{模式}
本批文件（{当前批次}/{总批次}）：

{文件路径列表}

---

## 文件内容

{每个文件的完整代码（Java + XML）}
````

---

### Step 4：汇总结果，写入报告文件

收集所有批次的审查结果，合并为完整报告，写入文件：

**文件路径**：`docs/code-review-<日期>-<简短主题>.md`

**报告结构**：

```markdown
# 代码审查报告

> 审查时间：{日期时间}
> 审查模式：{模式}
> 审查范围：{分支/提交描述}
> Worktree：{worktree路径}（分支：{branch-name}）  <!-- worktree模式时填写，普通模式删除此行 -->

## 1. 变更概述

| 文件 | 类型 | 变更说明 |
|------|------|---------|
| ... | Controller/Service/... | 新增/修改 |

## 2. 审查发现

> 问题总数：N（🔴 P0: x / 🟠 P1: x / 🟡 P2: x / 🔵 P3: x）

### 🔴 P0 — 阻塞（合并前必须修复）

#### P0-1：[问题标题]
- **文件**：`path/to/File.java` 第 N 行，方法 `methodName()`
- **现象**：[代码层面的现象描述]
- **影响**：[潜在风险]
- **修复**：
  ```java
  // 修复后代码
  ```

### 🟠 P1 — 严重（建议修复，可协商排期）

...（同上格式）

### 🟡 P2 — 规范（建议修复）

...

### 🔵 P3 — 建议（留待迭代优化）

...

## 3. 审查总结

| 级别 | 数量 | 处理要求 |
|------|------|---------|
| 🔴 P0 | x | 合并前必须修复 |
| 🟠 P1 | x | 建议修复，可协商排期 |
| 🟡 P2 | x | 建议修复 |
| 🔵 P3 | x | 留待迭代优化 |

**总体评价**：[通过 ✅ / 需修改后通过 ⚠️ / 需重大修改 ❌]

**优先处理**：[最应优先修复的 2-3 个问题]

## 4. AUTO-FIX LIST

> 此区块供 apm-auto-fix 机器解析，人工阅读可忽略。
> 只列出规则明确、无业务歧义、可安全自动修复的问题。

```json
[
  {
    "file": "path/to/File.java",
    "fixes": [
      {
        "id": "F01",
        "line": 12,
        "target": "字段 xxxService",
        "desc": "@Autowired 替换为 @Resource"
      },
      {
        "id": "F02",
        "line": 67,
        "target": "catch 块 log.error(\"加载失败\")",
        "desc": "补充异常参数 e"
      }
    ]
  }
]
```

**AUTO-FIXABLE 问题编号对照表（只有这些编号才能出现在此列表）：**

| ID | 问题类型 | 级别 |
|----|---------|------|
| F01 | `@Autowired` → `@Resource` | P1 |
| F02 | `log.error` 漏传异常对象 `e` | P2 |
| F03 | Entity/DTO 缺 `serialVersionUID` | P2 |
| F04 | `LocalDateTime` 字段缺 `@TableField(typeHandler=TimestampTypeHandler.class)` | P1 |
| F05 | 类缺 `@author` + `@since` 注释 | P2 |
| F06 | public 方法缺 Javadoc | P2 |
| F07 | SaveParam 必填字段缺 `@NotBlank`/`@NotNull` 模板 | P2 |
| F08 | public 方法体超过 50 行，需机械拆分（提取数据准备/映射/组装逻辑为私有方法） | P2 |

**不得出现在此列表的问题（有业务歧义）：** P0 全部、事务方法名、权限注解、循环内查询、方法签名、基类继承、Controller 业务逻辑、FindParam `@Query` 注解、Feign Fallback。

如本次审查无可自动修复问题，输出空数组：`[]`
```
```

### Step 5：告知结果

完成后输出：
1. 报告文件路径
2. P0/P1/P2/P3 数量摘要
3. 如有 P0 问题，高亮提醒需合并前修复
4. 如有 AUTO-FIX LIST 条目，提示：`"发现 N 个可自动修复的问题，可运行 apm-auto-fix 自动处理"`

---

## 注意事项

- **不要自己做审查**——代码规范检查全部由审查执行器完成，编排器只负责调度
- **不要在对话里输出完整报告**——报告写文件，对话里只输出摘要
- **文件过多时**（>20个）：先告知用户文件数量，询问是否全量审查还是只审查核心文件
- **docs/ 目录不存在时**：先创建目录再写文件
- **worktree 模式下**：
  - ⚠️ **必须在 Step E-2.5 确认分支后才能继续**，未确认分支禁止进入 Step E-3，这是防止审查错误分支代码的强制关卡
  - 所有 git/Read 操作使用 worktree 路径；报告仍写入当前 session workdir 的 `docs/`
  - 若 worktree 路径无效（不在 `git worktree list` 中），立即提示并终止，不要猜测路径
  - 文件路径在报告中显示为绝对路径或带 worktree 前缀，便于定位
  - 报告标题注明 worktree 路径和分支名，明确标识审查范围
