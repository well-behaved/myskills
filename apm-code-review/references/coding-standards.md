# 编码规范与命名约定

## 1. 类命名规范

### 1.1 分层类命名

| 层 | 命名模式 | 示例 |
|----|---------|------|
| Controller | `{Prefix}{Domain}Controller` | `OpsWorkshopController`, `SysUserController` |
| Service 接口 | `{Prefix}{Domain}Service` | `OpsWorkshopService`, `SysUserService` |
| Service 实现 | `{Prefix}{Domain}ServiceImpl` | `OpsWorkshopServiceImpl`, `SysUserServiceImpl` |
| Mapper | `{Prefix}{Domain}Mapper` | `OpsWorkshopMapper`, `SysUserMapper` |
| Entity | `{Prefix}{Domain}Entity` | `OpsWorkshopEntity`, `SysUserEntity` |
| DTO（输出） | `{Prefix}{Domain}DTO` | `OpsWorkshopDTO`, `SysUserDTO` |
| Excel DTO | `{Prefix}{Domain}ExcelDTO` | `OpsWorkshopExcelDTO` |
| 查询参数 | `{Prefix}{Domain}FindParam` | `OpsWorkshopFindParam`, `SysUserFindParam` |
| 保存参数 | `{Prefix}{Domain}SaveParam` | `OpsWorkshopSaveParam`, `SysUserSaveParam` |
| RPC DTO | `{Prefix}{Domain}RpcDTO` | `SysUserDeptRpcDTO` |
| RPC Param | `{Prefix}{Domain}RpcFindParam` | `SysDeptRpcFindParam` |
| Feign 接口 | `{Prefix}{Domain}Api` | `OpsSpotcheckOrderApi`, `SysUserDeptApi` |
| Feign Fallback | `{Prefix}{Domain}Fallback` | `OpsSpotcheckOrderFallback` |

### 1.2 前缀规则

| 模块 | 前缀 | 示例 |
|------|------|------|
| system | `Sys` | `SysUser`, `SysDept`, `SysRole` |
| operations | `Ops` | `OpsWorkshop`, `OpsSpotcheckOrder` |
| 第三方对接 | 无前缀或专属前缀 | `YigoController`, `SapController`, `EamController` |

**Review 要点**：
- ✅ 类名必须带模块前缀 + 领域名 + 层后缀
- ❌ `WorkshopController`（缺少模块前缀 `Ops`）
- ❌ `OpsWorkshopDao`（应为 `OpsWorkshopMapper`）
- ❌ `OpsWorkshopVO`（项目中统一用 DTO，不用 VO）

## 2. 方法命名规范

### 2.1 CRUD 方法名（Service 层标准方法）

| 操作 | 方法名 | 返回类型 | 说明 |
|------|--------|---------|------|
| 列表查询 | `findList(XxxFindParam)` | `List<XxxDTO>` | 不分页全量查询 |
| 分页查询 | `findPage(XxxFindParam)` | `IPage<XxxDTO>` | 分页查询 |
| 条件查询 | `findByConditions(Page, XxxFindParam)` | `IPage<XxxDTO>` | 自定义 SQL 分页 |
| 单条查询 | `getById(Long id)` | `XxxDTO` | 按 ID 查询 |
| 新增 | `add(XxxSaveParam)` | `void` | 含参数校验 |
| 修改 | `update(XxxSaveParam)` | `void` | 含参数校验 |
| 批量删除 | `deleteBatch(List<Long> ids)` | `void` | 逻辑删除 |
| 批量保存 | `batchSave(List<?> entityList)` | `void` | Excel 导入 |
| Excel 导出数据 | `getExcelOutputData(XxxFindParam)` | `List<XxxExcelDTO>` | 转换为导出 DTO |

### 2.1.1 方法签名封装规范（三层强制约定）

**Controller、Service public 方法、Mapper 自定义方法的入参和返回值必须使用项目约定的封装类型，禁止直接暴露 Entity 或散装基本类型。**

#### 入参规范

