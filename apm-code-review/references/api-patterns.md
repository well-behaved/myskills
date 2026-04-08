# API 设计与响应规范

## 1. 统一响应包装

### 1.1 CommonResult 结构

所有 REST 接口必须返回 `CommonResult<T>`：

```java
@Data
public class CommonResult<T> implements Serializable {
    private Integer code;   // 200=成功，其他=错误码
    private T data;         // 业务数据
    private String msg;     // 消息（成功时为空字符串）
}
```

### 1.2 使用方式

```java
// ✅ 正确：带数据的成功响应
return CommonResult.success(opsWorkshopService.findList(param));

// ✅ 正确：无数据的成功响应（增删改）
opsWorkshopService.add(param);
return CommonResult.success();

// ✅ 正确：泛型指定返回类型
public CommonResult<List<OpsWorkshopDTO>> getWorkShopTreeByPlant(OpsWorkshopFindParam param) {
    return CommonResult.success(opsWorkshopService.getWorkShopTreeByPlant(param));
}

// ❌ 错误：直接返回业务对象
return opsWorkshopService.findList(param);

// ❌ 错误：自定义响应格式
return new HashMap<String, Object>() {{ put("code", 200); put("data", list); }};
```

### 1.3 返回值泛型

```java
// ✅ 推荐：明确泛型类型（方便前端理解返回结构）
public CommonResult<OpsWorkshopCurrentUserDTO> getWorkshopByCurrentUser(...)

// ✅ 可接受：通配符（查询/分页接口常见）
public CommonResult<?> findList(...)
public CommonResult<?> findPage(...)
```

## 2. URL 路径规范

### 2.1 路径结构

```
@RequestMapping("/{feature}/{entity}")
```

| 模式 | 示例 | 说明 |
|------|------|------|
| 基础数据 | `/baseData/opsWorkshop` | feature = baseData |
| 用户管理 | `/user/sysUser` | feature = user |
| 点检管理 | `/spotcheck/opsSpotcheckOrder` | feature = spotcheck |
| 维保管理 | `/maintenance/opsMaintenanceOrder` | feature = maintenance |

### 2.2 端点命名

| 操作 | HTTP 方法 | 路径 | 示例 |
|------|-----------|------|------|
| 列表查询 | GET | `/findList` | `GET /baseData/opsWorkshop/findList` |
| 分页查询 | GET | `/findPage` | `GET /baseData/opsWorkshop/findPage` |
| 条件查询 | GET | `/findByConditions` | `GET /baseData/opsWorkshop/findByConditions` |
| 单条查询 | GET | `/{id}` | `GET /baseData/opsWorkshop/123` |
| 新增 | POST | `/add` | `POST /baseData/opsWorkshop/add` |
| 修改 | PUT | `/update` | `PUT /baseData/opsWorkshop/update` |
| 批量删除 | DELETE | `/{ids}` | `DELETE /baseData/opsWorkshop/1,2,3` |
| Excel 导入 | POST | `/import` | `POST /baseData/opsWorkshop/import` |
| Excel 导出 | GET | `/export` | `GET /baseData/opsWorkshop/export` |
| 树形查询 | GET | `/getXxxTree` | `GET /baseData/opsWorkshop/getWorkShopTree` |

**Review 要点**：
- ✅ URL 使用驼峰命名（不是 kebab-case）
- ✅ 查询用 GET，新增用 POST，修改用 PUT，删除用 DELETE
- ❌ `POST /baseData/opsWorkshop/delete`（应该用 DELETE 方法）
- ❌ `/base-data/ops-workshop`（项目不用 kebab-case）

## 3. 请求参数规范

### 3.1 GET 请求（Query Parameter）

```java
// ✅ 查询参数用对象接收（不加 @RequestBody）
@GetMapping("/findList")
public CommonResult<?> findList(OpsWorkshopFindParam param) { ... }

// ✅ 分页参数可单独声明
@GetMapping("/findByConditions")
public CommonResult<?> findByConditions(
    @RequestParam(name = "current", defaultValue = "1") Integer current,
    @RequestParam(name = "size", defaultValue = "10") Integer size,
    OpsWorkshopFindParam param) { ... }

// ✅ 路径参数
@GetMapping("/{id}")
public CommonResult<?> getById(@PathVariable Long id) { ... }
```

### 3.2 POST/PUT 请求（Request Body）

```java
// ✅ 保存/修改参数用 @RequestBody
@PostMapping("/add")
public CommonResult<?> add(@RequestBody OpsWorkshopSaveParam param) { ... }

@PutMapping("/update")
public CommonResult<?> update(@RequestBody OpsWorkshopSaveParam param) { ... }
```

### 3.3 DELETE 请求（路径参数）

```java
// ✅ 批量删除用路径参数，逗号分隔
@DeleteMapping("/{ids}")
public CommonResult<?> deleteBatch(@PathVariable List<Long> ids) { ... }
```

## 4. 注解规范

### 4.1 权限注解（Sa-Token）

```java
// ✅ 需要权限控制的接口必须加
@SaCheckPermission("operations:baseData:workshop:add")
@PostMapping("/add")
public CommonResult<?> add(@RequestBody OpsWorkshopSaveParam param) { ... }

// 权限码命名规则：{module}:{feature}:{entity}:{operation}
// operations:baseData:workshop:add
// operations:baseData:workshop:update
// operations:baseData:workshop:delete
// operations:baseData:workshop:export
// operations:workshop:tree
```

