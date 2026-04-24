---
name: apm-test-gen
description: |-
  根据 git diff、commit、分支对比或指定方法/文件，自动生成符合项目约定的 Spring Boot 测试类。
  覆盖 Service 集成测试（@SpringBootTest）、纯单元测试（Mockito）、工具类测试、场景链式集成测试四种模式。
  自动处理 Sa-Token 登录模拟、多租户上下文、MyBatis-Plus 注入等项目特有基础设施。
  生成测试类后，同时在对应模块的 suite 包下创建 @SpringBootTest 测试套件（非 @Suite），方便一键运行。
  支持场景链式模式：自动识别同一期改动中有业务关联的 CRUD 方法，生成新增→修改→查询→删除的串联场景测试类。

  使用场景：
  - user: "为最近一次提交生成测试" → 解析 HEAD commit diff，为变更类生成测试 + 套件
  - user: "为 feature/xxx 分支生成测试" → diff 该分支与 master，批量生成 + 套件
  - user: "为这个方法生成测试 XxxService.doSomething" → 精确到方法级生成
  - user: "为这几个文件生成测试" → 指定文件列表，逐个生成 + 套件
  - user: "给我的改动加测试" → 自动拉取未提交变更，生成测试 + 套件
  - user: "加场景联动测试" / "生成流程测试" / "新增改查流程测试" → 识别 CRUD 关联，生成 XxxScenarioTest
---

# APM 测试生成器

根据代码变更自动生成符合项目约定的测试类。

## 触发方式

| 触发 | 输入 | 解析方式 |
|------|------|----------|
| 最近 N 次提交 | `git diff HEAD~N` | 提取变更的 Java 源文件 |
| 指定分支对比 | `git diff master...feature/xxx` | 提取分支差异文件 |
| 未提交变更 | `git diff` + `git diff --cached` | staged + unstaged |
| 指定文件 | 用户给出文件路径 | 直接使用 |
| 指定方法 | 用户给出类名.方法名 | 定位到具体方法 |
| **Worktree 模式** | 用户指定 worktree 路径 | 在该路径下执行 git 命令 |
| **场景链式模式** | 用户说"加场景测试"/"联动测试" 或自动识别 | 生成 `{功能名}ScenarioTest.java` |

## 工作流

### Phase 0: Worktree 检测（新）

在执行任何 git 操作前，先确定工作目录：

1. **检测是否为 worktree 调用**：
   - 用户明确传入路径参数（如 `--worktree /path/to/worktree` 或直接给出绝对路径）
   - 用户说 "在 xxx 目录下生成测试"
   - 当前 shell 工作目录不在主仓库根目录下

2. **解析 worktree 根目录**：
   ```bash
   # 在目标目录下执行，获取 git 根目录
   git -C <worktree_path> rev-parse --show-toplevel
   ```
   该命令在 worktree 中同样有效，会返回 worktree 自身的根目录（不是主仓库）。

3. **确认路径映射**：
   - `{worktree_root}/src/main/java/` → 源码根
   - `{worktree_root}/src/test/java/` → 测试根
   - 对多模块项目：`{worktree_root}/{module}/src/main/java/` → 各模块源码根

4. **所有后续 git 命令**使用 `-C <worktree_root>` 或在该目录下执行：
   ```bash
   git -C <worktree_root> diff HEAD~1 --name-only --diff-filter=ACMR
   ```

5. **无 worktree 参数时**：默认使用当前工作目录，行为与原来一致。

### Phase 1: 确定变更范围

1. 根据用户输入确定 diff 来源（参考 [references/diff-analysis.md](references/diff-analysis.md)）
2. 执行 `git diff` 命令获取变更文件列表
3. **过滤规则**：
   - 只保留 `src/main/java/**/*.java` 下的文件
   - 排除：`**/entity/**`、`**/dto/**`、`**/param/**`、`**/vo/**`、`**/enums/**`、`**/constant/**`、`**/config/**`、`**/mapper/*.java`（纯接口）
   - 保留：`**/service/**`、`**/util/**`、`**/helper/**`、`**/handler/**`、`**/controller/**`、`**/manager/**`
4. 输出：待生成测试的 Java 类列表（含完整路径 + 包名）

### Phase 2: 分析源码

对每个待测试类：

1. 读取完整源文件
2. 提取信息：
   - 类类型：Service / ServiceImpl / Controller / Util / Handler / Manager
   - 类注解：`@Service`、`@Component`、`@RestController` 等
   - 依赖注入字段：`@Resource` / `@Autowired` 的字段
   - 公开方法签名：方法名、参数类型、返回类型
   - 如果是 diff 模式，聚焦变更的方法（通过 diff hunk 定位）
3. 判定测试策略（参考 [references/test-patterns.md](references/test-patterns.md)）：
   - **有 Spring 注解 + 注入依赖** → Service 集成测试
   - **纯工具类（无注解、无注入）** → 纯单元测试
   - **复杂逻辑 + 可 Mock 依赖** → Mockito 单元测试

### Phase 2.5: 场景关联分析（可选，自动触发）

在 Phase 2 完成源码分析后，检查是否需要生成额外的场景链式测试类（参考 [references/scenario-test-pattern.md](references/scenario-test-pattern.md)）：

**触发条件（满足任意一项）：**
1. 用户明确要求：包含"场景"、"联动"、"流程"、"新增改查"等关键词
2. 自动识别：同一 Service 内发现 ≥2 个不同语义的 CRUD 方法（如同时有 `add` + `update`，或 `add` + `findPage`）

**分析步骤：**

