# APM 代码审查执行器规范

> 本文档供**审查执行器 agent** 使用。由编排器派发任务、传入文件内容，执行器按本规范逐项检查并输出结构化问题列表。

---

## 一、审查原则（执行前必读）

1. **只报告有实际价值的问题**：代码质量高时明确说明，不强行找问题，不为凑数量而上报低价值问题
2. **先陈述现象再给判断**：描述「代码层面现象 → 潜在影响」，对业务背景不明的 P1/P2 附注「如有特殊业务背景请说明」
3. **建议必须可直接应用**：给出具体代码，不输出模糊建议（如「需要优化」「考虑重构」）
4. **不把刻意为之的代码强行标为 Bug**：业务场景下合理的写法，不应套用规则机械报错

---

## 二、问题分级

| 级别 | 含义 | 处理要求 |
|------|------|---------|
| 🔴 P0 | 运行时错误、安全漏洞、数据丢失风险 | 合并前**必须**修复 |
| 🟠 P1 | 违反项目核心约定，有潜在风险 | 建议修复，**可协商排期** |
| 🟡 P2 | 违反编码规范，不影响功能 | 建议修复 |
| 🔵 P3 | 可优化项 | 留待迭代优化 |

---

## 三、按文件类型快速定位检查重点

```
Controller  → P1: 不含业务逻辑 + 入参用SaveParam/FindParam（不用Entity）
                  + 返回值禁用?通配符（CommonResult<?>→明确类型如CommonResult<XxxDTO>）
              空指针: Service调用返回值在使用前判空

Service     → P0: 事务方法名匹配规则表 + 只读方法不含写操作
              P1: 方法签名封装（返回DTO，不返回Entity）+ public方法入参防御性校验
              P2: 方法长度≤50行 + 嵌套≤3层 + switch有default + if/for必须大括号
              空指针: 所有数据库/RPC返回值判空 + 集合用CollUtil判空 + 链式调用逐级判空

ServiceImpl → 同Service
              + lambdaQuery/IService方法封装（禁止在外部Service调用lambdaQuery/listByIds/getById/list等）
              + BeanUtil转换（非手动setter）
              + 禁循环内查询（先批量查询转Map再循环取值）
              + 查询结果转Map优先封装到对应Service方法中（先查Service有无现成方法再封装，判空逻辑要健全）
              + 禁foreach中remove/add（用Iterator或removeIf）
              空指针: getById/selectById/.one()/.first()返回值必须判空后再使用
                      + map.get()结果判空 + stream().findFirst()用orElse/orElseThrow

Mapper      → P1: @Mapper + @Param("param") + 返回DTO不返回Entity

Mapper XML  → P0: #{} 参数化禁止${}
              规范: schema前缀 + <if>判空 + resultMap中LocalDateTime列加typeHandler=TimestampTypeHandler
              P1: LIKE查询用 '%' || #{val} || '%' 拼接（禁止CONCAT或${}）

Entity      → P1: 继承TenantBaseEntity + @TableName(schema+autoResultMap=true)
                  + LocalDateTime字段加@TableField(typeHandler=TimestampTypeHandler.class) + 字段注释
                  + 布尔变量不加is前缀 + 属性必须用包装类型

DTO         → P1: 继承TenantBaseDTO（仅业务DTO，自定义SQL resultType类除外）+ @Data + serialVersionUID
                  + 布尔变量不加is前缀 + 属性必须用包装类型

SaveParam   → P1: 继承BaseSaveParam + @NotBlank/@NotNull + 验证组 + i18n消息
                  + 属性必须用包装类型

FindParam   → P1: 继承BaseFindParam + @Query注解（仅MybatisQueryHelper.getWrapper()查询模式，
                  自定义XML SQL查询无需@Query）
              + 自定义XML接口Param分层：Controller用XxxPageParam（前端字段），
                  Mapper用XxxMapperParam（SQL内部字段，含xxxFlagNull/useFlag等）；
                  禁止跨领域复用通用FindParam给Mapper
              + 转换代码超5行必须抽为 toMapperParam() 私有方法，不得内联在业务方法中

Feign API   → P1: @FeignClient + Fallback类 + ServerNameConstant常量
                  + Fallback已注册到spring.factories
              空指针: Feign返回值在getData()前必须判断result非null且getData()非null
```

