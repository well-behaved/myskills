# 模式 C: Worktree 执行

**适用场景：** 多需求并行开发——每个 feature 占一个独立 worktree，互不干扰。单个 feature 内所有 task 在同一 worktree 里串行执行，每个 task 提交一个独立 commit。

**前置条件：** 已有 feature branch（由 rflp-trd 交接时确认），记为 `FEATURE_BRANCH`，如 `feature/my-feature`。

---

## 启动：创建 feature worktree（仅执行一次）

> ⛔ **强制检查点（禁止跳过）：在任何文件读写或 git 操作之前，必须先完成以下步骤，确认 worktree 已就绪。**
> **严禁在主仓库目录（非 `.worktrees/` 路径）下直接修改文件或执行 git add/commit。**

> ⚠️ **worktree 必须放在主仓库的 `.worktrees/` 子目录下**，不要放到主仓库同级目录（如 `../worktree-xxx`），否则 IDE 打开主项目时看不到该目录。

```bash
# 先确认主仓库当前分支（不能是 feature 分支，否则 worktree add 会冲突）
git branch --show-current
# 如果当前在 feature 分支，先切回主分支：
# git checkout <base-branch>

# 情况 A：feature 分支已存在（rflp-trd 已建好）→ 不加 -b
git worktree add .worktrees/<feature-name> <FEATURE_BRANCH>

# 情况 B：feature 分支尚不存在 → 加 -b 新建
git worktree add .worktrees/<feature-name> -b <FEATURE_BRANCH>

# 示例：
# git worktree add .worktrees/pda-inventory-query feature/xue/pda-inventory-query
```

更新 `.progress.json`：`worktree_path = ".worktrees/<feature-name>"`（相对路径，不用绝对路径）

> **命名建议：** worktree 目录名与 feature branch 保持一致，例如 `feature/user-auth` → `.worktrees/user-auth`
>
> **⚠️ 常见错误：** rflp-trd 阶段已用 `git checkout -b <FEATURE_BRANCH>` 建好分支时，主仓库 HEAD 会切到该分支。必须先 `git checkout <base-branch>` 切回，再执行 `git worktree add`（情况 A），否则报错 "is already used by worktree"。

---

## 每个 task 的执行流程

> ⚠️ **每次开始新 task 前：确认你仍在同一个 feature worktree（`.worktrees/<feature-name>`）内操作。禁止为单个 task 创建新的 worktree。**

### C1: 在 worktree 内实现

派实现 agent，所有操作在 worktree 目录内进行：

```
task(category="deep", description="实现 Task N: [task name]", run_in_background=false,
     load_skills=[], prompt="
你负责实现 [trd-file] 中的 Task N。

⚠️ 重要：所有文件读写、git 操作必须使用 worktree 的【绝对路径】，严禁使用相对路径。
worktree 绝对路径：<WORKTREE_ABS_PATH>（例如 D:\myworkplace\proj\.worktrees\feature-x）

文件读写规则：
- 读文件：路径必须是 <WORKTREE_ABS_PATH>/src/... 形式的绝对路径
- 写文件：路径必须是 <WORKTREE_ABS_PATH>/src/... 形式的绝对路径
- 禁止：使用主仓库路径或任何相对路径读写文件

git 操作规则：
- 所有 git 命令必须加 -C <WORKTREE_ABS_PATH> 前缀，例如：
    git -C <WORKTREE_ABS_PATH> add src/Foo.java
    git -C <WORKTREE_ABS_PATH> commit -m '...'
    git -C <WORKTREE_ABS_PATH> status
- 禁止：在主仓库目录下执行任何 git add / git commit 操作
- 禁止：git add .（必须精确指定文件路径）

自检（每次 git add 前必做）：
  运行 git -C <WORKTREE_ABS_PATH> status --short
  确认变更文件全部在 <WORKTREE_ABS_PATH> 内，没有主仓库文件被误改

执行步骤：
1. 重新读取 [trd-file]（使用主仓库绝对路径，TRD 文件在主仓库 docs/plans/ 下），找到 Task N 的完整内容
2. 严格按 task 里的每个 step 执行，所有源码文件使用 worktree 绝对路径读写
3. 写测试（如果 task 要求 TDD）
4. 跑验证：mvn compile/test 命令的工作目录必须是 <WORKTREE_ABS_PATH>
5. 提交（仅此 task 的文件）：
   git -C <WORKTREE_ABS_PATH> add <精确文件路径列表>
   git -C <WORKTREE_ABS_PATH> commit -m 'feat/fix/chore: [Task N 描述]'
6. 报告：实现了什么、测试结果、改了哪些文件、遇到的问题
")
```

### C2: 验证 worktree 内有新 commit

