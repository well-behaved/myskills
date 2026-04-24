# 模式 B: 子 Agent 执行

每个 task 按"实现 → review → fix"循环执行：

## B1: 记录起始 SHA

```bash
git rev-parse HEAD  # 记为 BASE_SHA
```

## B2: 派实现 agent

```
task(category="deep", description="实现 Task N: [task name]", run_in_background=false,
     load_skills=[], prompt="
你负责实现 [trd-file] 中的 Task N。
1. 重新读取该技术方案文件，找到 Task N 的完整内容
2. 严格按 task 里的每个 step 执行
3. 写测试（如果 task 要求 TDD）
4. 跑验证确认通过
5. 提交时只 git add Task N 的 **文件：** 列表中声明的具体路径，禁止 git add .
6. 报告：实现了什么、测试结果、改了哪些文件、遇到的问题
")
```

## B3: 记录结束 SHA + 确认工作区干净

```bash
git rev-parse HEAD  # 记为 HEAD_SHA
git status --porcelain  # 必须为空
```

## B3.5: 编译检查

```bash
mvn -B -DskipTests compile
```

- **BUILD SUCCESS** → 继续 B4
- **BUILD FAILURE** → 派 fix agent 修复编译错误，修复提交后重新跑编译，通过后再进 B4：

```
task(category="quick", description="Fix compile error after Task N", run_in_background=false,
     load_skills=[], prompt="
编译失败，错误信息如下：
[粘贴 mvn 输出的 ERROR 部分]

修复编译错误。只修错误，不做任何额外改动。
修复后运行 mvn -B -DskipTests compile 确认 BUILD SUCCESS。
精确路径提交：git add <文件> && git commit -m 'fix: 修复 Task N 编译错误'
")
```

## B4: 派 review agent

```
task(category="deep", description="Review Task N: [task name]", run_in_background=false,
     load_skills=[], prompt="
审查 [BASE_SHA] 到 [HEAD_SHA] 之间的变更。
用 git diff [BASE_SHA] [HEAD_SHA] 查看。
对照 [trd-file] 中的 Task N 检查：

**文件范围校验（必须首先检查）：**
运行：git diff [BASE_SHA] [HEAD_SHA] --name-only
对比 Task N 中 **文件：** 列表声明的路径。
- 有未声明的文件被修改 → [Critical] 提交混入计划外文件，必须拆分
- 有声明的文件未被修改 → [Important] 未完成实现

**然后检查：**
- 是否完整实现了 task 要求？
- 代码质量：命名、结构、错误处理
- 安全：有无注入、硬编码密钥等
- 测试覆盖

输出：
**issues:**
- [Critical] [描述] — 必须修
- [Important] [描述] — 应该修
- [Minor] [描述] — 建议修
**assessment:** PASS / NEEDS_FIX
")
```

## B5: 处理 review 反馈

- PASS → 更新 `.progress.json`，标记 task 完成，进入下一个
- NEEDS_FIX：
  - Critical → 立即派 fix agent
  - Important → 下个 task 前派 fix agent
  - fix 完成后重新 review（回到 B3）

**不手动修子 agent 的产出**——派新 agent 修。（唯一例外：< 5 行纯机械修改如变量重命名，可由 leader 直接修，但 commit message 须标注 `[leader-fix]`）

**不并行派多个实现 agent**——git 冲突风险。