---

## 四、完整检查清单

### 🔴 P0 — 阻塞级（合并前必须修复）

#### 安全

- [ ] **SQL 注入**：Mapper XML 中是否使用了 `${}` 拼接用户输入？必须使用 `#{}`
- [ ] **敏感数据泄露**：DTO 中是否返回了密码、token 等敏感字段？
- [ ] **硬编码密码/密钥**：代码中是否有硬编码的密码、API Key？

#### 事务（全局 AOP 事务机制）

> 本项目使用 `TransactionalAopConfig` 对 `cn.sottop..*.service..*` 下的方法按名称前缀自动加事务。
> **写操作前缀**（REQUIRED，回滚，5000ms超时）：`create*`, `update*`, `delete*`, `import*`, `batch*`, `add*`, `assign*`, `execute*`, `adjust*`, `submit*`, `enDisable*`, `connect*`, `analysis*`, `confirm*`, `step*`, `revocation*`, `reject*`
> **嵌套事务前缀**（NESTED）：`nest*`
> **只读前缀**（NOT_SUPPORTED + readOnly）：`get*`, `find*`, `export*`
> **其他前缀 → 无事务保护！**（如 `save*`, `handle*`, `process*`, `remove*`）

- [ ] **方法名不匹配事务规则**：包含多步写操作的 Service 方法名前缀是否在全局规则表中？
- [ ] **只读方法含写操作**：以 `get*`/`find*`/`export*` 开头的方法中是否有数据库写操作？
- [ ] **同类内部调用事务失效**：同一个类内部直接调用含事务的方法，是否绕过了 AOP 代理？
- [ ] **超时风险**：写操作事务超时 5000ms，大批量操作是否可能超时？

#### 空指针（NullPointerException 风险）

> 任何可能返回 null 的表达式，在使用前都必须判空。

- [ ] **数据库查询返回值**：`selectById()`、`getById()`、`.one()`、`.first()` 返回值使用前是否判空？（应 `if (obj == null) throw new CommonException(...)` 或 `Optional` 包装）
- [ ] **链式调用 NPE**：`a.getB().getC()` 中间节点是否有为 null 的风险？`list.get(0)` 前是否检查 list 非空？
- [ ] **集合判空**：对集合进行 `.size()`/`.get()`/`.forEach()` 前，是否用 `CollUtil.isEmpty()` 判空？（不能用 `list != null && list.size() > 0`）
- [ ] **字符串判空**：对 String 调用 `.equals()`/`.contains()`/`.trim()` 前，是否用 `CharSequenceUtil.isBlank()` / `StrUtil.isNotBlank()` 判空？
- [ ] **equals 常量前置**：调用 `equals` 时，是否将常量或确定有值的对象放在前面？（正例：`"test".equals(obj)`；反例：`obj.equals("test")`）推荐使用 `Objects.equals(a, b)`
- [ ] **Map.get() 结果**：`map.get(key)` 结果在调用方法前是否判空？
- [ ] **Feign/RPC 返回值**：调用外部服务后，使用 `.getData()` 或取字段前是否检查 `result != null && result.getData() != null`？
- [ ] **Stream 终止操作**：`stream().filter(...).findFirst()` 返回 `Optional`，是否使用了 `.orElse(null)` 或 `.orElseThrow()`？（禁止直接 `.get()`）
- [ ] **BeanUtil 返回值**：`BeanUtil.toBean()` 返回值在使用前是否判空？
- **不报条件**：方法入口处已做过防御性判空、或明确业务上不可能为 null（须在代码注释中说明）

#### 数据完整性

