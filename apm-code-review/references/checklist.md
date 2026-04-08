# AI Code Review 检查清单

> 本清单供**审查执行器**使用，逐项检查 apm-backend 项目代码变更。
> P0/P1 为必查项，P2/P3 为建议项。

---

## 审查原则（执行前必读）

- **不强行找问题**：代码质量良好时明确说明，不为凑数量而上报低价值问题
- **先陈述现象再给判断**：描述「代码层面现象 → 潜在影响」，对业务背景不明的 P1/P2 附注「如有特殊业务背景请说明」
- **建议必须可直接应用**：给出具体代码，不输出模糊建议

---

## 🔴 P0 — 阻塞级（合并前必须修复）

### 安全

- [ ] **SQL 注入**：Mapper XML 中是否使用了 `${}` 拼接用户输入？必须使用 `#{}`
- [ ] **权限缺失**：增删改接口是否缺少 `@SaCheckPermission` 注解？
- [ ] **敏感数据泄露**：DTO 中是否返回了密码、token 等敏感字段？
- [ ] **硬编码密码/密钥**：代码中是否有硬编码的密码、API Key？

### 事务（全局 AOP 事务机制）

> 本项目使用 `TransactionalAopConfig` 对 `cn.sottop..*.service..*` 下的方法按名称前缀自动加事务。
> **写操作前缀**（REQUIRED + rollback on Exception, 5000ms）：`create*`, `update*`, `delete*`, `import*`, `batch*`, `add*`, `assign*`, `execute*`, `adjust*`, `submit*`, `enDisable*`, `connect*`, `analysis*`, `confirm*`, `step*`, `revocation*`, `reject*`
> **嵌套事务前缀**（NESTED）：`nest*`
> **只读前缀**（NOT_SUPPORTED + readOnly）：`get*`, `find*`, `export*`
> **其他前缀 → 无事务保护！**

- [ ] **方法名不匹配事务规则**：包含多步写操作的 Service 方法名前缀是否在全局规则表中？（如 `save*`, `handle*`, `process*`, `remove*` 均不在规则内 → 无事务！）
- [ ] **只读方法含写操作**：以 `get*`/`find*`/`export*` 开头的方法中是否有数据库写操作？（这些方法被设为只读事务）
- [ ] **同类内部调用**：同一个类内部的方法调用是否绕过了 AOP 代理导致事务失效？
- [ ] **超时风险**：写操作事务超时为 5000ms，大批量操作是否可能超时？
- [ ] **REQUIRES_NEW 场景**：需要独立事务的方法是否显式加了 `@Transactional(propagation = Propagation.REQUIRES_NEW)`？（全局 AOP 只配了 REQUIRED）

### 空指针

- [ ] **未判空**：`selectById` 返回值是否做了 null 检查？
- [ ] **集合未判空**：List/Map 操作前是否用 `CollUtil.isEmpty()` 判空？
- [ ] **链式调用 NPE**：`a.getB().getC()` 是否有中间节点为 null 的风险？

### 数据完整性

- [ ] **删除前校验**：删除操作前是否检查了子节点/关联数据？
- [ ] **唯一性校验**：新增时是否校验了编码等唯一字段？
- [ ] **数据库操作返回值**：insert/update/delete 是否检查了影响行数？

---

## 🟠 P1 — 严重级（违反项目核心约定，建议修复，可协商排期）

### 分层职责

- [ ] **Controller 越界**：Controller 中是否包含业务逻辑？
- [ ] **Controller 直接调 Mapper**：Controller 是否直接注入并调用了 Mapper？
- [ ] **Service 操作 Request/Response**：Service 层是否操作了 HTTP 相关对象？

### 方法签名封装

> 详见 `references/coding-standards.md` 第 2.1.1 节

