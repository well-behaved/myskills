# 异常处理与日志规范

## 1. 异常处理体系

### 1.1 异常类层次

```
Exception
└── RuntimeException
    └── CommonException            # 项目统一业务异常（ops-common 提供）
        ├── 构造方式1: new CommonException(ErrorCodeConstant.XXX)
        ├── 构造方式2: new CommonException(ErrorCodeConstant.XXX, message)
        └── 构造方式3: new CommonException(ErrorCodeConstant.XXX, e.getMessage())
```

### 1.2 异常抛出规范

```java
// ✅ 正确：使用 ErrorCodeConstant 中预定义的错误码
throw new CommonException(ErrorCodeConstant.DATA_NOT_EXIST);
throw new CommonException(ErrorCodeConstant.FAILED_TO_ADD);
throw new CommonException(ErrorCodeConstant.FAILED_TO_UPDATE);
throw new CommonException(ErrorCodeConstant.FAILED_TO_DELETE);
throw new CommonException(ErrorCodeConstant.CODE_EXISTS);
throw new CommonException(ErrorCodeConstant.BAD_REQUEST, "具体错误描述");
throw new CommonException(ErrorCodeConstant.FAILED_TO_IMPORT, e.getMessage());
throw new CommonException(ErrorCodeConstant.FAILED_TO_EXPORT, e.getMessage());

// ❌ 错误：自定义异常类（项目统一用 CommonException）
throw new WorkshopNotFoundException("Workshop not found");

// ❌ 错误：直接抛 RuntimeException
throw new RuntimeException("操作失败");

// ❌ 错误：使用魔法数字作为错误码
throw new CommonException(500, "操作失败");
```

### 1.3 常用错误码

| 常量名 | 场景 |
|--------|------|
| `ErrorCodeConstant.DATA_NOT_EXIST` | 查询数据不存在 |
| `ErrorCodeConstant.FAILED_TO_ADD` | 新增失败 |
| `ErrorCodeConstant.FAILED_TO_UPDATE` | 修改失败 |
| `ErrorCodeConstant.FAILED_TO_DELETE` | 删除失败 |
| `ErrorCodeConstant.CODE_EXISTS` | 编码已存在 |
| `ErrorCodeConstant.BAD_REQUEST` | 通用参数错误（可附带自定义消息） |
| `ErrorCodeConstant.FAILED_TO_IMPORT` | Excel 导入失败 |
| `ErrorCodeConstant.FAILED_TO_EXPORT` | Excel 导出失败 |
| `ErrorCodeConstant.SCHEDULED_TASK_ERROR` | 定时任务执行失败 |
| `ErrorCodeConstant.WORKSHOP_CHILD_EXISTS` | 车间存在子节点不能删除 |
| `ErrorCodeConstant.WORKSHOP_NOT_SAME_PLANT` | 父子车间不在同一工厂 |
| `ErrorCodeConstant.ID_FAILED_TO_ADD` | 父节点ID不合法 |

## 2. 异常处理模式

### 2.1 数据不存在

```java
// ✅ 正确模式：查询后判空
OpsWorkshopEntity entity = opsWorkshopMapper.selectById(id);
if (Objects.isNull(entity)) {
    throw new CommonException(ErrorCodeConstant.DATA_NOT_EXIST);
}
return BeanUtil.copyProperties(entity, OpsWorkshopDTO.class);
```

### 2.2 数据库操作失败

```java
// ✅ 正确模式：检查影响行数
if (opsWorkshopMapper.insert(entity) < 1) {
    throw new CommonException(ErrorCodeConstant.FAILED_TO_ADD);
}

if (opsWorkshopMapper.updateById(entity) < 1) {
    throw new CommonException(ErrorCodeConstant.FAILED_TO_UPDATE);
}

if (opsWorkshopMapper.deleteById(entity) < 1) {
    throw new CommonException(ErrorCodeConstant.FAILED_TO_DELETE);
}
```

### 2.3 业务校验

```java
// ✅ 正确模式：存在性校验
opsWorkshopMapper.selectList(MybatisQueryHelper.getWrapper(new OpsWorkshopEntity(), findParam))
    .stream()
    .findFirst()
    .ifPresent(l -> {
        throw new CommonException(ErrorCodeConstant.CODE_EXISTS);
    });

// ✅ 正确模式：关联校验
if (selfAndChildren.contains(param.getParentId())) {
    throw new CommonException(ErrorCodeConstant.ID_FAILED_TO_ADD);
}
```

### 2.4 外部调用异常包装

```java
// ✅ 正确模式：try-catch 包装为 CommonException
try {
    List<?> list = ExcelUtil.importExcel(file, OpsWorkshopExcelDTO.class);
    opsWorkshopService.batchSave(list);
    return CommonResult.success();
} catch (Exception e) {
    throw new CommonException(ErrorCodeConstant.FAILED_TO_IMPORT, e.getMessage());
}
```

### 2.5 禁止的异常处理模式

```java
// ❌ 吞掉异常（空 catch 块）
try {
    opsWorkshopMapper.insert(entity);
} catch (Exception e) {
    // 什么都不做 — 严重禁止！
}

// ❌ 打印栈追踪后不抛出
try {
    opsWorkshopMapper.insert(entity);
} catch (Exception e) {
    e.printStackTrace();  // 不要用 printStackTrace
}

// ❌ catch 后返回 null 而不是抛异常
try {
    return opsWorkshopMapper.selectById(id);
} catch (Exception e) {
    return null;  // 隐藏了错误
}
```