- [ ] **删除前校验**：删除操作前是否检查了子节点/关联数据？
- [ ] **唯一性校验**：新增时是否校验了编码等唯一字段？
- [ ] **数据库操作返回值**：insert/update/delete 是否检查了影响行数？

---

### 🟠 P1 — 严重级（违反项目核心约定，建议修复，可协商排期）

#### 分层职责

- [ ] **Controller 越界**：Controller 中是否包含业务逻辑或直接注入 Mapper？
- [ ] **Service 操作 Request/Response**：Service 层是否操作了 HTTP 相关对象？
- [ ] **Service 外 lambdaQuery 链式调用**：是否存在 `xyzService.lambdaQuery().eq(...).one()` 在目标 Service 外部调用？（应封装为语义化方法）
- [ ] **跨 Service 调用 IService 方法**：是否在 Service A 中直接调用了 Service B 继承自 MyBatis-Plus IService 的方法（如 `listByIds()`、`getById()`、`list()`、`count()`、`lambdaQuery()`、`lambdaUpdate()` 等）？这些方法属于数据访问层实现细节，不应泄漏到调用方。应在 Service B 中封装语义化方法供外部调用。
- [ ] **循环内查询（N+1 问题）**：`for`/`forEach`/`stream().map()` 循环体内是否有单条数据库查询或 Feign 单条查询？（应先批量查询转 Map，再循环取值）
- [ ] **查询结果转 Map 未封装到 Service**：当需要将查询结果转为 Map（如 `Collectors.toMap()`）供循环或其他逻辑使用时，是否直接在调用处内联实现？（应优先封装为对应 Service 的方法，如 `xxxService.findMapByIds(ids)`，方便复用；**封装前必须先检查目标 Service 中是否已存在语义相同的方法，避免重复封装**）
- [ ] **跨 Service 直接注入他人 Mapper**：是否存在 Service A 直接注入 Service B 领域的 Mapper？（应通过 Service B 提供语义化方法间接访问，禁止跨领域直接操作他人 Mapper；Mapper 只允许被其所属 Service 的实现类注入和调用）

#### 方法签名封装

- [ ] **Service public 返回 Entity**：Service public 方法是否直接返回了 `XxxEntity` 或 `List<XxxEntity>`？（必须返回 `XxxDTO`）
- [ ] **Controller 入参用 Entity**：Controller 是否直接接收了 `XxxEntity` 作为请求入参？（必须用 `XxxSaveParam`）
- [ ] **散装入参**：超过 2 个相关参数是否未封装直接散传？（必须封装为 Param 对象）
- [ ] **参数/返回值泛型通配符**：方法签名（入参或返回值）中禁止使用 `?` 通配符（如 `List<?>`、`Map<String, ?>`、`batchSave(List<?> list)`）；必须指定明确的泛型类型（如 `List<OpsSupplierExcelDTO>`）。
- [ ] **Mapper @Param 注解**：Mapper 自定义方法的参数是否加了 `@Param("param")` 注解？

#### SQL 编写规范

- [ ] **LIKE 查询拼接方式**：自己实现的 SQL LIKE 模糊查询是否使用 PostgreSQL 字符串拼接运算符 `||`？（正例：`AND col LIKE '%' || #{val} || '%'`；反例：`AND col LIKE CONCAT('%', #{val}, '%')` 或 `AND col LIKE '%${val}%'`。使用 `CONCAT` 在 PostgreSQL 下等价但非惯用写法；使用 `${}` 直接拼接则有 SQL 注入风险）

#### 类型规范

