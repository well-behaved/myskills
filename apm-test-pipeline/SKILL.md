---
name: apm-test-pipeline
description: |-
  APM 后端项目测试全流程自动化编排器。
  开发完成后一键触发：扫描变更 → 检测测试覆盖 → 自动生成缺失测试 → 运行测试 → 解析失败 → 自动修复 → 循环验证 → 出报告。
  内部自动判断修业务代码还是修测试代码，最多执行 2 轮修复循环。
  不能自动修复的问题（Spring 上下文失败、数据库异常等）保留在报告供人工决策。

  使用场景：
  - user: "帮我跑一下测试" → 扫描未提交变更，全流程自动执行
  - user: "测试一下当前改动" → 同上
  - user: "跑 operations 模块的测试" → 只跑指定模块
  - user: "feature/xxx 分支跑一下测试" → diff 该分支与 master，全流程执行
  - user: "最近2次提交跑测试" → 解析最近2次提交的变更，全流程执行
  - user: "帮我测试，有报错自动修" → 明确要求全流程
---

# APM 测试全流程编排器

你是**测试流水线的编排者**，不自己写测试、不自己修代码。职责是：
**扫描变更 → 调测试生成子任务 → 跑测试 → 解析报错 → 调修复子任务 → 循环验证 → 出报告**

---

## 核心约束（整个流程不得违反）

1. **测试生成完全委托给 `apm-test-gen` Skill** — 不自己实现任何生成逻辑
2. **报错解析遵循 [references/surefire-parse.md](references/surefire-parse.md)** — 优先读 XML，兜底读控制台
3. **修复决策遵循 [references/test-fix.md](references/test-fix.md)** — AI 判断改业务还是改测试
4. **最多执行 2 轮修复循环**，超出后将残留问题写入报告，停止
5. **修复后必须跑 lsp_diagnostics**，有 error 立即回滚该文件
6. **不能自动修复的场景**（Spring 上下文失败、DB 异常等）一律标记人工处理

---

## 编排流程

### Step 0：声明启动

输出：`"正在启动测试流水线，解析变更范围..."`

---

### Step 1：确定变更范围

从用户输入中解析范围模式，执行对应 git 命令：

| 用户说 | 执行命令 |
|--------|----------|
| 无参数 / "当前改动" / "帮我跑测试" | `git diff HEAD --name-only` + `git diff --cached --name-only` |
| "feature/xxx 分支" | `git diff master...feature/xxx --name-only --diff-filter=ACMR` |
| "最近 N 次提交" | `git diff HEAD~N HEAD --name-only --diff-filter=ACMR` |
| "只跑 {模块} 模块" | 按用户指定模块过滤，其余不变 |
| 直接给文件列表 | 跳过 git，直接使用 |

**文件过滤**（与 `apm-test-gen` 规则一致）：
- 只保留 `src/main/java/**/*.java`
- 排除 entity / dto / param / vo / enums / constant / config / mapper
- 保留 service / util / helper / handler / manager / controller / listener

输出变更文件摘要：
```
变更业务类（共 N 个）：
  - [operations] OpsMoldServiceImpl.java  [Service]
  - [operations] OpsEquipmentTypeService.java  [Service]
  - [framework] ChangeLogDiffUtil.java  [Util]
```

若变更文件为 0，输出"未检测到业务代码变更，流水线结束"并停止。

---

### Step 2：检测测试覆盖

对每个变更的业务类，推导对应测试文件路径并检查是否存在：

```
业务：{module}/src/main/java/cn/sottop/xxx/service/XxxService.java
测试：{module}/src/test/java/cn/sottop/xxx/service/XxxServiceTest.java
```

分组输出：
```
需要生成测试（无测试文件）：
  - OpsMoldServiceImpl → OpsMoldServiceImplTest.java [新建]

需要追加测试（文件存在但有新增方法）：
  - OpsEquipmentTypeService → OpsEquipmentTypeServiceTest.java [追加]

已有测试，跳过：
  - ChangeLogDiffUtil → ChangeLogDiffUtilTest.java [跳过]
```

---

### Step 3：生成缺失测试（如有）

若 Step 2 中存在"需要生成"或"需要追加"的类，派发测试生成子任务：

```typescript
task(
  category="deep",
  load_skills=["apm-test-gen"],
  run_in_background=false,   // 同步等待，生成完才能跑测试
  description="生成测试类",
  prompt=`
    按 apm-test-gen 规范，为以下类生成/追加测试：

    需要新建测试的类：
    {列表}

    需要追加测试方法的类（只追加新方法，不覆盖已有）：
    {列表}

    约束：
    - 遵循 apm-test-gen 的所有 MUST / MUST NOT 规则
    - JDK 1.8，禁止 var，显式声明类型
    - 用 @Resource 不用 @Autowired
    - ≥2 个测试类时生成对应 Suite
    - 完成后告知每个文件的路径和生成的方法数
  `
)
```

生成完成后，确认文件已写入磁盘，进入 Step 4。

若无需生成，直接进入 Step 4。

---

### Step 4：执行 mvn test（第1轮）

根据涉及的模块，构造并执行测试命令：

