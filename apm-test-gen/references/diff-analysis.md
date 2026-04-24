# Git Diff 解析规则

从用户输入到提取待测试 Java 类列表的完整规则。

## Worktree 模式

当用户指定了 worktree 路径时，所有 git 命令必须在该路径下执行。

### 识别方式

- 用户明确提供路径：`/path/to/worktree`、`D:\path\to\worktree`
- 用户说 "在 xxx 目录下..."、"xxx worktree 里的改动"
- 当前 shell 工作目录本身就是 worktree（非主仓库根目录）

### 确认 worktree 根目录

```bash
# 验证路径是合法的 git 工作树（主仓库或 worktree 均可）
git -C <path> rev-parse --show-toplevel
```

返回值即为该 worktree 的根目录（对主仓库和 worktree 行为一致）。

### Worktree 下的命令模板

所有 git 命令统一加 `-C <worktree_root>` 前缀，或在该目录下执行：

```bash
# 文件列表
git -C <worktree_root> diff HEAD~1 --name-only --diff-filter=ACMR

# 方法级 diff
git -C <worktree_root> diff HEAD~1 -U0 -- <relative_path>

# 未提交变更
git -C <worktree_root> diff --name-only --diff-filter=ACMR
git -C <worktree_root> diff --cached --name-only --diff-filter=ACMR
```

### 路径映射

Worktree 的路径结构与主仓库完全一致，路径映射规则不变：

```
{worktree_root}/{module}/src/main/java/...  →  源文件
{worktree_root}/{module}/src/test/java/...  →  测试文件（写入目标）
```

测试文件**写入 worktree 目录**，不写入主仓库。

---

## 输入识别

根据用户的自然语言指令，映射到对应的 git 命令：

| 用户说 | 识别为 | 执行命令 |
|--------|--------|----------|
| "为最近一次提交生成测试" | 最近 1 次 commit | `git diff HEAD~1 --name-only --diff-filter=ACMR` |
| "为最近 N 次提交生成测试" | 最近 N 次 commit | `git diff HEAD~N --name-only --diff-filter=ACMR` |
| "给我的改动加测试" / "为未提交的变更生成测试" | 工作区变更 | `git diff --name-only --diff-filter=ACMR` + `git diff --cached --name-only --diff-filter=ACMR` |
| "为 feature/xxx 分支生成测试" | 分支对比 | `git diff master...feature/xxx --name-only --diff-filter=ACMR` |
| "为 branchA 和 branchB 的差异生成测试" | 双分支对比 | `git diff branchA...branchB --name-only --diff-filter=ACMR` |
| "为这几个文件生成测试 A.java B.java" | 指定文件 | 直接使用文件列表 |
| "为 XxxService.doSomething 方法生成测试" | 指定方法 | 定位文件 → 提取方法 |
| "在 /path/to/worktree 下生成测试" | Worktree 模式 | 先执行 Phase 0 路径检测，再执行对应命令 |

### diff-filter 说明

`--diff-filter=ACMR` 只保留：
- **A** (Added) — 新增文件
- **C** (Copied) — 复制文件
- **M** (Modified) — 修改文件
- **R** (Renamed) — 重命名文件

排除 **D** (Deleted)，已删除的文件不需要生成测试。

## 文件过滤

从 diff 输出的文件列表中，按以下规则筛选：

### 步骤 1：只保留 Java 源文件

```
✅ 保留：src/main/java/**/*.java
❌ 排除：src/test/java/**       （已有测试文件）
❌ 排除：src/main/resources/**  （配置文件）
❌ 排除：pom.xml, *.xml, *.yml  （非代码文件）
```

### 步骤 2：排除不测试的类型

按路径或类名后缀排除：

```
❌ **/entity/**          或 *Entity.java
❌ **/dto/**             或 *DTO.java
❌ **/param/**           或 *Param.java
❌ **/vo/**              或 *VO.java
❌ **/enums/**           或 枚举定义
❌ **/constant/**        或 *Constant.java / *Constants.java
❌ **/config/**          或 *Config.java / *Configuration.java
❌ **/mapper/*Mapper.java（MyBatis Mapper 接口）
❌ **/convert/**         或 *Convert.java / *Converter.java
❌ **/Application.java   （启动类）
```