1. 对本次变更的所有 Service 方法，按语义分组：
   - **写-新增**：方法名前缀 `add`、`save`、`create`、`insert`、`batchAdd`、`batchSave`
   - **写-修改**：方法名前缀 `update`、`modify`、`edit`、`change`
   - **写-删除**：方法名前缀 `delete`、`remove`、`del`、`batchDelete`
   - **读-查询**：方法名前缀 `find`、`get`、`list`、`page`、`query`、`count`
   - **状态变更**：`enable`、`disable`、`submit`、`approve`、`reject`

2. 判断是否操作同一实体：
   - 方法参数类型、返回值类型中出现相同的业务前缀（如 `OpsInventory`）
   - 同一 Service 接口/实现类中的方法默认视为操作同一实体

3. **决策**：
   - 找到 ≥2 种语义分组（如"新增"+"查询"）→ **生成** `{功能名}ScenarioTest.java`
   - 只有纯查询方法，无写操作 → **跳过**，不生成场景测试
   - 用户明确要求 → **强制生成**，即便只有查询

4. 生成的场景测试文件：
   - 文件名：`{ServiceName去掉Service后缀}ScenarioTest.java`（如 `OpsInventoryScenarioTest`）
   - 路径：与主 Service 测试类同包
   - 方法按语义顺序排列（s01 新增 → s02 修改 → s03 状态变更 → s04 查询 → s05 删除）
   - 使用 `@TestMethodOrder(MethodOrderer.MethodName.class)` 确保执行顺序

### Phase 3: 生成测试类

遵循项目测试约定（参考 [references/project-conventions.md](references/project-conventions.md)）：

1. 确定测试文件路径：`src/test/java/` + 源文件对应包路径
2. 确定测试类名：`{SourceClassName}Test.java`
3. 如果测试文件已存在：
   - 读取已有测试
   - 只为新增/变更的方法追加测试方法
   - 不覆盖已有测试
4. 如果测试文件不存在：生成完整测试类

每个测试方法：
- 方法名格式：**驼峰命名**，与被测方法同名（`findPage`）或加驼峰场景后缀（`findPageWithBizModule`）
- **禁止下划线**分隔，如 `findPage_withBizModule` 是错误的（详见 [references/project-conventions.md](references/project-conventions.md) 命名约定）
- 至少覆盖：正常流程 + 异常/边界 case
- 使用 `assertDoesNotThrow` / `assertEquals` / `assertNotNull` / `assertThrows` 等断言
- 对 void 方法使用 `assertDoesNotThrow`
- 对查询方法验证返回值结构

#### Phase 3.5: 生成测试套件

每次批量生成测试类后（≥2 个），**必须**在对应模块的 `suite` 包下生成测试套件（参考 [references/suite-pattern.md](references/suite-pattern.md)）：

1. **禁止使用 `@Suite` / `@SelectClasses`** — 需要 `junit-platform-suite` jar，项目内网环境无法下载，**编译报错**
2. 使用 `@SpringBootTest` 单类套件 — 所有测试方法直接写在套件类里，零额外依赖
3. 按模块分组：同一模块的测试类合并到一个套件，跨模块分别建套件
4. 套件文件路径：`src/test/java/{模块根包}/suite/{功能名}Suite.java`
5. 生成前检查模块 pom 是否有 `spring-boot-starter-test`，没有则补充

### Phase 4: 验证

1. 对生成的测试文件执行 `lsp_diagnostics` 确认无编译错误
2. 如有导入缺失，自动修复
3. 汇报生成结果：
   ```
   已生成 N 个测试类：
   - XxxServiceTest.java (3 个测试方法)
   - YyyUtilTest.java (5 个测试方法)
   ```

## 约束

### MUST

- 遵循项目已有的测试风格和命名约定
- Service 测试必须处理租户上下文
- 测试类与源类在相同包路径下（`src/test/java` 对应 `src/main/java`）
- 每个测试方法有中文注释说明测试目的
- 已存在的测试文件只追加不覆盖
- **所有代码必须兼容 JDK 1.8**，变量必须显式声明类型（`List<XxxDTO> list = ...`）

### MUST NOT

- 不生成 Entity / DTO / Param / VO 的测试（纯数据类）
- 不生成 Mapper 接口的测试（MyBatis-Plus 自动实现）
- 不删除或修改已有测试方法
- 不使用 Testcontainers（项目不用）
- 不使用 AssertJ（项目用的是 JUnit 5 原生断言）
- 不使用 `@Autowired`（项目约定用 `@Resource`）
- **不使用 `var`**（Java 10+ 语法，JDK 1.8 编译报错）
- **不使用 `@Suite` / `@SelectClasses`**（需要 `junit-platform-suite`，内网无法下载，编译报错）
- **场景测试不放入 Suite**（有数据副作用，独立运行）
- **禁止生成全量写操作测试**：update/delete 方法必须明确操作目标（设置 id 或传具体 id 列表），禁止无条件更新全表或传空列表/null 删除（详见 [references/project-conventions.md](references/project-conventions.md) 禁止的测试行为）

## 参考文档

| 文档 | 何时加载 |
|------|----------|
| [references/project-conventions.md](references/project-conventions.md) | 生成任何测试时 — 项目特有约定 |
| [references/test-patterns.md](references/test-patterns.md) | 判定测试策略时 — 分层模板 |
| [references/diff-analysis.md](references/diff-analysis.md) | Phase 1 解析 diff 时 |
| [references/suite-pattern.md](references/suite-pattern.md) | Phase 3.5 生成套件时 |
| [references/scenario-test-pattern.md](references/scenario-test-pattern.md) | Phase 2.5 生成场景链式测试时 |
| [references/examples.md](references/examples.md) | 需要参考具体生成示例时 |
