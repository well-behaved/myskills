---
name: RFLP 技术方案
description: 从确认的 PRD 到技术方案（L 内容 + UT 规格 + 实现计划），含任务清单
when_to_use: 当 PRD 已确认（打分 ≥ 9），需要编写技术方案和实现计划时
version: 1.0.0
---

# RFLP 技术方案

## 概述

把确认的 PRD 拆成技术方案（L 内容 + 实现计划），每个 task 粒度 2-5 分钟，假设执行者对代码库零上下文。

技术方案保存在 `docs/plans/`，**不动正式 L 文档**。正式文档在 rflp-finish 验证通过后才合并。

**启动时声明：** "我正在使用 rflp-trd 编写技术方案。"

**输入：** `docs/plans/YYYY-MM-DD-<feature>-prd.md`（上一步产出）

**输出：** `docs/plans/YYYY-MM-DD-<feature>-trd.md`（技术方案）
**DDL 输出（如有）：** `docs/plans/YYYY-MM-DD-<feature>-ddl.sql`（数据库变更脚本，与 TRD 同名族）

**无 PRD 文件时不执行**（快速通道不进入此 skill）。

## Step 1: 审计现有状态

1. 读 PRD 的「变更类型」，确定默认 task 模板（每个 task 按实际性质选模板，同一 PRD 可混合使用）：
   - **功能** → 默认模板 A（TDD：写失败测试 → 实现 → 通过）
   - **重构** → 默认模板 C（绿灯下改造：基线全绿 → 重构 → 确认仍绿）
   - **接线/配置** → 默认模板 B（修改 → 冒烟测试 → 提交）
2. 检查代码中已有的实现和测试，避免重复工作
3. 读 `docs/archive/` 历史设计文档，提炼可复用的结构
4. **扫描 PRD 是否涉及数据库变更**（CREATE TABLE / ALTER TABLE / 新增列/索引等）：
   - 有 → 在 Step 3 中单独抽出 DDL task，并生成 `docs/plans/YYYY-MM-DD-<feature>-ddl.sql`
   - 无 → 跳过 DDL 相关步骤

## Step 2: 写 L 内容

定义变更后的组件职责和接口：

```markdown
### 组件 A: [名称]
**职责：** [一句话]

**接口：**
| 方法/端点 | 协议 | 入参 | 出参 | 约束 |

**UT 规格：**
| UT | 验什么 | 输入 | 期望 |
```

**要求：**
- 接口定义 > 架构图。接口是"合同"——协议、入参、出参、约束必须明确
- 从 archive/ 历史设计文档提炼结构，不重新发明

## Step 3: 写实现计划

### 粒度与测试

- 每个 task 粒度 **2-5 分钟**，单一动作（强依赖的两个小动作可合并，但须注明合并理由）
- 合并条件（必须全部满足）：
  1. 两个动作有**强依赖**——A 的产出是 B 的直接输入，单独测 A 无意义
  2. 合并后粒度 **≤ 10 分钟**
  3. 在 task 描述中**显式标注**：`[合并理由]: <具体原因>`
  - 不允许以"上下文连贯""拆太碎"为由合并——这些是效率偏好，不是强依赖
- 每个 task 的测试必须标明**层次**（AT / FT / UT）和**来源**（哪个规格）
- 测试代码从规格派生：**AT ← R 文档，FT ← F 文档，UT ← L 内容**
- 1 场景 = 1 AT 测试，验全部步骤；1 行为规则 = 1 FT 测试；1 接口约束 = 1 UT 测试
- 路径精确，逻辑关键代码完整（含条件分支、错误处理、边界判断的都算关键），命令可运行。不写"添加验证逻辑"这种模糊描述

### 分组

- 超过 10 个 task → 按阶段（Phase）分组，每个 Phase 是一个 review 检查点
- task 间有依赖时标注"依赖 Task N"

### Task 模板 A: TDD（新功能/修复）

```markdown
### Task N: [名称]
> [上下文]
> 测试层次: AT / FT / UT（来源: R文档场景X / F文档规则Y / L接口约束Z）

**文件：**
- 新建/修改：`exact/path/to/file`
- 测试：`tests/acceptance/R0x/test_xxx.py`（AT）

**Step 1: 写失败测试**
[具体测试代码]

**Step 2: 跑测试确认失败**
运行：[具体命令]
预期：FAIL

**Step 3: 写最小实现**
[具体实现代码]

**Step 4: 跑测试确认通过**
运行：[同 Step 2]
预期：PASS

**Step 5: 提交**
git add src/path/to/FileA.java src/path/to/FileB.java tests/path/to/TestFile.java && git commit -m "feat: [描述]"
# 必须列出 **文件：** 里声明的每个具体路径，禁止使用 git add .
```

### Task 模板 B: 接线（配置/脚手架）

```markdown
### Task N: [名称]
> [上下文]

**文件：**
- 修改：`exact/path/to/file`

**Step 1: 修改**
[具体改动]

**Step 2: 冒烟测试**
运行：[命令]
预期：[结果]

**Step 3: 提交**
git add exact/path/to/file && git commit -m "chore: [描述]"
# 必须列出 **文件：** 里声明的每个具体路径，禁止使用 git add .
```

### Task 模板 C: 重构

```markdown
### Task N: [名称]
> [上下文]

**文件：**
- 修改：`exact/path/to/file`
- 基线测试：[AT/FT/UT 列表]

**Step 1: 跑基线测试确认全绿**
运行：[命令——AT + FT + UT]
预期：PASS

**Step 2: 重构**
[具体改动]

**Step 3: 跑测试确认仍绿**
运行：[同 Step 1]
预期：PASS

**Step 4: 提交**
git add exact/path/to/file && git commit -m "refactor: [描述]"
# 必须列出 **文件：** 里声明的每个具体路径，禁止使用 git add .
```