### 步骤 3：保留可测试的类型

```
✅ **/service/**         Service 接口和实现
✅ **/service/impl/**    Service 实现类
✅ **/util/**            工具类
✅ **/helper/**          辅助类
✅ **/handler/**         处理器
✅ **/manager/**         管理器
✅ **/controller/**      控制器
✅ **/listener/**        事件监听器
✅ **/aspect/**          AOP 切面
✅ **/spi/**             SPI 接口
```

### 步骤 4：接口 vs 实现类处理

- 如果 diff 中同时包含 `XxxService.java`（接口）和 `XxxServiceImpl.java`（实现），**只为实现类生成测试**
- 如果只有接口变更，检查是否存在实现类，为实现类生成测试
- 如果是纯接口（SPI、抽象类），按模板 C 生成接口契约测试

## 方法级 Diff 提取

当需要精确到方法级别时（指定方法或增量追加测试），解析 diff hunk：

```bash
git diff HEAD~1 -U0 -- path/to/File.java
```

`-U0` 表示不输出上下文行，只输出变更行。

### Hunk 解析规则

```diff
@@ -45,0 +46,15 @@ public class XxxServiceImpl {
+    public void newMethod() {
+        // 新增方法体
+    }
```

- `@@` 行中 `+46,15` 表示新文件从第 46 行开始的 15 行是新增内容
- 通过行号范围定位到具体方法
- 用 AST 或正则匹配方法签名：`(public|protected|private)?\s+\w+\s+\w+\s*\(`

### 变更方法提取流程

1. 获取 diff hunk 的行号范围
2. 读取完整源文件
3. 找到包含变更行的方法（往上找最近的方法签名）
4. 提取方法名和签名
5. 只为这些变更的方法生成测试

## 已有测试检查

在生成前，检查目标测试文件是否已存在：

```bash
# 推导测试文件路径
源文件：apm-module-operations/apm-module-operations-biz/src/main/java/cn/sottop/operations/mold/service/OpsMoldStokeService.java
测试文件：apm-module-operations/apm-module-operations-biz/src/test/java/cn/sottop/operations/mold/service/OpsMoldStokeServiceTest.java
```

### 已存在时的处理

1. 读取已有测试文件
2. 提取已有测试方法名列表
3. 对比变更方法 vs 已有测试方法
4. 只为**没有对应测试**的方法生成新测试方法
5. 追加到已有测试类末尾（在最后一个 `}` 之前插入）

### 追加模式注意

- 不重复添加 import 语句（检查已有 import）
- 不修改类声明和已有的 `@BeforeAll` / `@BeforeEach`
- 新方法保持与已有方法相同的代码风格（缩进、空行）

## 多模块定位

项目有多个 Maven 模块，需要根据文件路径确定所属模块：

```
apm-framework/apm-starter-log/src/main/java/...    → framework 模块
apm-module-operations/apm-module-operations-biz/... → operations 模块
apm-module-system/apm-module-system-biz/...         → system 模块
```

测试文件必须放在**同一模块**的 `src/test/java` 下。

## 输出格式

完成 diff 分析后，输出结构化的待测试清单：

```
待生成测试的类：
1. [operations] OpsMoldStokeService (Service 集成测试)
   - 变更方法：add(), findPage(), updateStatus()
   - 测试文件：已存在，追加模式
   
2. [framework] ChangeLogDiffUtil (纯单元测试)
   - 变更方法：diff() — 新增方法
   - 测试文件：已存在，追加模式

3. [operations] OpsNewHandler (Service 集成测试)
   - 变更方法：handle() — 新增类
   - 测试文件：不存在，新建
```

确认清单后，进入 Phase 2（源码分析）和 Phase 3（生成）。