**Review 要点**：
- ✅ 增删改必须加权限注解
- ✅ 权限码遵循层级命名
- 可选：查询接口可以不加（视业务需求）

### 4.2 操作日志注解

```java
// ✅ 需要审计的操作加 @OperateLog
@OperateLog(
    menuCode = DictConstant.OPERATE_MENU_012,   // 菜单编码（字典值）
    menuName = "车间管理",                        // 菜单名称
    type = DictConstant.OPERATE_TYPE_002,        // 操作类型：001查/002增/003改/004删
    content = "新增车间"                          // 操作描述
)
@PostMapping("/add")
public CommonResult<?> add(@RequestBody OpsWorkshopSaveParam param) { ... }
```

### 4.3 数据权限注解

```java
// ✅ 需要数据权限过滤的查询接口
@DataPrivilege(tableNames = "ops_workshop")
@GetMapping("/findListByDataPrivilege")
public CommonResult<?> findListByDataPrivilege(OpsWorkshopFindParam param) { ... }
```

### 4.4 接口对接日志注解

```java
// ✅ 第三方接口对接时使用
@DockLog(interfaceEnum = InterfaceEnum.YIGO_UPDATE_MATERIAL, dockType = DockConstant.DOCK_TYPE_RECEIVE)
@PostMapping("/updateMaterial")
public CommonResult<?> updateMaterial(@RequestBody Object param) { ... }
```

## 5. Controller 编码规范

### 5.1 标准 Controller 结构

```java
/**
 * 车间
 *
 * @Author choki
 * @since 2023-03-30 15:58:19
 */
@RestController
@RequestMapping("/baseData/opsWorkshop")
public class OpsWorkshopController {

    @Resource
    private OpsWorkshopService opsWorkshopService;

    // 1. 查询接口在前
    @GetMapping("/findList")
    public CommonResult<?> findList(OpsWorkshopFindParam param) { ... }

    @GetMapping("/findPage")
    public CommonResult<?> findPage(OpsWorkshopFindParam param) { ... }

    @GetMapping("/{id}")
    public CommonResult<?> getById(@PathVariable Long id) { ... }

    // 2. 增删改接口在后
    @PostMapping("/add")
    public CommonResult<?> add(@RequestBody OpsWorkshopSaveParam param) { ... }

    @PutMapping("/update")
    public CommonResult<?> update(@RequestBody OpsWorkshopSaveParam param) { ... }

    @DeleteMapping("/{ids}")
    public CommonResult<?> deleteBatch(@PathVariable List<Long> ids) { ... }

    // 3. 导入导出接口最后
    @PostMapping("/import")
    public CommonResult<?> importEvents(MultipartFile file) { ... }

    @GetMapping("/export")
    public CommonResult<?> export(OpsWorkshopFindParam param, HttpServletResponse response) { ... }
}
```

### 5.2 Controller 禁止事项

```java
// ❌ Controller 中写业务逻辑
@PostMapping("/add")
public CommonResult<?> add(@RequestBody OpsWorkshopSaveParam param) {
    // 禁止在 Controller 中直接操作数据库
    OpsWorkshopEntity entity = BeanUtil.copyProperties(param, OpsWorkshopEntity.class);
    workshopMapper.insert(entity);  // ❌ 应该放 Service 中
    return CommonResult.success();
}

// ❌ Controller 中注入 Mapper
@Resource
private OpsWorkshopMapper opsWorkshopMapper;  // ❌ Controller 只注入 Service

// ❌ Controller 中做复杂的数据转换
@GetMapping("/findList")
public CommonResult<?> findList(OpsWorkshopFindParam param) {
    List<OpsWorkshopEntity> entities = opsWorkshopService.findList(param);
    // ❌ 大量转换逻辑不应在 Controller
    List<OpsWorkshopDTO> dtos = entities.stream().map(e -> {
        OpsWorkshopDTO dto = new OpsWorkshopDTO();
        dto.setCode(e.getCode());
        // ... 20行转换代码
        return dto;
    }).collect(Collectors.toList());
    return CommonResult.success(dtos);
}
```

## 6. Excel 导入导出规范

### 6.1 导入模式

```java
@PostMapping("/import")
public CommonResult<?> importEvents(MultipartFile file) {
    try {
        List<?> list = ExcelUtil.importExcel(file, OpsWorkshopExcelDTO.class);
        opsWorkshopService.batchSave(list);
        return CommonResult.success();
    } catch (Exception e) {
        throw new CommonException(ErrorCodeConstant.FAILED_TO_IMPORT, e.getMessage());
    }
}
```

### 6.2 导出模式

```java
@GetMapping("/export")
@SaCheckPermission("operations:baseData:workshop:export")
public CommonResult<?> export(OpsWorkshopFindParam param, HttpServletResponse response) {
    List<OpsWorkshopExcelDTO> excelList = opsWorkshopService.getExcelOutputData(param);
    try {
        ExcelUtil.export("车间管理", excelList, OpsWorkshopExcelDTO.class, response);
        return CommonResult.success();
    } catch (Exception e) {
        throw new CommonException(ErrorCodeConstant.FAILED_TO_EXPORT, e.getMessage());
    }
}
```

**Review 要点**：
- ✅ 使用框架提供的 `ExcelUtil`
- ✅ 导出接口加权限注解
- ✅ 异常要包装为 `CommonException`