- [ ] **Entity 基类**：Entity 是否继承了 `TenantBaseEntity`？
- [ ] **DTO 基类**：DTO 是否继承了 `TenantBaseDTO`？（例外：Mapper 自定义SQL `resultType` 返回类型，保持 plain POJO）
- [ ] **SaveParam 基类**：SaveParam 是否继承了 `BaseSaveParam`？
- [ ] **FindParam 基类**：FindParam 是否继承了 `BaseFindParam`？
- [ ] **@TableName 完整**：Entity 的 `@TableName` 是否包含 `schema` 和 `autoResultMap=true`？
- [ ] **LocalDateTime TypeHandler**：Entity 中 `LocalDateTime` 字段是否添加了 `@TableField(typeHandler = TimestampTypeHandler.class)`？

#### 响应与异常

- [ ] **返回值包装**：所有 REST 接口是否统一返回 `CommonResult<T>`？
- [ ] **返回值泛型明确**：`CommonResult` 及其他泛型返回值的类型参数禁止使用 `?` 通配符，必须指定明确类型（如 `CommonResult<XxxDTO>`、`CommonResult<PageResult<XxxDTO>>`、`CommonResult<List<XxxDTO>>`）？同理，方法返回类型中的 `List<?>`、`Map<String, ?>` 等均须替换为明确类型。
- [ ] **异常类型**：业务异常是否统一使用 `CommonException`（带 `ErrorCodeConstant` 错误码）？

#### 校验

- [ ] **FindParam @Query 注解**：查询字段是否标注了 `@Query`？（例外：走自定义 XML SQL 的 FindParam 无需标注）
- [ ] **自定义 XML 接口的 Param 分层**：新建接口若使用自定义 XML SQL（而非 `MybatisQueryHelper.getWrapper()`），入参应按层拆分为独立的 Param 类：Controller 层用 `XxxPageParam`（仅含前端需要传的字段，无 `xxxFlagNull` 等内部转换字段），Mapper 层用 `XxxMapperParam`（含 SQL 实际需要的全部字段，含 Service 内部转换后填充的字段如 `xxxFlagNull`/`useFlag` 等）；Service 负责两者之间的转换。禁止直接将通用 FindParam（如 `OpsMaterialFindParam`）跨领域复用给 Mapper，避免语义污染和字段扩展耦合。
- [ ] **Param 转换逻辑抽取为私有方法**：Service 中 Controller Param → Mapper Param 的转换代码若超过 5 行，必须抽取为独立的 `private XxxMapperParam toMapperParam(XxxPageParam param)` 方法，不得内联在业务方法体中。
- [ ] **public 方法入参防御性校验**：所有 public Service 方法是否在方法体开头对入参做防御性检查（null/blank/空集合）？查询方法入参为空返回空结果，写操作抛 `CommonException`。
- [ ] **private 方法必要校验**：被多处调用或处于关键业务路径的 private 方法，是否对入参做了必要的 null/空值检查？
- [ ] **封装方法判空逻辑健全**：新封装的 Service 方法（尤其是查询后转 Map / 按条件查单条等）是否具备完整的防御性逻辑？包括：入参集合/ID 判空、**集合入参过滤 null 元素并去重后再判空**、查询结果为空时返回空 Map/空集合而非 null、`Collectors.toMap` 的 value 过滤 null 及 key 冲突处理。方法应做到被任意调用方直接使用而不会 NPE。

#### 模块边界

- [ ] **跨模块直接依赖**：`*-biz` 模块是否直接 import 了其他 `*-biz` 的类？
- [ ] **Feign 缺少 Fallback**：Feign 接口是否定义了对应的 Fallback 类？
- [ ] **Feign Fallback 未注册**：新增 Feign 接口的 Fallback 是否已注册到对应 `-api` 模块的 `spring.factories`？（未注册则 Fallback Bean 不会加载，降级完全失效）
- [ ] **@Autowired 替换 @Resource**：是否统一使用 `@Resource`（而非 `@Autowired`）？

#### OOP 与集合规约（源自阿里 Java 开发手册）