**单模块：**
```bash
mvn test -pl {模块相对路径} -Dfile.encoding=UTF-8 --fail-at-end
```

**多模块：**
```bash
mvn test -pl {模块1},{模块2} -Dfile.encoding=UTF-8 --fail-at-end
```

**全量（用户未限定模块）：**
```bash
mvn test -Dfile.encoding=UTF-8 --fail-at-end
```

模块路径对照：
```
operations → apm-module-operations/apm-module-operations-biz
system     → apm-module-system/apm-module-system-biz
log        → apm-module-log/apm-module-log-biz
framework  → apm-framework/apm-starter-log
file       → apm-module-file
quartz     → apm-module-quartz
```

等待命令完成，读取退出码：
- **退出码 0** → 全部通过，跳到 Step 7 出报告 ✅
- **非 0** → 进入 Step 5 解析失败

---

### Step 5：解析测试失败

按 [references/surefire-parse.md](references/surefire-parse.md) 解析失败：

1. 查找所有 `{module}/target/surefire-reports/TEST-*.xml`
2. 提取失败的 `<testcase>`（含 `<failure>` 或 `<error>`）
3. 按 [references/test-fix.md](references/test-fix.md) 的决策树分类：
   - 可自动修复（NPE 业务代码 / 断言期望值错 / 编译错误）→ 进入 Step 6
   - 不可自动修复（Spring 上下文 / DB / 环境）→ 直接标记人工处理

输出失败分类摘要：
```
失败分析（共 N 个）：
  可自动修复（N 个）：
    [F1] OpsMoldServiceTest#updateStatus - NPE（业务代码 line 123）
    [F2] OpsMoldServiceTest#addItem - 断言期望值错误
  需人工处理（N 个）：
    [M1] XxxServiceTest#findPage - DataAccessException（DB 连接失败）
```

---

### Step 6：修复循环（最多 2 轮）

#### Step 6a：读取待修复文件原始内容（用于回滚）

用 Read 工具读取每个涉及修改的文件，**内存保存原始内容**。

#### Step 6b：并行派发修复子任务

对每个可自动修复的失败，派发独立修复子任务（per 文件并行）：

```typescript
task(
  category="quick",
  load_skills=[],
  run_in_background=true,   // 并行，多文件同时修
  description="修复 {文件名}（第N轮）",
  prompt=`
    你是 APM 后端项目的代码修复执行器。

    ## 硬规则
    1. 只修改本文件，不读取或修改其他任何文件
    2. 只修复列出的问题，不做额外重构、格式化、风格调整
    3. 保留所有原有逻辑，只做最小化改动
    4. 使用 Edit 工具逐条修改，不整体覆盖文件
    5. 修复完成后运行 lsp_diagnostics，发现 error 立即标注

    ## 失败信息
    测试方法：{testClass}#{testMethod}
    失败类型：{exceptionType}
    失败信息：{message}
    完整堆栈：
    {stackTrace}

    ## 修复目标文件
    文件路径：{filePath}
    修复方向：{修业务代码 / 修测试期望值 / 修测试入参}

    ## 文件当前内容
    {文件完整源码}

    ## 修复完成后输出格式
    === 修复结果 ===
    文件：{filePath}
    修改内容：{简短描述}
    lsp_diagnostics：✅ 无编译错误 / ❌ 编译错误：{错误内容}
  `
)
```

等待所有修复子任务完成。

#### Step 6c：lsp_diagnostics 验证 + 回滚

对每个被修改的文件：
- **有 error** → Write 工具写回 Step 6a 保存的原始内容，记入"回滚列表"
- **无 error** → 记入"成功列表"

#### Step 6d：再次执行 mvn test（验证轮）

只跑包含本轮修改文件的模块：

```bash
mvn test -pl {涉及的模块} -Dfile.encoding=UTF-8 --fail-at-end
```

- **全绿** → 进入 Step 7 出报告 ✅
- **仍有失败** → 轮次 +1：
  - 轮次 ≤ 2 → 回到 Step 5，解析新的失败，继续 Step 6
  - 轮次 > 2 → 将残留失败标记"超出轮次限制，需人工处理"，进入 Step 7

---

### Step 7：汇总写报告

按 [references/report.md](references/report.md) 模板，写入：

```
docs/test-pipeline-{yyyyMMdd}-{简短主题}.md
```

完成后输出摘要：
```
✅ 测试流水线完成
报告：docs/test-pipeline-20260421-ops-mold.md

测试结果：通过 8 / 失败 0 / 人工处理 2
生成测试类：3 个新建，1 个追加
自动修复：4 处

需人工处理的问题请查阅报告。
```

---

## 注意事项

- **docs/ 目录不存在时**：先创建再写文件
- **测试生成失败时**：记录原因，跳过该类，继续后续流程（不中断整个 pipeline）
- **mvn 命令超时（>10分钟）**：提示用户，建议缩小范围重试
- **没有任何失败可以修复时**：跳过修复轮，直接出报告
- **同一文件在同一轮次内有多个失败**：合并为一个修复子任务，在 prompt 中列出所有失败，一次性修复
