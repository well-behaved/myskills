# APM 代码审查执行器规范

> 本文档供**审查执行器 agent** 使用。由编排器派发任务、传入文件内容，执行器按本规范逐项检查并输出结构化问题列表。

---

## 零、执行姿态（最高优先级，凌驾于所有规则之上）

> 这些是 AI 审查时最容易漏掉或执行错误的点，**每次审查前必须过一遍，不得跳过**。

### 0-A：ServiceImpl public 方法入参校验——逐方法过一遍

审查 ServiceImpl 文件时，**列出文件中所有 `public` 方法**，对每一个单独检查：
- 方法体第一行是否对入参做了防御性校验（null / blank / 空集合）？
- **不得以「调用方已保证非空」为由跳过**，只要方法签名是 `public`，就必须有自己的校验。
- 正例：`if (CollUtil.isEmpty(ids)) { return Collections.emptyList(); }`
- 反例：直接进入 `lambdaQuery` / `listByIds` 而不判空。
- 集合入参必须用 `CollUtil.isEmpty()`，不得只判 `!= null`。

### 0-B：跨 Service 调用 IService 方法——主动判断方法来源

审查 ServiceImpl 中每一处 `xyzService.someMethod(...)` 调用时，**判断 `someMethod` 是该 Service 自己封装的语义方法，还是从 MyBatis-Plus `IService` 继承来的基础方法**：

IService 基础方法包括（不限于）：`listByIds`、`getById`、`list`、`count`、`saveBatch`、`removeByIds`、`lambdaQuery`、`lambdaUpdate`、`save`、`updateById`、`removeById`。

- 若是 IService 继承方法被外部 Service 调用 → **标记为 P1**
- 拿不准时宁可标记，附注"请确认是否为封装方法"

### 0-C：FindParam @Query 注解——判断查询模式再下结论

**先确认该 FindParam 对应的查询方式，再判断是否需要 `@Query`：**

- 查询方式 = `MybatisQueryHelper.getWrapper(findParam)` → **必须有 `@Query`**，缺少则报 P1
- 查询方式 = 自定义 XML SQL（Mapper XML 里手写 SQL） → **不需要 `@Query`**，有无均不报错

**不得在不确认查询方式的情况下直接报"缺少 @Query"。**

### 0-D：封装方法判空——拆成 4 个独立子项逐一检查

针对新封装的 Service 方法（尤其是"查询后转 Map"类方法），**逐项检查以下 4 条**，每条独立判断：

1. **入参判空**：集合/ID 入参是否在方法体开头判空？空时返回空集合/空 Map，不得返回 null
2. **null 元素过滤**：集合入参是否过滤了 null 元素并去重（`stream().filter(Objects::nonNull).distinct()`）后再判空？
3. **查询结果为空时返回空容器**：`listByIds` / `lambdaQuery` 查询结果为空时，是否返回 `Collections.emptyMap()` / `Collections.emptyList()`，而不是 `null`？
4. **toMap 安全**：`Collectors.toMap()` 是否提供了 mergeFunction（key 冲突）？value 是否过滤了 null？

### 0-E：查询结果转 Map——读不到其他文件时的处理方式

"查询结果转 Map 是否已封装到目标 Service"这一条，若**当前批次没有目标 Service 的文件**，不得凭空猜测：
- 标注为"**疑似未封装，建议检查 XxxService 是否已有同语义方法**"
- 级别照常报 P1，但在描述中注明"因未读取目标 Service 文件，无法确认"

---

## 一、审查原则

1. **只报告有实际价值的问题**：代码质量高时明确说明，不强行找问题
2. **先陈述现象再给判断**：描述「代码层面现象 → 潜在影响」，对业务背景不明的 P1/P2 附注「如有特殊业务背景请说明」
3. **建议必须可直接应用**：给出具体代码，不输出模糊建议
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
                  + 返回值禁用?通配符（CommonResult<?>/List<?>均不允许，必须明确类型：
                    列表→CommonResult<List<XxxDTO>>，分页→CommonResult<IPage<XxxDTO>>，
                    单对象→CommonResult<XxxDTO>，无返回→CommonResult<Void>）
              空指针: Service调用返回值在使用前判空

Service     → P0: 事务方法名匹配规则表 + 只读方法不含写操作
              P1: 方法签名封装（返回DTO，不返回Entity）+ public方法入参防御性校验（见0-A）
              P2: 方法长度≤50行 + 嵌套≤3层 + switch有default + if/for必须大括号
              空指针: 所有数据库/RPC返回值判空 + 集合用CollUtil判空 + 链式调用逐级判空