- [ ] **布尔变量不加 is 前缀**：POJO 类中布尔类型变量是否以 `is` 开头？（部分框架反序列化时会将 `isDeleted` 误解析为属性名 `deleted`，导致序列化错误）
- [ ] **包装类型比较用 equals**：所有包装类对象（`Integer`/`Long`/`Boolean` 等）之间值比较是否使用了 `equals`？（禁止用 `==`，`Integer` 缓存仅覆盖 -128~127）
- [ ] **POJO 属性必须用包装类型**：Entity/DTO/Param 类的属性是否使用了包装数据类型（`Integer`/`Long`/`Boolean`）而非基本类型（`int`/`long`/`boolean`）？（基本类型有默认值，无法区分「未赋值」和「值为 0/false」）
- [ ] **禁止使用过时 API**：是否使用了 `@Deprecated` 标记的类或方法？应替换为推荐的新 API
- [ ] **Collectors.toMap 安全使用**：`Collectors.toMap()` 是否提供了 mergeFunction（key 冲突解决函数）？value 为 null 时是否做了过滤？（不提供 mergeFunction 在 key 重复时抛 `IllegalStateException`；value 为 null 时抛 NPE）
- [ ] **禁止 foreach 中增删元素**：是否在 `for-each` 循环中对集合执行了 `remove`/`add` 操作？（应使用 `Iterator` 方式或 `removeIf()`，否则抛 `ConcurrentModificationException`）

---

### 🟡 P2 — 规范级（违反编码规范，建议修复）

#### 命名

- [ ] **类命名**：是否遵循 `{Prefix}{Domain}{Layer}` 模式（Ops/Sys 前缀 + 领域名 + 层后缀）？
- [ ] **方法命名**：CRUD 方法是否使用标准命名（findList/findPage/add/update/deleteBatch）？
- [ ] **变量命名**：是否使用驼峰命名？集合变量是否用复数？变量名是否合乎语义（能清晰表达其含义，禁止 `temp`、`data`、`obj`、`result2`、单字母（循环变量 `i/j/k` 除外）等无意义命名）？
- [ ] **常量命名**：常量是否全大写 + 下划线分隔，且语义完整？（正例：`MAX_STOCK_COUNT`；反例：`MAX_COUNT`）
- [ ] **杜绝不规范缩写**：是否存在望文不知义的缩写？（反例：`AbstractClass` 缩写成 `AbsClass`，`condition` 缩写成 `condi`）
- [ ] **拼音/中文命名**：是否存在拼音与英文混合或直接使用中文的命名方式？（反例：`DaZhePromotion`、`getPingfenByName()`）

#### 注释

- [ ] **类注释**：类是否有 `@Author` + `@since` 的 Javadoc？
- [ ] **方法注释**：public 方法是否有 Javadoc？
- [ ] **业务注释**：复杂业务逻辑是否有中文步骤注释（`//1：xxx`）？

#### 代码结构

- [ ] **方法长度**：单个方法是否超过 50 行？超过必须拆分为职责单一的 private 方法
- [ ] **应抽取但未抽取的逻辑块**：方法内是否存在以下任一情形，却未提取为独立 private 方法？①代码块前有步骤注释（如 `// 1: 查询计划` `// 2: 过滤部门`）说明其为独立阶段；②同一逻辑在方法内被调用超过一次；③逻辑块职责与方法主干明显不同（如主方法负责编排，但内嵌了数据组装、格式转换、校验等细节）。满足任一条件均应抽取为私有方法，方法名清晰表达该段逻辑的意图。
- [ ] **类长度**：单个类文件是否超过 800 行？超过应考虑按职责拆分为多个类（如提取 Helper/Strategy/子 Service）。此项需人工决策拆分方案，不适合自动修复。
- [ ] **嵌套层级**：`if/for/try` 嵌套是否超过 3 层？超过应提前 return 或提取方法
- [ ] **覆写方法加 @Override**：所有覆写的方法是否添加了 `@Override` 注解？（防止方法签名拼写错误导致未真正覆写）
- [ ] **switch 规范**：`switch` 块中每个 `case` 是否以 `break`/`return` 终止？是否包含 `default` 分支？外部参数为 `String` 类型时是否先做了 null 判断？
- [ ] **if/else/for/while 必须大括号**：条件和循环语句是否使用了大括号包裹代码块？（即使只有一行代码，禁止 `if (condition) statement;` 单行形式）
- [ ] **构造方法无业务逻辑**：构造方法中是否包含业务逻辑？（初始化逻辑应放在 `init` 方法中）
- [ ] **方法返回类型优先用集合而非数组**：私有/公共方法返回多个同类元素时，是否返回了 `T[]` 数组？（应改为 `List<T>`，数组下标越界风险高且语义不如集合清晰；返回空结果时用 `Collections.emptyList()` 而非 `null`）
- [ ] **静态内部类放置位置**：静态内部类是否夹在普通方法之间？（应统一放在外层类所有方法之后、类末尾的 `}` 之前，且每个字段必须有 Javadoc 行注释说明用途）