## 3. 全局异常处理

网关层有全局异常处理器 `GlobalExceptionHandler`，会统一拦截并格式化响应。
业务代码只需抛 `CommonException`，无需自行处理 HTTP 响应码。

```java
// 网关层自动处理：
// CommonException → CommonResult.error(errorCode, message) → HTTP 200 + JSON body
// 未知 Exception → CommonResult.error(500, "系统异常") → HTTP 200 + JSON body
```

## 4. 日志规范

### 4.1 日志框架

- 使用 **SLF4J**，通过 Lombok `@Slf4j` 注解自动生成 logger
- 底层实现为 **Logback**，配置文件在 `src/main/resources/logback/`

### 4.2 基本使用

```java
@Slf4j
@Service
public class OpsWorkshopServiceImpl extends ServiceImpl<...> implements OpsWorkshopService {

    // ✅ 正确：info 记录正常流程
    log.info("=>> Start：HttpMethod：{},Url：{}", request.getMethod(), request.getURI().getRawPath());

    // ✅ 正确：error 记录异常，带异常对象
    log.error("[defaultExceptionHandler][uri({}/{}) 发生异常]", request.getURI(), request.getMethod(), ex);

    // ✅ 正确：error 记录 RPC 调用失败
    log.error("RPC调用失败，OpsSpotcheckOrderApi.executeCreateTask远程调用失败");

    // ✅ 正确：error 记录操作失败
    log.error("运行任务失败", e);

    // ✅ 正确：warn 记录潜在问题
    log.warn("用户 {} 尝试访问无权限接口 {}", userId, uri);

    // ✅ 正确：debug 记录调试信息
    log.debug("查询参数: {}", param);
}
```

### 4.3 日志级别使用规则

| 级别 | 使用场景 | 示例 |
|------|---------|------|
| `error` | 系统异常、业务操作失败、RPC 调用失败 | `log.error("msg", e)` |
| `warn` | 潜在问题、非预期但可恢复的状况 | `log.warn("data不一致")` |
| `info` | 关键业务流程节点、请求出入口 | `log.info("开始处理订单")` |
| `debug` | 调试信息、参数详情、中间结果 | `log.debug("param: {}", p)` |

### 4.4 日志编写规范

```java
// ✅ 正确：使用 {} 占位符（SLF4J 风格）
log.info("用户 {} 创建了车间 {}", userId, workshopName);

// ✅ 正确：error 必须带异常对象（最后一个参数）
log.error("创建车间失败，参数: {}", param, e);

// ✅ 正确：方法名+描述格式
log.error("[createWorkshop][创建车间失败]", e);

// ❌ 错误：字符串拼接（性能差）
log.info("用户 " + userId + " 创建了车间 " + workshopName);

// ❌ 错误：error 不带异常对象（丢失堆栈信息）
log.error("创建车间失败");
log.error("创建车间失败, 原因: " + e.getMessage());  // 丢失堆栈

// ❌ 错误：使用 System.out / e.printStackTrace()
System.out.println("创建车间失败");
e.printStackTrace();
```

### 4.5 日志内容规范

```java
// ✅ 使用中文描述业务含义
log.info("开始执行点检工单自动生成任务");
log.error("RPC调用失败，OpsSpotcheckOrderApi.executeCreateTask远程调用失败");

// ✅ 关键操作记录入参
log.info("批量删除车间，ids: {}", ids);

// ✅ 耗时操作记录时长
long start = System.currentTimeMillis();
// ... 操作
log.info("=>> END: Time: {}ms", System.currentTimeMillis() - start);
```

## 5. 操作日志（审计日志）

除了技术日志（SLF4J），项目还有业务操作日志（AOP 切面实现）：

### 5.1 @OperateLog 注解

```java
@OperateLog(
    menuCode = DictConstant.OPERATE_MENU_012,   // 菜单编码
    menuName = "车间管理",                        // 菜单名称（中文）
    type = DictConstant.OPERATE_TYPE_002,        // 操作类型
    content = "新增车间"                          // 操作描述（中文）
)
```

### 5.2 操作类型对照

| 常量 | 含义 | 使用场景 |
|------|------|---------|
| `OPERATE_TYPE_001` | 查询 | findPage、findByConditions 等（可选） |
| `OPERATE_TYPE_002` | 新增 | add 方法 |
| `OPERATE_TYPE_003` | 修改 | update 方法 |
| `OPERATE_TYPE_004` | 删除 | deleteBatch 方法 |

### 5.3 @DockLog 注解（第三方接口对接）

```java
@DockLog(
    interfaceEnum = InterfaceEnum.YIGO_UPDATE_MATERIAL,
    dockType = DockConstant.DOCK_TYPE_RECEIVE    // RECEIVE=接收 / SEND=发送
)
```

**Review 要点**：
- ✅ 增删改接口建议加 `@OperateLog`
- ✅ 第三方对接接口必须加 `@DockLog`
- ✅ 菜单编码和操作类型使用 `DictConstant` 中的常量