```bash
# 获取 worktree 最新 commit SHA
git -C .worktrees/<feature-name> rev-parse HEAD

# 对比 .progress.json 的 last_commit_sha
# 如果两者相同 → worktree 内没有新 commit，实现未完成 → 回 C1 重新派实现 agent
# 检查工作区干净
git -C .worktrees/<feature-name> status --porcelain  # 必须为空
```

### C2.5: 编译检查

在 worktree 目录内执行编译：

```bash
mvn -B -DskipTests compile -f <WORKTREE_ABS_PATH>/pom.xml
```

- **BUILD SUCCESS** → 继续 C3
- **BUILD FAILURE** → 在 worktree 内派 fix agent 修复编译错误，修复提交后重新跑编译，通过后再进 C3：

```
task(category="quick", description="Fix compile error after Task N", run_in_background=false,
     load_skills=[], prompt="
编译失败，错误信息如下：
[粘贴 mvn 输出的 ERROR 部分]

在 worktree 内修复编译错误。只修错误，不做任何额外改动。
worktree 绝对路径：<WORKTREE_ABS_PATH>
所有文件读写和 git 操作使用 worktree 绝对路径。
修复后运行 mvn -B -DskipTests compile -f <WORKTREE_ABS_PATH>/pom.xml 确认 BUILD SUCCESS。
精确路径提交：git -C <WORKTREE_ABS_PATH> add <文件> && git -C <WORKTREE_ABS_PATH> commit -m 'fix: 修复 Task N 编译错误'
")

### C3: 派 review agent

```
task(category="deep", description="Review Task N: [task name]", run_in_background=false,
     load_skills=[], prompt="
审查 worktree .worktrees/<feature-name> 中 Task N 的变更。

**第一步：文件范围校验**
运行：git -C .worktrees/<feature-name> diff HEAD~1 HEAD --name-only
对比 [trd-file] Task N 中 **文件：** 列表声明的路径。
- 有未声明的文件被修改 → [Critical] 提交混入计划外文件，必须拆分
- 有声明的文件未被修改 → [Important] 未完成实现

**第二步：实现质量检查**
运行：git -C .worktrees/<feature-name> diff HEAD~1 HEAD
检查：
- 是否完整实现了 task 要求？
- 代码质量：命名、结构、错误处理
- 安全：有无注入、硬编码密钥等
- 测试覆盖是否充分

输出：
**issues:**
- [Critical] [描述] — 必须修
- [Important] [描述] — 应该修
- [Minor] [描述] — 建议修
**assessment:** PASS / NEEDS_FIX
")
```

### C4: 更新进度

PASS 后更新 `.progress.json`：`last_commit_sha` 更新为当前 HEAD，`current_task` 递增，`completed_tasks` 追加当前 task 编号。

```
Task N 完成，已提交到 .worktrees/<feature-name>（branch: <FEATURE_BRANCH>）。
改动文件：[列出] — 测试：PASS

⚠️ 上下文提示：如已执行较多 task，建议新开 session 后用 /rflp-code 继续，
进度已保存至 docs/plans/.progress.json。
```

---

## NEEDS_FIX 处理

在同一 worktree 内派 fix agent 修复（继续在 `.worktrees/<feature-name>` 内操作），修复后重新 review（回 C3）。

---

## 全部 task 完成后：合并回主分支 + 清理 worktree

所有 task 完成后（final review PASS），在 rflp-finish 阶段执行：

```bash
# 回到主仓库目录（sprint branch / main）
git checkout <base-branch>

# ff-only 合并 feature branch（保留每个 task 的 commit）
git merge --ff-only <FEATURE_BRANCH>

# 清理 worktree 和分支（如不再需要独立分支）
git worktree remove .worktrees/<feature-name>
# 分支保留由 rflp-finish Step 3 决定（推 PR 或本地合并）
```

> **注意：** `git merge --ff-only` 要求 feature branch 上无分叉。如果 base branch 有新提交，先 rebase：
> ```bash
> git -C .worktrees/<feature-name> rebase <base-branch>
> ```

---

## 断点恢复

恢复时读 `.progress.json` 的 `worktree_path` 字段，验证 worktree 是否还存在：

```bash
git worktree list
```

| 状态 | 处理方式 |
|------|---------|
| worktree 存在，有未完成 task | 直接继续，从 `current_task` 开始 |
| worktree 不存在（意外删除） | 重新创建：`git worktree add .worktrees/<feature-name> <FEATURE_BRANCH>` |
| 所有 task 已完成 | 进入 final review → rflp-finish |

---

## 注意事项

- **不并行派多个实现 agent**——单 worktree 串行执行，不存在物理隔离
- worktree 内每个 task 独立 commit，保持 commit 历史清晰
- 多个需求并行时：每个需求各自有独立 worktree（`.worktrees/feature-a`、`.worktrees/feature-b`），互不干扰
