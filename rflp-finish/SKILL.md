---
name: RFLP 收尾
description: 验证完成度，合并 RFL 文档，处理分支
when_to_use: 当实现计划的所有 task 已执行完毕
version: 1.0.0
---

# RFLP 收尾

## 概述

确认所有工作完成，将 PRD 和技术方案的内容合并到正式 R/F/L 文档，然后处理分支。

**启动时声明：** "我正在使用 rflp-finish 收尾。"

## Step 0: 启动检查

**重新读取** TRD 文件（`docs/plans/*-trd.md`）和进度文件 `docs/plans/.progress.json`（若存在）——不依赖对话记忆。

**模式 C（worktree）检查：**
```bash
git worktree list
```
如果发现 `.worktrees/` 下仍有 feature worktree：
- 正常情况——模式 C 的 feature worktree 在 rflp-finish 阶段才合并和清理
- 检查 `.progress.json` 确认所有 task 均已完成（`completed_tasks` 包含全部 task）
- 未完成 → 停止，提示用户先回到 rflp-code 处理未完成的 task

## Step 1: 验证完成度

1. **确认任务清单**：待办清单全部 completed，无遗漏
2. **分层跑测试**：AT → FT → UT，确保全绿
   - 测试命令从技术方案获取，不硬编码
   - 如果项目只有部分层的测试，跑已有的即可

**测试失败时：**
```
测试未通过（N 个失败）。必须修复后才能继续：

[展示失败详情]

无法继续合并/PR。
```
停下。不进入 Step 2。

**Pre-existing failure：** 如果失败的测试与本次变更无关（本次未 touch 相关文件），必须：
1. 明确记录并展示给用户：
   ```
   ⚠️ 发现 pre-existing failure：
   - [测试名]：[错误信息]
   - 判断依据：本次未修改相关文件（[文件列表]）

   这不是本次引入的问题，但需要你确认后才能继续。
   输入 'confirm-preexisting' 确认知晓并继续。
   ```
2. 等用户输入 `confirm-preexisting` 后方可进入 Step 2
3. 不接受"忽略它""差不多就行"等模糊回复

## Step 2: 合并 RFL 文档

读技术方案的 PRD 引用，找到 PRD 文件。然后执行合并：

**从 PRD 合并 R 内容：**
- 将 PRD 的「R 内容」（场景 + AT 规格）合并到正式 R 文档（`docs/R层-需求/R0x.md`）
- 新项目：创建 `docs/R层-需求/` 目录 + 新文件
- 已有项目：更新已有文件或追加新场景

**从 PRD 合并 F 内容：**
- 将 PRD 的「F 内容」（命令参考 + 行为规则 + FT 规格）合并到正式 F 文档（`docs/F层-功能/F0x.md`）

**从技术方案合并 L 内容：**
- 将技术方案的「L 内容」（组件职责 + 接口定义 + UT 规格）合并到正式 L 文档（`docs/L层-逻辑/L0x.md`）

**逐文档检查一致性：**
- [ ] R 文档 AT 规格 ↔ `tests/acceptance/` 下的 AT 代码
- [ ] F 文档 FT 规格 ↔ `tests/functional/` 下的 FT 代码
- [ ] L 文档接口定义 ↔ 实际代码

**有遗漏 → 先补完再继续。** R 文档是权威源——发现文档与代码不一致时，补代码而非改文档。这是知识沉淀的最后关卡。

**CLAUDE.md 待办：** 如果本次完成了待办项，标记为完成。

**提交：**
```bash
git add docs/
git commit -m "docs: 合并 RFL — [功能名称]"
```

**清理 `.progress.json`：** RFL 合并提交成功后再删除，确保测试失败可回到 rflp-code 时断点仍有效：
```bash
rm docs/plans/.progress.json
```

## Step 3: 处理分支

**在 main 上直接工作时：** 跳过分支处理，直接报告完成。

否则，展示选项：

```
所有工作已完成。接下来怎么处理？

1. 本地合并到 <base-branch>
2. 推送并创建 Pull Request
3. 保留分支，稍后处理
4. 丢弃本次工作

选哪个？
```

### 选项 1: 本地合并

```bash
git checkout <base-branch>
git pull
# 模式 C：先 ff-only 合并 feature branch，再合并到 base
git merge --ff-only <feature-branch>  # 如 base branch 有新提交，先 rebase feature branch
# 合并后再跑一次测试
<test command>
# 测试通过后删除 feature branch 和 worktree
git branch -d <feature-branch>
git worktree remove .worktrees/<feature-name>  # 模式 C 特有
```

### 选项 2: 推送 + 创建 PR

```bash
git push -u origin <feature-branch>

gh pr create --title "<标题>" --body "$(cat <<'EOF'
## 变更摘要
- [要点 1]
- [要点 2]

## 测试计划
- [ ] [验证步骤]

## RFL 文档更新
- [列出本次更新的 R/F/L 文档]
EOF
)"
```

PR body **必须列出更新的 RFL 文档**。

### 选项 3: 保留

报告："保留分支 `<name>`，稍后处理。"

**注意：** 模式 C 的 feature worktree（`.worktrees/<feature-name>`）同样保留，不清理。

### 选项 4: 丢弃

**必须确认：**
```
这将永久删除：
- 分支 <name>
- 所有提交：<commit-list>

输入 'discard' 或 '确认丢弃' 确认。
```

等用户输入 `discard` 或 `确认丢弃` 后才执行。不接受模糊回复（如"好的删吧"）。

> **模式 C 说明：** 丢弃操作同时删除 feature branch 和 worktree（`.worktrees/<feature-name>`）。

## 快速参考

| 选项 | 合并 | 推送 | 删除分支 |
|------|------|------|----------|
| 1. 本地合并 | ✓ | - | ✓ |
| 2. 创建 PR | - | ✓ | - |
| 3. 保留 | - | - | - |
| 4. 丢弃 | - | - | ✓ (强制) |

## 红线

**绝不：**
- 测试没全绿就继续
- RFL 不一致就合并
- 不确认就丢弃工作
- 没有明确要求就 force-push

**始终：**
- 先测试，再展示选项
- 合并 RFL 文档到正式目录
- 丢弃前要求输入 'discard' 确认
- 用户选择前展示所有选项，不替用户做决定

## 记住

- 测试必须全绿
- RFL 合并是最后的知识沉淀关卡，不能跳过
- PR body 里要列出更新了哪些 RFL 文档
- 用户选择前先展示所有选项，不要替用户做决定