| 层 | 查询方法入参 | 写操作入参 | 禁止 |
|----|------------|----------|------|
| Controller | `XxxFindParam` | `XxxSaveParam`（`@RequestBody`） | 直接接收 `XxxEntity`、多个散装字段 |
| Service public | `XxxFindParam` | `XxxSaveParam` | 直接接收 `XxxEntity` 作为业务入参 |
| Mapper 自定义方法 | `@Param("param") XxxFindParam` | — | 无 `@Param` 注解、直接传 Entity |

#### 返回值规范

| 层 | 查询返回 | 写操作返回 | 禁止 |
|----|---------|----------|------|
| Controller | `CommonResult<List<XxxDTO>>` / `CommonResult<IPage<XxxDTO>>` | `CommonResult<?>` (void 包装) | 直接返回 Entity、裸 List、`Map<String,Object>` |
| Service public | `List<XxxDTO>` / `IPage<XxxDTO>` / `XxxDTO` | `void` | 直接返回 `List<XxxEntity>`、`XxxEntity` |
| Mapper 自定义方法 | `List<XxxDTO>` / `IPage<XxxDTO>` | — | 直接返回 `List<XxxEntity>`（复杂查询应在 Mapper 层已组装为 DTO） |

#### 正确示例

```java
// ✅ Controller — 入参 FindParam，返回 CommonResult<?>
@GetMapping("/findList")
public CommonResult<?> findList(OpsWorkshopFindParam param) {
    return CommonResult.success(opsWorkshopService.findList(param));
}

@PostMapping("/add")
public CommonResult<?> add(@RequestBody OpsWorkshopSaveParam param) {
    opsWorkshopService.add(param);
    return CommonResult.success();
}

// ✅ Service public — 入参 FindParam/SaveParam，返回 DTO / void
List<OpsWorkshopDTO> findList(OpsWorkshopFindParam param);
void add(OpsWorkshopSaveParam param);
OpsWorkshopDTO getById(Long id);

// ✅ Mapper 自定义方法 — 入参 FindParam（加 @Param），返回 DTO
IPage<OpsWorkshopDTO> findByConditions(Page<OpsWorkshopDTO> page,
                                       @Param("param") OpsWorkshopFindParam param);
List<OpsWorkshopDTO> findLists(@Param("param") OpsWorkshopFindParam param);
```

#### 禁止示例

```java
// ❌ Service 直接返回 Entity 列表（业务层不应暴露 Entity）
List<OpsWorkshopEntity> findList(OpsWorkshopFindParam param);

// ❌ Controller 直接接收 Entity（Entity 含 MyBatis-Plus 注解，不适合作为 HTTP 入参）
@PostMapping("/add")
public CommonResult<?> add(@RequestBody OpsWorkshopEntity entity) { ... }

// ❌ Service 入参使用散装字段，超过 2 个相关参数应封装为 Param
void update(Long id, String name, String code, Long plantId, Integer useFlag);
// ✅ 应封装为
void update(OpsWorkshopSaveParam param);

// ❌ Mapper 自定义方法返回 Entity（复杂查询已有 DTO 字段，无需再转）
List<OpsWorkshopEntity> findByConditions(@Param("param") OpsWorkshopFindParam param);

// ❌ Controller 直接返回裸 Map 或 Entity
@GetMapping("/findList")
public List<OpsWorkshopEntity> findList(...) { ... }  // 无 CommonResult 包装，返回 Entity
```

**Review 要点**：
- 🟠 P1：Service public 方法返回 `XxxEntity` 或 `List<XxxEntity>` → 必须改为 `XxxDTO`
- 🟠 P1：Controller 接收 `XxxEntity` 作为入参 → 必须改为 `XxxSaveParam`
- 🟠 P1：超过 2 个相关入参散装传递 → 必须封装为 `XxxParam` 或 `XxxSaveParam`
- 🟠 P1：Mapper 自定义方法缺少 `@Param("param")` 注解 → MyBatis XML 无法正确映射参数