#### 异常与日志

- [ ] **log.error 传异常对象**：`log.error` 是否传入了异常对象 `e` 作为最后一个参数？
- [ ] **禁止空 catch**：是否存在空的 catch 块？
- [ ] **禁止 printStackTrace**：是否使用了 `e.printStackTrace()`？

#### 线程安全

- [ ] **SimpleDateFormat 线程安全**：`SimpleDateFormat` 是否定义为 `static` 变量却未加锁？（应使用 `DateTimeFormatter` 或 `ThreadLocal<DateFormat>` 替代）
- [ ] **线程池创建方式**：是否使用 `Executors` 工具类创建线程池？（`FixedThreadPool`/`CachedThreadPool` 的队列/线程数为 `Integer.MAX_VALUE`，可能导致 OOM；应使用 `ThreadPoolExecutor` 显式指定参数）
- [ ] **ThreadLocal 回收**：自定义的 `ThreadLocal` 变量在使用完毕后是否在 `finally` 块中调用了 `remove()`？（线程池场景下线程复用，不清理会导致数据串联和内存泄漏）

#### Bean 转换

- [ ] **使用 BeanUtil**：对象转换是否使用 `BeanUtil.copyProperties()`？（禁止手动 setter 逐字段赋值）
- [ ] **null 过滤**：Stream 中的 map 操作后是否做了 `filter(Objects::nonNull)`？

#### 参数校验注解

- [ ] **SaveParam 校验注解**：必填字段是否标注了 `@NotBlank`/`@NotNull` + 验证组（`AddGroup.class`, `UpdateGroup.class`）+ i18n 消息（`{domain.field.notBlank}`）？

#### 操作日志

- [ ] **@OperateLog**：增删改接口是否添加了操作日志注解？

---

### 🔵 P3 — 建议级（可优化但不阻塞）

- [ ] **批量操作分批**：大批量操作是否做了分批（每批 100 条）？
- [ ] **Collectors.toMap 冲突解决**：`Collectors.toMap` 是否提供了冲突解决函数（key 重复会抛异常）？
- [ ] **重复代码**：是否有可提取为公共方法的重复逻辑？
- [ ] **TODO 标记**：代码中是否有遗留的 TODO/FIXME？
- [ ] **日志占位符**：日志是否使用 `{}` 占位符而非字符串 `+` 拼接？（反例：`log.info("result=" + obj)`；正例：`log.info("result={}", obj)` 或 `log.info("result={}", JSONUtil.toJsonStr(obj))`。注意：将对象序列化后作为参数传入占位符是合理写法，不应报为问题）
- [ ] **@Value 默认值**：`@Value` 注入是否提供了默认值？
- [ ] **循环内字符串拼接**：循环体内字符串连接是否使用了 `StringBuilder.append()`？（循环内 `+` 拼接每次迭代都会 `new StringBuilder`，浪费内存）
- [ ] **HashMap 初始容量**：`HashMap`/`HashSet` 创建时是否指定了初始容量？（推荐 `initialCapacity = 需要存储的元素个数 / 0.75 + 1`，避免频繁 rehash）
- [ ] **entrySet 遍历 Map**：遍历 Map 的 KV 时是否使用 `entrySet()` 而非 `keySet()`？（`keySet` 方式遍历了 2 次，`entrySet` 只遍历 1 次；JDK8+ 推荐 `Map.forEach`）
- [ ] **long 赋值用大写 L**：`long`/`Long` 赋值时是否使用了大写 `L`？（小写 `l` 易与数字 `1` 混淆，如 `Long a = 2l` 易被误读为 `21`）
- [ ] **集合初始化指定大小**：数据结构构造或初始化时是否指定了大小？（避免无限增长吃光内存）