ServiceImpl → 同Service
              + lambdaQuery/IService方法封装（见0-B，禁止在外部Service调用IService基础方法）
              + BeanUtil转换（非手动setter）
              + 禁循环内查询（先批量查询转Map再循环取值）
              + 查询结果转Map优先封装到对应Service方法中（见0-E）
              + 禁foreach中remove/add（用Iterator或removeIf）
              空指针: getById/selectById/.one()/.first()返回值必须判空后再使用
                      + map.get()结果判空 + stream().findFirst()用orElse/orElseThrow

Mapper      → P1: @Mapper + @Param("param") + 返回DTO不返回Entity
                  + 禁止使用注解SQL（@Select/@Update/@Insert/@Delete），所有自定义SQL必须写在对应的*Mapper.xml中

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

FindParam   → P1: @Query注解判断规则见0-C（MybatisQueryHelper模式需要，自定义XML SQL模式不需要）
              + 继承BaseFindParam
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
- [ ] **跨 Service 调用 IService 方法**（见 0-B）：是否在 Service A 中直接调用了 Service B 继承自 MyBatis-Plus IService 的方法（`listByIds`、`getById`、`list`、`count`、`lambdaQuery`、`lambdaUpdate` 等）？应在 Service B 中封装语义化方法供外部调用。
- [ ] **循环内查询（N+1 问题）**：`for`/`forEach`/`stream().map()` 循环体内是否有单条数据库查询或 Feign 单条查询？（应先批量查询转 Map，再循环取值）
- [ ] **查询结果转 Map 未封装到 Service**（见 0-E）：需要将查询结果转为 Map 供循环使用时，是否直接在调用处内联实现？应封装为对应 Service 方法（如 `xxxService.findMapByIds(ids)`）。
- [ ] **跨 Service 直接注入他人 Mapper**：是否存在 Service A 直接注入 Service B 领域的 Mapper？（Mapper 只允许被其所属 ServiceImpl 注入和调用）

#### 方法签名封装

- [ ] **Service public 返回 Entity**：Service public 方法是否直接返回了 `XxxEntity` 或 `List<XxxEntity>`？（必须返回 `XxxDTO`）
- [ ] **Controller 入参用 Entity**：Controller 是否直接接收了 `XxxEntity` 作为请求入参？（必须用 `XxxSaveParam`）
- [ ] **散装入参**：超过 2 个相关参数是否未封装直接散传？（必须封装为 Param 对象）
- [ ] **参数/返回值泛型通配符**：方法签名中禁止使用 `?` 通配符（如 `List<?>`、`Map<String, ?>`）；必须指定明确的泛型类型。
- [ ] **Mapper @Param 注解**：Mapper 自定义方法的参数是否加了 `@Param("param")` 注解？
- [ ] **Mapper 禁止注解 SQL**：Mapper 接口方法是否使用了 `@Select`/`@Update`/`@Insert`/`@Delete` 注解写 SQL？所有自定义 SQL 必须写在对应的 `*Mapper.xml` 中。

#### SQL 编写规范

- [ ] **LIKE 查询拼接方式**：LIKE 模糊查询是否使用 PostgreSQL `||` 拼接？（正例：`AND col LIKE '%' || #{val} || '%'`；反例：`CONCAT(...)` 或 `'%${val}%'`）

#### 类型规范

- [ ] **Entity 基类**：Entity 是否继承了 `TenantBaseEntity`？
- [ ] **DTO 基类**：DTO 是否继承了 `TenantBaseDTO`？（例外：Mapper 自定义SQL `resultType` 返回类型，保持 plain POJO）
- [ ] **SaveParam 基类**：SaveParam 是否继承了 `BaseSaveParam`？
- [ ] **FindParam 基类**：FindParam 是否继承了 `BaseFindParam`？
- [ ] **@TableName 完整**：Entity 的 `@TableName` 是否包含 `schema` 和 `autoResultMap=true`？
- [ ] **LocalDateTime TypeHandler**：Entity 中 `LocalDateTime` 字段是否添加了 `@TableField(typeHandler = TimestampTypeHandler.class)`？

#### 响应与异常

- [ ] **返回值包装**：所有 REST 接口是否统一返回 `CommonResult<T>`？
- [ ] **返回值泛型明确**：`CommonResult` 及其他泛型返回值的类型参数禁止使用 `?` 通配符，必须指定明确类型。
- [ ] **异常类型**：业务异常是否统一使用 `CommonException`（带 `ErrorCodeConstant` 错误码）？

#### 校验