- [ ] **Service public 返回值**：Service public 方法是否直接返回了 `XxxEntity` 或 `List<XxxEntity>`？（必须返回 `XxxDTO`）
- [ ] **Controller 入参类型**：Controller 是否直接接收了 `XxxEntity` 作为请求入参？（必须用 `XxxSaveParam`）
- [ ] **散装入参**：超过 2 个相关参数是否未封装，直接散传？（必须封装为 `XxxParam` / `XxxSaveParam`）
- [ ] **Mapper 自定义方法返回值**：Mapper 的自定义查询方法是否返回了 `List<XxxEntity>`？（复杂查询应直接返回 `List<XxxDTO>`）
- [ ] **Mapper @Param 注解**：Mapper 自定义方法的参数是否加了 `@Param("param")` 注解？

### 模块边界

- [ ] **跨模块直接依赖**：`*-biz` 模块是否直接 import 了其他 `*-biz` 的类？
- [ ] **API 模块污染**：`*-api` 模块是否包含了 ServiceImpl、Mapper 等实现类？
- [ ] **Feign 缺少 Fallback**：Feign 接口是否定义了对应的 Fallback 类？

### 类型规范

- [ ] **Entity 基类**：Entity 是否继承了 `TenantBaseEntity`？
- [ ] **DTO 基类**：DTO 是否继承了 `TenantBaseDTO`？
- [ ] **SaveParam 基类**：SaveParam 是否继承了 `BaseSaveParam`？
- [ ] **FindParam 基类**：FindParam 是否继承了 `BaseFindParam`？
- [ ] **@TableName 完整**：Entity 的 `@TableName` 是否包含 `schema` 和 `autoResultMap=true`？

### 响应格式

- [ ] **返回值包装**：所有 REST 接口是否统一返回 `CommonResult<?>`？
- [ ] **异常类型**：业务异常是否统一使用 `CommonException`？
- [ ] **错误码**：是否使用 `ErrorCodeConstant` 中的预定义错误码？

### 验证

- [ ] **SaveParam 校验注解**：必填字段是否标注了 `@NotBlank` / `@NotNull`？
- [ ] **验证组**：校验注解是否指定了 `groups = {AddGroup.class, UpdateGroup.class}`？
- [ ] **@Query 注解**：FindParam 中的查询字段是否标注了 `@Query`？
- [ ] **i18n 消息**：校验消息是否使用 i18n 格式 `{domain.field.notBlank}`？
- [ ] **public 方法入参校验**：所有 public Service 方法是否在方法体开头对入参做防御性检查（null/blank/空集合）？（详见 `references/coding-standards.md` 第 9 节）
- [ ] **private 方法必要校验**：被多处调用或处于关键业务路径的 private 方法，是否对入参做了必要的 null/空值检查？
- [ ] **校验后处理恰当**：查询方法入参为空时返回空结果；写操作入参非法时抛出 `CommonException` 而非静默返回或抛 NPE？

---

## 🟡 P2 — 规范级（违反编码规范，建议修复）

### 命名

- [ ] **类命名**：是否遵循 `{Prefix}{Domain}{Layer}` 模式？
- [ ] **方法命名**：CRUD 方法是否使用标准命名（findList/findPage/add/update/deleteBatch）？
- [ ] **模块前缀**：operations 模块类名是否带 `Ops` 前缀？system 模块是否带 `Sys`？
- [ ] **变量命名**：是否使用驼峰命名？集合变量是否用复数？

### 注释

- [ ] **类注释**：类是否有 `@Author` + `@since` 的 Javadoc？
- [ ] **方法注释**：公开方法是否有 Javadoc？
- [ ] **业务注释**：复杂业务逻辑是否有中文步骤注释（`//1：xxx`）？

### 依赖注入

- [ ] **@Resource**：是否统一使用 `@Resource`（而非 `@Autowired`）？
- [ ] **循环依赖**：循环依赖是否使用了 `@Lazy` 处理？

### 异常处理