### 2.2 Controller 方法名

Controller 方法名应与 Service 方法名一致或语义对应：

```java
@GetMapping("/findList")     → findList()
@GetMapping("/findPage")     → findPage()
@GetMapping("/{id}")         → getById()
@PostMapping("/add")         → add()
@PutMapping("/update")       → update()
@DeleteMapping("/{ids}")     → deleteBatch()
@PostMapping("/import")      → importEvents()
@GetMapping("/export")       → export()
```

### 2.3 内部方法命名

| 用途 | 命名模式 | 示例 |
|------|---------|------|
| 参数校验 | `checkSaveParam(param)` | `checkSaveParam(OpsWorkshopSaveParam)` |
| Entity→DTO 转换 | `getOtherData(entityList)` | `getOtherData(List<OpsWorkshopEntity>)` |
| 导入参数校验 | `validateParam(list)` | `validateParam(List<OpsWorkshopEntity>)` |
| 构建树形结构 | `buildTree(list)` | `buildTree(List<OpsWorkshopDTO>)` |

### 2.4 方法长度限制

**单个方法不得超过 70 行，超过必须拆分。**

```java
// ❌ 违规：一个方法超过 70 行，混杂参数校验、数据查询、业务计算、结果组装
@Override
public OpsOrderResultDTO executeOrder(OpsOrderParam param) {
    // 参数校验 10 行
    // 查询仓库 15 行
    // 计算库存 20 行
    // 生成单据 25 行
    // 发送通知 10 行
    // 共 80 行 ❌
}

// ✅ 拆分为职责单一的私有方法，每个方法 ≤ 70 行
@Override
public OpsOrderResultDTO executeOrder(OpsOrderParam param) {
    validateOrderParam(param);                    // 参数校验
    OpsWarehouseEntity warehouse = getOrderWarehouse(param);  // 查询仓库
    BigDecimal stock = calculateStock(warehouse, param);      // 计算库存
    OpsOrderEntity order = buildOrderEntity(param, stock);    // 生成单据
    notifyOrderCreated(order);                    // 发送通知
    return BeanUtil.copyProperties(order, OpsOrderResultDTO.class);
}
```

**拆分原则**：
- 按业务步骤拆分（校验 → 查询 → 计算 → 持久化 → 通知）
- 每个 private 方法只做一件事，方法名能准确描述其职责
- 拆分后的 private 方法同样受 70 行限制约束

**Review 要点**：
- 🟠 P1：方法超过 70 行 → 必须拆分，不可豁免
- 统计行数时包含空行和注释行，不含方法签名行和结束 `}`

## 3. 变量命名规范

```java
// ✅ 驼峰命名
private String empNo;
private Long parentId;
private Integer useFlag;
private LocalDate birthday;

// ✅ 集合变量用复数或 List 后缀
List<Long> ids;
List<Long> userIdList;
List<String> nameList;
Set<Long> plantIds;

// ✅ Map 变量说明 key-value 语义
Map<Long, OpsWorkshopDTO> workshopMap;  // key=workshopId, value=workshop
Map<String, OpsWorkshopDTO> workshopMapByName;  // key=name

// ❌ 禁止
private String EMPNO;      // 不是常量不用全大写
private String emp_no;     // 不用下划线命名
private List<Long> id;     // 集合应用复数
```

## 4. 常量命名规范

```java
// ✅ 全大写 + 下划线分隔 + 语义化前缀
public static final String OPERATE_MENU_012 = "012";
public static final String OPERATE_TYPE_001 = "001";  // 查询
public static final String OPERATE_TYPE_002 = "002";  // 新增
public static final String OPERATE_TYPE_003 = "003";  // 修改
public static final String OPERATE_TYPE_004 = "004";  // 删除
public static final Integer FALSE = 0;
public static final Integer TRUE = 1;

// 常量统一定义在 Constant 类中
DictConstant.OPERATE_MENU_012
BaseConstant.FALSE
ErrorCodeConstant.DATA_NOT_EXIST
ServerNameConstant.OPERATIONS_SERVER_NAME
```