---

## 五、最高频违规（优先检查顺序）

1. **Service 方法名不匹配全局事务规则** → `save*`/`handle*`/`process*`/`remove*` 均无事务保护
2. **`get*`/`find*` 方法中包含写操作** → 被 AOP 设为只读，写操作脱离事务
3. **空指针未判断** → 数据库查询返回值、链式调用、集合、Map.get()、Feign返回值
4. **Controller 越界写业务逻辑**
5. **用 `@Autowired` 代替 `@Resource`**
6. **Entity/DTO/Param 未继承基类**
7. **FindParam 缺 `@Query` 注解**
8. **返回值未用 `CommonResult` 包装**
9. **`log.error` 未传异常对象 `e`**
10. **Mapper XML 用 `${}` 拼接**
11. **Service 外直接调用其他 Service 的 IService 方法**（`lambdaQuery()`/`listByIds()`/`getById()`/`list()` 等）
12. **public 方法缺少入参防御性校验**
13. **循环内查询（N+1 问题）**
14. **查询结果转 Map 内联实现未封装到 Service** → 不利于复用，封装前需先检查 Service 是否已有同功能方法
15. **封装方法判空逻辑不健全** → 入参未判空、返回 null 而非空集合/空 Map、toMap 未处理 null value
16. **方法超过 50 行未拆分**
16a. **应抽取但未抽取**：有步骤注释、重复逻辑、或职责明显不同的代码块内嵌在方法体中，未提取为 private 方法
17. **方法签名用 Entity 而非 DTO/Param**
18. **嵌套层级超过 3 层**
19. **POJO 布尔变量加 is 前缀** → 框架序列化错误
20. **包装类用 == 比较** → `Integer` 缓存仅 -128~127，超出则不相等
21. **foreach 中 remove/add 元素** → `ConcurrentModificationException`
22. **Collectors.toMap 未处理 key 冲突和 null value** → 运行时异常
23. **返回值/参数使用 `?` 通配符**（`CommonResult<?>`、`List<?>`）→ 必须指定明确泛型类型

---

## 六、输出格式（严格遵守）

```
=== 批次审查结果 ===

【文件：path/to/File.java】

[P0] 事务保护缺失
- 位置：第 42 行，方法 saveOrder()
- 现象：方法名前缀 save* 不在全局 AOP 事务规则表内
- 影响：方法内含多步写操作（insert + update），失败时不会回滚
- 修复：将方法名改为 addOrder() 或加 @Transactional(rollbackFor = Exception.class)

[P1] public 方法缺少入参校验
- 位置：第 18 行，方法 getEntityByCode(String code)
- 现象：方法体未对 code 做 null/blank 检查，直接进入 lambdaQuery 链式调用
- 影响：code 为空时查询结果不可预期
- 修复：方法体首行加 if (CharSequenceUtil.isBlank(code)) { return null; }
- 注：如有特殊业务背景（调用方已保证非空）请说明

【文件：path/to/AnotherFile.java】

（无问题）

=== 无问题文件 ===
- path/to/CleanFile.java（符合所有规范，无问题）
```

**注意**：
- 每个问题必须包含：位置（文件+行号+方法名）、现象、影响、修复方案
- 无问题的文件必须在「无问题文件」中列出，不要静默跳过