- [ ] **异常对象传递**：`log.error` 是否传入了异常对象 `e` 作为最后一个参数？
- [ ] **禁止空 catch**：是否存在空的 catch 块？
- [ ] **禁止 printStackTrace**：是否使用了 `e.printStackTrace()`？

### Bean 转换

- [ ] **使用 BeanUtil**：对象转换是否使用 `BeanUtil.copyProperties()`？
- [ ] **null 过滤**：Stream 中的 map 操作后是否做了 `filter(Objects::nonNull)`？

### 操作日志

- [ ] **@OperateLog**：增删改接口是否添加了操作日志注解？
- [ ] **操作类型**：是否使用了正确的 `DictConstant.OPERATE_TYPE_xxx`？

---

## 🔵 P3 — 建议级（可优化但不阻塞）

### 性能

- [ ] **N+1 查询**：循环中是否有单条数据库查询？是否可以批量查询优化？
- [ ] **批量操作分批**：大批量操作是否做了分批（每批 100 条）？
- [ ] **Stream 去重**：`Collectors.toMap` 是否提供了冲突解决函数？

### 代码质量

- [ ] **魔法数字**：是否存在硬编码的数字/字符串？应使用常量
- [ ] **重复代码**：是否有可提取为公共方法的重复逻辑？
- [ ] **方法长度**：单个方法是否超过 70 行？超过必须拆分为多个语义清晰的私有方法
- [ ] **嵌套层级**：`if/for/try` 嵌套是否超过 3 层？超过应提前 return 或提取方法
- [ ] **参数个数**：方法参数是否超过 5 个？建议封装为 Param 对象

### 配置

- [ ] **硬编码配置值**：应该放在配置文件中的值是否写在了代码里？
- [ ] **@Value 默认值**：`@Value` 注入是否提供了默认值？

### 其他

- [ ] **TODO 标记**：代码中是否有遗留的 TODO/FIXME？
- [ ] **无用 import**：是否有未使用的 import？
- [ ] **日志占位符**：日志是否使用 `{}` 占位符而非字符串拼接？

---

## 审查报告模板

```markdown
## 审查摘要

- **审查范围**：[描述变更涉及的模块和文件]
- **审查文件数**：N
- **问题总数**：N（🔴 x / 🟠 x / 🟡 x / 🔵 x）
- **总体评价**：[通过 ✅ / 需修改后通过 ⚠️ / 需重大修改 ❌]

## 问题详情

### 📄 `path/to/File.java`

#### 🔴 P0: [问题标题]
- **位置**: 第 N 行
- **问题**: [描述违反了什么规范]
- **现有代码**: `[问题代码片段]`
- **修复建议**: `[修复后的代码]`
- **规范参考**: [对应的规范文档和条目]

#### 🟡 P2: [问题标题]
- **位置**: 第 N 行
- **问题**: [描述]
- **修复建议**: [建议]

## ✅ 做得好的地方

- [列出代码中值得肯定的实践]

## 📋 建议后续优化

- [列出不阻塞但值得后续改进的点]
```

---

## 快速判断流程图

```
收到代码 → 是否新文件？
├── 是 → 检查：类命名、基类继承、注解完整性、包位置
└── 否 → 检查：变更是否破坏已有约定

→ 文件类型判断：
├── Controller → API 模式 + 权限注解 + 操作日志 + 响应包装
├── Service    → 事务管理 + 异常处理 + 参数校验
├── ServiceImpl → 同 Service + Bean 转换 + 查询模式
├── Mapper     → 继承 BaseMapper + @Mapper + @Param
├── Mapper XML → #{} 参数化 + <if> 判空 + schema 前缀
├── Entity     → 基类 + @TableName + 字段注释
├── DTO        → 基类 + @Data + serialVersionUID
├── SaveParam  → 基类 + 校验注解 + 验证组
├── FindParam  → 基类 + @Query 注解
└── Feign API  → @FeignClient + Fallback + prefix 常量
```