### 尾部 task（必须）

最后一个 task 是**验证 RFL 文档与代码的一致性**：
- R 文档 AT 规格 ↔ tests/acceptance/ 下的 AT 代码
- F 文档 FT 规格 ↔ tests/functional/ 下的 FT 代码
- L 文档接口定义 ↔ 实际代码

如果实现过程发现 RFL 需要修正，在对应 task 中标注。

### DDL 规范（涉及数据库变更时）

**何时生成 DDL 文件：** PRD 中出现 CREATE TABLE / ALTER TABLE / 新增列 / 新增索引等数据库变更时。
**注意：** CREATE INDEX / DROP INDEX 与 ALTER TABLE / CREATE TABLE **同等对待**，一律放进 DDL 文件，不因"较小"而内联。

**文件位置：** `docs/plans/YYYY-MM-DD-<feature>-ddl.sql`（与 TRD 同名族）

**文件格式：**
```sql
-- Feature: [功能名称]
-- TRD: docs/plans/YYYY-MM-DD-<feature>-trd.md
-- Date: YYYY-MM-DD

-- Task N: [task 名称]
CREATE TABLE xxx (
    id BIGINT PRIMARY KEY,
    ...
);

-- Task M: [task 名称]
ALTER TABLE yyy ADD COLUMN zzz VARCHAR(255) NOT NULL DEFAULT '';
```

**规则：**
- 每段 SQL 前注释标注来源 task（`-- Task N: [名称]`）
- TRD 对应的 task 中**只写引用**，不内联 SQL：
  ```
  执行 DDL：docs/plans/YYYY-MM-DD-<feature>-ddl.sql（Task N 部分）
  ```
- DDL 文件仅用于 review 和 DBA 审批，最终迁移脚本由开发者按项目规范（Flyway/Liquibase/手动）处理
- 不与 `src/main/resources/db/migration/` 等迁移目录冲突——`docs/` 里是"待审批草稿"

### 底部常驻任务清单

```markdown
---

## 任务清单（底部常驻）

- [ ] Task 1: [名称]
- [ ] Task 2: [名称]
- ...
- [ ] Task N: 验证 RFL 一致性
```

## Step 4: 确认与交接

1. 技术方案保存到 `docs/plans/YYYY-MM-DD-<feature>-trd.md`，不动正式 L 文档
2. **如有 DDL**：保存到 `docs/plans/YYYY-MM-DD-<feature>-ddl.sql`，并在 TRD 中对应 task 内替换为引用
3. 将实现计划的每个 task 建为**待办清单项**（使用当前环境的任务管理工具，如 todowrite）
4. 提供三种执行模式选择：

**"技术方案已保存。三种执行方式：**
**1. 批量执行（模式 A）** — 按 Phase 或每 3 个 task 一批，批间暂停等 review
**2. 子 Agent 执行（模式 B）** — 每个 task 派独立 agent 实现 + review，严格 SHA 隔离
**3. Worktree 执行（模式 C）** — 整个 feature 一个 git worktree，在其中串行执行所有 task，适合多需求并行开发（每个需求占一个 worktree）
**选哪个？"**

5. 用户选择后：
   - 模式 A/B → 声明："技术方案已保存到 `docs/plans/YYYY-MM-DD-xxx-trd.md`。切到 rflp-code，模式 A/B。"
   - 模式 C → 先确认 feature worktree 分支：
     ```
     模式 C 需要一个 feature 分支作为本次需求的 worktree 基础。
     请确认：
     1. 已有分支（输入分支名，如 feature/my-feature）
     2. 现在创建（我来执行 git checkout -b feature/my-feature）
     ```
     用户确认后执行（如需创建）：
     ```bash
     git checkout -b <feature-branch>
     ```
     然后**立即创建 worktree**，放在主仓库的 `.worktrees/` 子目录下（这样 IDE 打开主项目时可见）：
     ```bash
     # 确认当前分支不是 feature 分支，否则先切回主分支
     git branch --show-current
     # 创建 worktree（统一放 .worktrees/ 下，不要放到主仓库外部）
     git worktree add .worktrees/<feature-name> <feature-branch>
     # 示例：git worktree add .worktrees/pda-inventory-query feature/xue/pda-inventory-query
     ```
     > ⚠️ worktree **必须放在 `.worktrees/` 下**，不要放到主仓库同级目录（如 `../worktree-xxx`），否则 IDE 里看不到。
     
     再声明："技术方案已保存到 `docs/plans/YYYY-MM-DD-xxx-trd.md`。feature branch：`<feature-branch>`，worktree：`.worktrees/<feature-name>`。切到 rflp-code，模式 C。"

## 记住

- 接口定义 > 架构图，接口是"合同"
- 路径必须精确，代码必须完整（逻辑关键部分），命令必须可运行
- 不写"添加验证逻辑"这种模糊描述，写具体代码
- 用项目实际的测试框架和命令，不要假设 pytest
- 接线任务用模板 B，不要硬套 TDD
- 最后一个 task 一定是验证 RFL 一致性
- 正式 L 文档不动——L 内容只存 docs/plans/
- R+F+L 合力生成测试：AT←R，FT←F，UT←L，信息充分但不重叠
- 有数据库变更 → 必须生成 DDL 文件到 docs/plans/，TRD task 里只写引用
