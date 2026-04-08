# 模式 C: Worktree 执行

**适用场景：** 多需求并行开发——每个 feature 占一个独立 worktree，互不干扰。单个 feature 内所有 task 在同一 worktree 里串行执行，每个 task 提交一个独立 commit。

**前置条件：** 已有 feature branch（由 rflp-trd 交接时确认），记为 `FEATURE_BRANCH`，如 `feature/my-feature`。

---

## 启动：创建 feature worktree（仅执行一次）

```bash
# 基于当前分支（main/develop）创建 feature worktree
git worktree add .worktrees/<feature-name> -b <FEATURE_BRANCH>
```

更新 `.progress.json`：`worktree_path = ".worktrees/<feature-name>"`

> **命名建议：** worktree 目录名与 feature branch 保持一致，例如 `feature/user-auth` → `.worktrees/user-auth`

---

## 每个 task 的执行流程

> ⚠️ **每次开始新 task 前：确认你仍在同一个 feature worktree（`.worktrees/<feature-name>`）内操作。禁止为单个 task 创建新的 worktree。**

### C1: 在 worktree 内实现

派实现 agent，所有操作在 worktree 目录内进行：

```
task(category="deep", description="实现 Task N: [task name]", run_in_background=false,
     load_skills=[], prompt="
你负责实现 [trd-file] 中的 Task N。

⚠️ 重要：所有文件读写、git 操作必须在 worktree 目录内进行。
具体方式（二选一，以项目实际情况为准）：
- 方式一（推荐）：所有命令加 -C 前缀，例如：
    git -C /abs/path/to/.worktrees/<feature-name> add src/Foo.java
    git -C /abs/path/to/.worktrees/<feature-name> commit -m '...'
  文件路径使用 worktree 下的绝对路径读写
- 方式二：在 prompt 开头声明 '工作目录已切换至 .worktrees/<feature-name>'，
  所有相对路径均以该目录为根

禁止：在主仓库目录下执行任何 git add / git commit 操作。

执行步骤：
1. 重新读取 [trd-file]，找到 Task N 的完整内容
2. 严格按 task 里的每个 step 执行
3. 写测试（如果 task 要求 TDD）
4. 跑验证确认通过
5. 提交（仅此 task 的文件）：
   git -C .worktrees/<feature-name> add <Task N 文件列表中声明的具体路径>
   git -C .worktrees/<feature-name> commit -m 'feat/fix/chore: [Task N 描述]'
   禁止 git add .
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