- [ ] **FindParam @Query 注解**（见 0-C）：MybatisQueryHelper 查询模式的字段必须标注 `@Query`；自定义 XML SQL 模式无需标注。先确认查询方式再下结论。
- [ ] **自定义 XML 接口的 Param 分层**：新建接口若使用自定义 XML SQL，入参应拆分为：Controller 层 `XxxPageParam`（仅含前端字段） + Mapper 层 `XxxMapperParam`（含 SQL 内部字段）；Service 负责转换。禁止将通用 FindParam 跨领域复用给 Mapper。
- [ ] **Param 转换逻辑抽取为私有方法**：Controller Param → Mapper Param 的转换代码若超过 5 行，必须抽取为 `private XxxMapperParam toMapperParam(XxxPageParam param)` 方法。
- [ ] **public 方法入参防御性校验**（见 0-A）：所有 public Service 方法是否在方法体开头对入参做防御性检查？查询方法入参为空返回空结果，写操作抛 `CommonException`。
- [ ] **private 方法必要校验**：被多处调用或处于关键业务路径的 private 方法，是否对入参做了必要的 null/空值检查？
- [ ] **封装方法入参判空**（见 0-D 子项1）：新封装的 Service 方法入参是否在开头判空？空时返回空集合/空 Map，不得返回 null。
- [ ] **封装方法 null 元素过滤**（见 0-D 子项2）：集合入参是否过滤了 null 元素并去重后再判空？
- [ ] **封装方法查询结果为空时返回空容器**（见 0-D 子项3）：查询结果为空时是否返回 `Collections.emptyMap()` / `Collections.emptyList()`？
- [ ] **封装方法 toMap 安全**（见 0-D 子项4）：`Collectors.toMap()` 是否提供了 mergeFunction？value 是否过滤了 null？

#### 模块边界

- [ ] **跨模块直接依赖**：`*-biz` 模块是否直接 import 了其他 `*-biz` 的类？
- [ ] **Feign 缺少 Fallback**：Feign 接口是否定义了对应的 Fallback 类？
- [ ] **Feign Fallback 未注册**：新增 Feign 接口的 Fallback 是否已注册到对应 `-api` 模块的 `spring.factories`？
- [ ] **@Autowired 替换 @Resource**：是否统一使用 `@Resource`（而非 `@Autowired`）？

#### OOP 与集合规约

- [ ] **布尔变量不加 is 前缀**：POJO 类中布尔类型变量是否以 `is` 开头？（框架反序列化时会将 `isDeleted` 误解析为属性名 `deleted`）
- [ ] **包装类型比较用 equals**：包装类对象（`Integer`/`Long`/`Boolean` 等）之间值比较是否使用了 `equals`？（禁止用 `==`，`Integer` 缓存仅覆盖 -128~127）
- [ ] **POJO 属性必须用包装类型**：Entity/DTO/Param 类的属性是否使用了包装数据类型（`Integer`/`Long`/`Boolean`）而非基本类型？
- [ ] **禁止使用过时 API**：是否使用了 `@Deprecated` 标记的类或方法？
- [ ] **Collectors.toMap 安全使用**：`Collectors.toMap()` 是否提供了 mergeFunction？value 为 null 时是否做了过滤？
- [ ] **禁止 foreach 中增删元素**：是否在 `for-each` 循环中对集合执行了 `remove`/`add` 操作？（应使用 `Iterator` 或 `removeIf()`）

---

### 🟡 P2 — 规范级（违反编码规范，建议修复）

#### 命名

- [ ] **类命名**：是否遵循 `{Prefix}{Domain}{Layer}` 模式（Ops/Sys 前缀 + 领域名 + 层后缀）？
- [ ] **方法命名**：CRUD 方法是否使用标准命名（findList/findPage/add/update/deleteBatch）？
- [ ] **变量命名**：是否使用驼峰命名？集合变量是否用复数？变量名是否合乎语义（禁止 `temp`、`data`、`obj`、`result2`、单字母（循环变量 `i/j/k` 除外）等无意义命名）？
- [ ] **常量命名**：常量是否全大写 + 下划线分隔，且语义完整？（正例：`MAX_STOCK_COUNT`；反例：`MAX_COUNT`）
- [ ] **杜绝不规范缩写**：是否存在望文不知义的缩写？
- [ ] **拼音/中文命名**：是否存在拼音与英文混合或直接使用中文的命名方式？

#### 注释

- [ ] **类注释**：类是否有 `@author` + `@since` 的 Javadoc？
- [ ] **方法注释**：public 方法是否有 Javadoc？
- [ ] **业务注释**：复杂业务逻辑是否有中文步骤注释（`// 1：xxx`）？

#### 代码结构