**Review 要点**：
- ✅ 使用已有的 Constant 类，不重复定义
- ❌ 在业务代码中硬编码魔法数字 `if (status == 1)`
- ❌ 在 Service 中自定义常量（应放 Constant 类）

## 5. 注释规范

### 5.1 类注释（必须）

```java
/**
 * 车间
 *
 * @Author choki
 * @since 2023-03-30 15:58:19
 */
@RestController
@RequestMapping("/baseData/opsWorkshop")
public class OpsWorkshopController { ... }
```

### 5.2 方法注释

```java
/**
 * 车间列表
 *
 * @param param
 * @return
 * @Author choki
 * @since 2023-03-30 15:58:19
 */
@GetMapping("/findList")
public CommonResult<?> findList(OpsWorkshopFindParam param) { ... }
```

### 5.3 行内注释

```java
// 中文注释，解释业务逻辑
// 1：检查保存参数
checkSaveParam(param);
// 2：保存
OpsWorkshopEntity entity = BeanUtil.copyProperties(param, OpsWorkshopEntity.class);
```

**Review 要点**：
- ✅ 每个类必须有 `@Author` + `@since` 注释
- ✅ 公开方法必须有 Javadoc
- ✅ 业务逻辑步骤用中文编号注释（`//1：xxx`、`//2：xxx`）
- ❌ 缺少类级别注释
- ❌ 英文注释（项目统一用中文）

## 6. 类结构规范

### 6.1 Entity 类模板

```java
@Data
@EqualsAndHashCode(callSuper = true)
@TableName(schema = "operations", value = "ops_workshop", autoResultMap = true)
public class OpsWorkshopEntity extends TenantBaseEntity {
    private static final long serialVersionUID = 1L;

    /** 编号 */
    private String code;
    /** 名称 */
    private String name;
    /** 父级ID */
    private Long parentId;
}
```

**必须元素**：
- `@Data` + `@EqualsAndHashCode(callSuper = true)`
- `@TableName` 带 `schema` 和 `value`，`autoResultMap = true`
- 继承 `TenantBaseEntity`
- `serialVersionUID`
- 每个字段有注释

### 6.2 DTO 类模板

```java
@Data
@EqualsAndHashCode(callSuper = true)
public class OpsWorkshopDTO extends TenantBaseDTO {
    private static final long serialVersionUID = 1L;

    /** 编号 */
    private String code;
    /** 名称 */
    private String name;
    /** 父级车间名称（关联查询字段） */
    private String parentName;
}
```

**必须元素**：
- `@Data` + `@EqualsAndHashCode(callSuper = true)`
- 继承 `TenantBaseDTO`
- `serialVersionUID`
- 可包含关联查询的展示字段

### 6.3 SaveParam 类模板

```java
@Data
@EqualsAndHashCode(callSuper = true)
public class OpsWorkshopSaveParam extends BaseSaveParam {
    private static final long serialVersionUID = 1L;

    @NotBlank(message = "{workshop.code.notBlank}", groups = {AddGroup.class, UpdateGroup.class})
    private String code;

    @NotBlank(message = "{workshop.name.notBlank}", groups = {AddGroup.class, UpdateGroup.class})
    private String name;

    private Long parentId;
}
```

**必须元素**：
- 继承 `BaseSaveParam`
- 必填字段标注 `@NotBlank` 或 `@NotNull`
- 使用 i18n 消息 key：`{domain.field.notBlank}`
- 使用验证组：`AddGroup.class`, `UpdateGroup.class`

### 6.4 FindParam 类模板

```java
@Data
@EqualsAndHashCode(callSuper = true)
public class OpsWorkshopFindParam extends BaseFindParam {
    private static final long serialVersionUID = 1L;

    @Query(propName = "id", type = Query.Type.IN)
    private List<Long> ids;

    @Query(type = Query.Type.INNER_LIKE)
    private String code;

    @Query(type = Query.Type.INNER_LIKE)
    private String name;

    @Query(type = Query.Type.EQUAL)
    private Long parentId;

    @Query(type = Query.Type.EQUAL)
    private Long plantId;
}
```