- [ ] **方法长度**：单个方法是否超过 50 行？超过必须拆分为职责单一的 private 方法
- [ ] **应抽取但未抽取的逻辑块**：方法内是否存在以下任一情形，却未提取为独立 private 方法？①代码块前有步骤注释说明其为独立阶段；②同一逻辑在方法内被调用超过一次；③逻辑块职责与方法主干明显不同。满足任一条件均应抽取。
- [ ] **类长度**：单个类文件是否超过 800 行？超过应考虑按职责拆分（需人工决策，不适合自动修复）。
- [ ] **嵌套层级**：`if/for/try` 嵌套是否超过 3 层？超过应提前 return 或提取方法
- [ ] **覆写方法加 @Override**：所有覆写的方法是否添加了 `@Override` 注解？
- [ ] **switch 规范**：`switch` 每个 `case` 是否以 `break`/`return` 终止？是否包含 `default` 分支？`String` 类型参数是否先做了 null 判断？
- [ ] **if/else/for/while 必须大括号**：即使只有一行代码，禁止 `if (condition) statement;` 单行形式
- [ ] **构造方法无业务逻辑**：构造方法中是否包含业务逻辑？（应放在 `init` 方法中）
- [ ] **方法返回类型优先用集合而非数组**：返回多个同类元素时，是否返回了 `T[]` 数组？（应改为 `List<T>`）
- [ ] **静态内部类放置位置**：静态内部类是否统一放在外层类所有方法之后？

#### 异常与日志

- [ ] **log.error 传异常对象**：`log.error` 是否传入了异常对象 `e` 作为最后一个参数？
- [ ] **禁止空 catch**：是否存在空的 catch 块？
- [ ] **禁止 printStackTrace**：是否使用了 `e.printStackTrace()`？

#### 线程安全

- [ ] **SimpleDateFormat 线程安全**：`SimpleDateFormat` 是否定义为 `static` 变量却未加锁？（应使用 `DateTimeFormatter` 替代）
- [ ] **线程池创建方式**：是否使用 `Executors` 工具类创建线程池？（应使用 `ThreadPoolExecutor` 显式指定参数）
- [ ] **ThreadLocal 回收**：自定义的 `ThreadLocal` 变量在使用完毕后是否在 `finally` 块中调用了 `remove()`？

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
- [ ] **重复代码**：是否有可提取为公共方法的重复逻辑？
- [ ] **TODO 标记**：代码中是否有遗留的 TODO/FIXME？
- [ ] **日志占位符**：日志是否使用 `{}` 占位符而非字符串 `+` 拼接？（正例：`log.info("result={}", obj)`；反例：`log.info("result=" + obj)`）
- [ ] **@Value 默认值**：`@Value` 注入是否提供了默认值？
- [ ] **循环内字符串拼接**：循环体内字符串连接是否使用了 `StringBuilder.append()`？
- [ ] **HashMap 初始容量**：`HashMap`/`HashSet` 创建时是否指定了初始容量？
- [ ] **entrySet 遍历 Map**：遍历 Map 的 KV 时是否使用 `entrySet()` 或 `Map.forEach`？（`keySet` 方式遍历了 2 次）
- [ ] **long 赋值用大写 L**：`long`/`Long` 赋值时是否使用了大写 `L`？（小写 `l` 易与数字 `1` 混淆）
- [ ] **集合初始化指定大小**：数据结构构造或初始化时是否指定了大小？

---

## 五、最高频违规（优先检查顺序）

1. **Service 方法名不匹配全局事务规则** → `save*`/`handle*`/`process*`/`remove*` 均无事务保护
2. **`get*`/`find*` 方法中包含写操作** → 被 AOP 设为只读，写操作脱离事务
3. **空指针未判断** → 数据库查询返回值、链式调用、集合、Map.get()、Feign返回值
4. **Controller 越界写业务逻辑**
5. **用 `@Autowired` 代替 `@Resource`**
6. **Entity/DTO/Param 未继承基类**
7. **FindParam @Query 注解判断错误**（见 0-C，先确认查询方式再下结论）
8. **返回值未用 `CommonResult` 包装**
9. **`log.error` 未传异常对象 `e`**
10. **Mapper 注解 SQL**（`@Select`/`@Update`/`@Insert`/`@Delete`）→ 必须移入对应 `*Mapper.xml`
10a. **Mapper XML 用 `${}` 拼接**
11. **Service 外直接调用其他 Service 的 IService 方法**（见 0-B）
12. **public 方法缺少入参防御性校验**（见 0-A，ServiceImpl 逐方法过一遍）
13. **循环内查询（N+1 问题）**
14. **查询结果转 Map 内联实现未封装到 Service**（见 0-E）
15. **封装方法判空逻辑不健全**（见 0-D，4 个子项逐一检查）
16. **方法超过 50 行未拆分**
16a. **应抽取但未抽取**：有步骤注释、重复逻辑、或职责明显不同的代码块未提取为 private 方法
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
- P3 问题同样要报，不得因优先级低而静默跳过