**必须元素**：
- 继承 `BaseFindParam`
- 每个查询字段标注 `@Query`（指定查询类型）
- 不参与 MyBatis-Plus Wrapper 构建的字段不加 `@Query`

### 6.5 @Query 注解类型参考

| 类型 | 用途 | 示例 |
|------|------|------|
| `Query.Type.EQUAL` | 等值匹配 | 状态、ID、类型字段 |
| `Query.Type.INNER_LIKE` | 模糊匹配（`%value%`） | 名称、编号搜索 |
| `Query.Type.IN` | IN 查询 | ID 列表、名称列表 |
| `Query.Type.BETWEEN` | 范围查询 | 日期区间 |
| `Query.Type.GE` | 大于等于 | 开始时间 |
| `Query.Type.LE` | 小于等于 | 结束时间 |

## 7. 依赖注入规范

```java
// ✅ 正确：使用 @Resource（JDK 标准注解）
@Resource
private OpsWorkshopService opsWorkshopService;

@Resource
private OpsWorkshopMapper opsWorkshopMapper;

// ✅ 正确：循环依赖时用 @Lazy
@Lazy
@Resource
private OpsPlantService opsPlantService;

// ❌ 错误：使用 @Autowired（项目不使用）
@Autowired
private OpsWorkshopService opsWorkshopService;

// ❌ 错误：构造函数注入（项目不使用）
public OpsWorkshopServiceImpl(OpsWorkshopService service) { ... }
```

## 8. Bean 转换规范

```java
// ✅ 正确：使用 Hutool BeanUtil
// 单个对象转换
OpsWorkshopDTO dto = BeanUtil.copyProperties(entity, OpsWorkshopDTO.class);

// 集合转换
List<SysUserMainDTO> list = BeanUtil.copyToList(entityList, SysUserMainDTO.class);

// Entity 列表转 DTO 列表（带额外数据填充）
private List<OpsWorkshopDTO> getOtherData(List<OpsWorkshopEntity> entityList) {
    if (CollUtil.isEmpty(entityList)) {
        return new ArrayList<>();
    }
    return entityList.stream().map(e -> {
        OpsWorkshopDTO dto = BeanUtil.copyProperties(e, OpsWorkshopDTO.class);
        if (Objects.isNull(dto)) {
            return null;
        }
        // 填充关联数据
        dto.setPlantName(opsPlantService.getById(e.getPlantId()).getName());
        return dto;
    }).filter(Objects::nonNull).collect(Collectors.toList());
}

// ❌ 错误：手动 setter 逐字段赋值（字段多时容易遗漏）
// ❌ 错误：使用 ModelMapper、MapStruct（项目未引入）
```

## 9. 参数校验规范

### 9.1 核心原则

| 方法可见性 | 校验要求 | 说明 |
|-----------|---------|------|
| `public` | **必须校验** | 对外暴露，调用方不可信，入参必须防御性检查 |
| `private` | **必要时校验** | 内部方法，但关键路径、可能被多处调用时同样需要 |

### 9.2 public 方法 — 必须校验

所有 public 方法必须在方法开头对入参做防御性检查，检查通过后再执行业务逻辑。

```java
// ✅ 单个对象参数：判断对象本身及关键字段
@Override
public OpsWarehouseEntity getEntityByCode(String code) {
    if (CharSequenceUtil.isBlank(code)) {
        return null;   // 或 throw new CommonException(ErrorCodeConstant.PARAM_ERROR)
    }
    // 业务逻辑...
}

// ✅ 集合参数：判空后再操作
@Override
public Map<Long, OpsWarehouseDTO> findMapByIds(List<Long> ids) {
    if (CollUtil.isEmpty(ids)) {
        return new HashMap<>();
    }
    // 业务逻辑...
}

// ✅ 复合参数对象：判对象本身 + 必要的关键字段
@Override
public OpsWarehouseFindByBusinessTypeDTO findByBusinessType(OpsWarehouseFindByBusinessTypeParam param) {
    if (param == null || param.getPlantQueryId() == null || param.getBusinessType() == null) {
        return new OpsWarehouseFindByBusinessTypeDTO();
    }
    // 业务逻辑...
}
```

**校验后的返回策略**：

| 场景 | 推荐做法 |
|------|---------|
| 查询方法，参数为空 | 返回 `null` 或空集合/空对象，不抛异常 |
| 写操作方法，参数为空或不合法 | 抛出 `CommonException`（带 `ErrorCodeConstant` 错误码） |
| 必须有值的字段缺失 | 抛出 `CommonException(ErrorCodeConstant.PARAM_ERROR)` |

### 9.3 private 方法 — 必要时校验

private 方法由自身类内部调用，调用路径可控，但以下场景**必须**加校验：

1. **被多处调用**：同一 private 方法被 2 处以上调用，入参来源不一致
2. **关键业务路径**：方法内涉及数据库写入、外部调用、金额/状态变更
3. **递归或复杂逻辑**：输入非法可能导致死循环或数据损坏

```java
// ✅ 被多处调用的 private 方法，加必要校验
private List<OpsWarehouseDTO> getOtherData(List<OpsWarehouseEntity> entityList) {
    if (CollUtil.isEmpty(entityList)) {   // ✅ 空集合直接返回，避免后续 NPE
        return new ArrayList<>();
    }
    // 业务逻辑...
}

// ✅ 关键校验逻辑，参数可能为 null
private void validMaterialInventory(Long id) {
    if (id == null) {   // ✅ 防止传入 null 导致查询异常
        return;
    }
    // 数据库查询校验...
}

// ⚪ 仅单处调用且调用方已校验，可以不加（但加了更安全）
private void checkSaveParam(OpsWorkshopSaveParam param) {
    // 调用方 add/update 已确保 param 非 null（通过 @Valid），此处可省略 null 检查
    // 但若 param 内部字段校验是本方法职责，仍需在此处做
}
```

### 9.4 禁止模式

```java
// ❌ public 方法无任何校验，直接使用入参
@Override
public OpsWarehouseEntity getEntityByCode(String code) {
    return this.lambdaQuery()
            .eq(OpsWarehouseEntity::getCode, code)  // code 为 null/空时查询结果不可预期
            .one();
}

// ❌ public 方法接收集合参数，未判空直接操作
@Override
public Map<Long, OpsWarehouseDTO> findMapByIds(List<Long> ids) {
    return ids.stream()   // ids 为 null 时 NPE
            .collect(...);
}

// ❌ 校验了 param 对象本身为 null，但未校验关键字段
@Override
public void executeOrder(OpsOrderParam param) {
    if (param == null) { return; }
    // orderId 可能为 null，后续数据库操作出错
    opsOrderMapper.selectById(param.getOrderId());
}
```

## 10. 集合操作规范

```java
// ✅ 使用 Hutool CollUtil 做判空
if (CollUtil.isEmpty(ids)) { return new ArrayList<>(); }
if (CollUtil.isNotEmpty(collect)) { ... }

// ✅ 使用 Stream API 做转换和过滤
List<Long> idList = ids.stream()
    .filter(Objects::nonNull)
    .distinct()
    .collect(Collectors.toList());

// ✅ 使用 Collectors.toMap 构建 Map（带冲突解决）
Map<Long, OpsWorkshopDTO> map = list.stream()
    .filter(Objects::nonNull)
    .filter(item -> item.getId() != null)
    .collect(Collectors.toMap(
        OpsWorkshopDTO::getId,
        Function.identity(),
        (oldValue, newValue) -> newValue  // 冲突取新值
    ));

// ❌ 错误：不做 null 过滤直接 collect
// ❌ 错误：Collectors.toMap 不提供冲突解决函数（key 重复会抛异常）
```
