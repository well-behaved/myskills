# 数据库访问与 SQL 规范

## 1. ORM 框架：MyBatis-Plus

项目统一使用 **MyBatis-Plus 3.5.3.1**，基于 `BaseMapper` + `ServiceImpl` 模式。

### 1.1 Mapper 接口规范

```java
/**
 * 车间
 *
 * @Author choki
 * @since 2023-03-30 15:58:19
 */
@Mapper
public interface OpsWorkshopMapper extends BaseMapper<OpsWorkshopEntity> {

    // 自定义查询（复杂 SQL 写在 XML 中）
    IPage<OpsWorkshopDTO> findByConditions(
        Page<OpsWorkshopDTO> page,
        @Param("param") OpsWorkshopFindParam param
    );

    List<OpsWorkshopDTO> findLists(
        @Param("param") OpsWorkshopFindParam param
    );
}
```

**必须元素**：
- `@Mapper` 注解
- 继承 `BaseMapper<XxxEntity>`
- 自定义方法参数用 `@Param` 注解

### 1.2 Service 实现规范

```java
@Service
public class OpsWorkshopServiceImpl
    extends ServiceImpl<OpsWorkshopMapper, OpsWorkshopEntity>
    implements OpsWorkshopService {

    @Resource
    private OpsWorkshopMapper opsWorkshopMapper;

    // ... 业务方法
}
```

**必须元素**：
- `@Service` 注解
- 继承 `ServiceImpl<XxxMapper, XxxEntity>`
- 实现 `XxxService` 接口

## 2. 查询模式

### 2.1 使用 MybatisQueryHelper（首选）

项目自研的查询构建器，配合 `@Query` 注解自动构建 Wrapper：

```java
// ✅ 正确：通过 MybatisQueryHelper 自动构建查询条件
@Override
public List<OpsWorkshopDTO> findList(OpsWorkshopFindParam param) {
    return this.getOtherData(
        opsWorkshopMapper.selectList(
            MybatisQueryHelper.getWrapper(new OpsWorkshopEntity(), param)
        )
    );
}

// ✅ 正确：分页查询
@Override
public IPage<OpsWorkshopDTO> findPage(OpsWorkshopFindParam param) {
    OpsWorkshopEntity entity = new OpsWorkshopEntity();
    IPage<OpsWorkshopEntity> entityPage = opsWorkshopMapper.selectPage(
        MybatisQueryHelper.getPage(entity, param),    // 分页参数
        MybatisQueryHelper.getWrapper(entity, param)   // 查询条件
    );
    // 转换为 DTO 分页
    IPage<OpsWorkshopDTO> dtoPage = new Page<>(
        entityPage.getCurrent(),
        entityPage.getSize(),
        entityPage.getTotal()
    );
    dtoPage.setRecords(this.getOtherData(entityPage.getRecords()));
    return dtoPage;
}
```

### 2.2 使用 lambdaQuery（简单条件查询）

```java
// ✅ 适用于简单的链式查询
List<Long> userIdList = sysUserDeptService.lambdaQuery()
    .eq(SysUserDeptEntity::getDeptId, param.getDeptId())
    .list()
    .stream()
    .map(SysUserDeptEntity::getUserId)
    .collect(Collectors.toList());
```

### 2.3 Service 方法封装规范（禁止在 Service 外暴露 lambdaQuery 链式调用）

**核心原则**：禁止在一个 Service 内部直接调用另一个 Service 的 MyBatis-Plus 扩展方法（`lambdaQuery()`、`lambdaUpdate()` 等），这些方法属于数据访问层的实现细节，不应泄漏到业务逻辑中。

```java
// ❌ 禁止：在 ServiceA 内部通过 ServiceB 暴露的 lambdaQuery 构建查询条件
OpsWarehouseEntity warehouse = opsWarehouseService.lambdaQuery()
    .eq(OpsWarehouseEntity::getCode, param.getLgort())
    .eq(OpsWarehouseEntity::getEnable, BaseConstant.TRUE)
    .last("LIMIT 1")
    .one();

// ✅ 正确：先检查 opsWarehouseService 是否已有合适方法
//         若没有，则在 OpsWarehouseService 中封装语义化方法，再调用
OpsWarehouseEntity warehouse = opsWarehouseService.getEntityByCode(param.getLgort());
```

**封装方法示例**（在目标 Service 内实现）：

```java
// OpsWarehouseService 接口
/**
 * 根据仓库编码获取启用的仓库实体
 *
 * @param code 仓库编码
 * @return 仓库实体，不存在则返回 null
 */
OpsWarehouseEntity getEntityByCode(String code);

// OpsWarehouseServiceImpl 实现
@Override
public OpsWarehouseEntity getEntityByCode(String code) {
    if (CharSequenceUtil.isBlank(code)) {
        return null;
    }
    return this.lambdaQuery()
            .eq(OpsWarehouseEntity::getCode, code)
            .eq(OpsWarehouseEntity::getEnable, BaseConstant.TRUE)
            .last("LIMIT 1")
            .one();
}
```

**Review 要点**：
- ✅ 调用方只依赖语义化方法名，不感知底层查询构建细节
- ✅ 被调用 Service 内部可自由使用 `lambdaQuery()` 实现
- ❌ 在外部 Service / Controller 中链式调用 `xyzService.lambdaQuery().eq(...).one()`
- ❌ 为了"方便"将 `lambdaQuery()` 结果直接返回给调用方

### 2.3 使用 Mapper XML（复杂查询）

当查询涉及多表 JOIN、子查询等复杂 SQL 时，使用 XML：

```xml
<select id="findByConditions"
        parameterType="cn.sottop.operations.baseData.param.OpsWorkshopFindParam"
        resultType="cn.sottop.operations.baseData.dto.OpsWorkshopDTO">
    select e1.id, plant.name plantName, e1.code, e1.name,
           e2.id parentId, e2.name parentName, e2.enable parentEnable,
           e1.description, e1.dept_id deptId, dept.name deptName
    from operations.ops_workshop e1
    LEFT OUTER JOIN operations.ops_workshop e2 ON e1.parent_id = e2.id
    LEFT JOIN operations.ops_plant plant ON e1.plant_id = plant.id
    LEFT JOIN system.sys_dept dept ON dept.enable=1 and e1.dept_id = dept.id
    <where>
        e1.enable=1
        <if test="param.code != null">
            and e1.code like '%' || #{param.code} || '%'
        </if>
        <if test="param.name != null">
            and e1.name like '%' || #{param.name} || '%'
        </if>
        <if test="param.parentId != null">
            and e1.parent_id = #{param.parentId}
        </if>
        <if test="param.plantId != null">
            and e1.plant_id = #{param.plantId}
        </if>
    </where>
    ORDER BY e1.updated_time DESC
</select>
```

## 3. Mapper XML 规范

### 3.1 文件位置

```
src/main/resources/mapper/{feature}/XxxMapper.xml
```

与 Java Mapper 接口对应：
- `mapper/baseData/OpsWorkshopMapper.xml` → `cn.sottop.operations.baseData.mapper.OpsWorkshopMapper`

### 3.2 XML 结构模板

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
    "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="cn.sottop.operations.baseData.mapper.OpsWorkshopMapper">

    <!-- ResultMap 定义 -->
    <resultMap type="cn.sottop.operations.baseData.entity.OpsWorkshopEntity" id="BaseResultMap">
        <result property="id" column="id"/>
        <result property="code" column="code"/>
        <result property="createdTime" column="created_time"
                typeHandler="cn.sottop.framework.mybatis.handler.TimestampTypeHandler"/>
        <!-- ... -->
    </resultMap>

    <!-- 公共列定义 -->
    <sql id="Base_Column_List">
        id, code, name, parent_id, enable, ...
    </sql>

    <!-- 自定义查询 -->
    <select id="findByConditions" ...>
        ...
    </select>
</mapper>
```

### 3.3 SQL 编写规范

```xml
<!-- ✅ 正确：使用 #{} 参数化（防 SQL 注入） -->
<if test="param.code != null">
    and e1.code like '%' || #{param.code} || '%'
</if>

<!-- ✅ 正确：使用 <foreach> 处理 IN 查询 -->
<if test="param.ids != null">
    and e1.id in
    <foreach collection='param.ids' item='id' open='(' separator=',' close=')'>
        #{id}
    </foreach>
</if>

<!-- ✅ 正确：使用 <where> 自动处理 AND 前缀 -->
<where>
    e1.enable=1
    <if test="param.code != null">and e1.code like '%' || #{param.code} || '%'</if>
</where>

<!-- ❌ 禁止：使用 ${} 拼接（SQL 注入风险） -->
<if test="param.code != null">
    and e1.code like '%${param.code}%'  <!-- 严重安全风险！ -->
</if>

<!-- ❌ 禁止：不加 <if> 判空就拼接条件 -->
and e1.code = #{param.code}  <!-- param.code 为 null 时会出错 -->
```

### 3.4 PostgreSQL 特殊注意

- 表名必须带 schema 前缀：`operations.ops_workshop`、`system.sys_dept`
- 字符串拼接用 `||`：`'%' || #{param.code} || '%'`
- 时间字段用 `TimestampTypeHandler` 处理

## 4. Entity 与表映射规范

### 4.1 @TableName 注解

```java
// ✅ 正确：指定 schema 和表名
@TableName(schema = "operations", value = "ops_workshop", autoResultMap = true)
public class OpsWorkshopEntity extends TenantBaseEntity { ... }

// 不同模块的 schema：
// system 模块 → schema = "system"
// operations 模块 → schema = "operations"
// log 模块 → schema = "log"
```

### 4.2 表命名规范

| 模块 | 表前缀 | 示例 |
|------|--------|------|
| system | `sys_` | `sys_user`, `sys_dept`, `sys_role` |
| operations | `ops_` | `ops_workshop`, `ops_spotcheck_order` |
| log | `log_` | `log_operate`, `log_exception` |

### 4.3 字段映射

```java
// Entity 字段（驼峰） → 数据库字段（下划线）自动映射
private Long parentId;       // → parent_id
private String createdBy;    // → created_by
private Long tenantId;       // → tenant_id
```

### 4.4 基类字段（TenantBaseEntity 提供）

所有 Entity 通过继承获得以下公共字段：

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | Long | 主键 |
| `enable` | Integer | 启用/禁用（1/0，逻辑删除标记） |
| `description` | String | 描述 |
| `plantId` | Long | 工厂 ID |
| `createdBy` | String | 创建人 |
| `createdTime` | Timestamp | 创建时间 |
| `updatedBy` | String | 更新人 |
| `updatedTime` | Timestamp | 更新时间 |
| `trxId` | String | 事务 ID |
| `creationPid` | String | 创建进程 ID |
| `updatePid` | String | 更新进程 ID |
| `tenantId` | Long | 租户 ID |
| `version` | Integer | 乐观锁版本号 |

**Review 要点**：
- ✅ 不要在 Entity 中重复定义基类已有的字段
- ✅ 逻辑删除通过 `enable=0` 实现，不用物理删除
- ❌ 自定义 `id`、`createTime` 等基类已提供的字段

## 5. 事务管理规范

### ⚠️ 核心机制：全局 AOP 事务拦截（TransactionalAopConfig）

本项目配置了**全局声明式事务 AOP**，自动对 `cn.sottop..*.service..*` 包下的方法按方法名前缀匹配事务策略。
**大多数 Service 方法无需手动加 `@Transactional`，全局 AOP 已自动覆盖。**

> 配置类：`cn.sottop.framework.mybatis.transaction.TransactionalAopConfig`
> 切入点：`execution(* cn.sottop..*.service..*.*(..))`

### 5.1 全局事务规则表

| 方法名前缀 | 事务传播 | 回滚规则 | 超时 | 说明 |
|-----------|---------|---------|------|------|
| `create*` | REQUIRED | rollback on Exception | 5000ms | 写操作 |
| `update*` | REQUIRED | rollback on Exception | 5000ms | 写操作 |
| `delete*` | REQUIRED | rollback on Exception | 5000ms | 写操作 |
| `import*` | REQUIRED | rollback on Exception | 5000ms | 写操作（Excel 导入） |
| `batch*` | REQUIRED | rollback on Exception | 5000ms | 写操作（批量） |
| `add*` | REQUIRED | rollback on Exception | 5000ms | 写操作 |
| `assign*` | REQUIRED | rollback on Exception | 5000ms | 写操作（分配） |
| `execute*` | REQUIRED | rollback on Exception | 5000ms | 写操作（执行） |
| `adjust*` | REQUIRED | rollback on Exception | 5000ms | 写操作（调整） |
| `submit*` | REQUIRED | rollback on Exception | 5000ms | 写操作（提交） |
| `enDisable*` | REQUIRED | rollback on Exception | 5000ms | 写操作（启用/禁用） |
| `connect*` | REQUIRED | rollback on Exception | 5000ms | 写操作（关联） |
| `analysis*` | REQUIRED | rollback on Exception | 5000ms | 写操作（分析） |
| `confirm*` | REQUIRED | rollback on Exception | 5000ms | 写操作（确认） |
| `step*` | REQUIRED | rollback on Exception | 5000ms | 写操作（步骤） |
| `revocation*` | REQUIRED | rollback on Exception | 5000ms | 写操作（撤销） |
| `reject*` | REQUIRED | rollback on Exception | 5000ms | 写操作（驳回） |
| `nest*` | NESTED | rollback on Exception | 5000ms | 嵌套事务 |
| `get*` | NOT_SUPPORTED | readOnly | — | 只读查询 |
| `find*` | NOT_SUPPORTED | readOnly | — | 只读查询 |
| `export*` | NOT_SUPPORTED | readOnly | — | 只读导出 |
| **其他前缀** | **无事务** | — | — | **⚠️ 不匹配任何规则，无事务保护！** |

### 5.2 Review 关键要点

#### ✅ 已被全局 AOP 覆盖 → 通常不需要加 `@Transactional`

```java
// ✅ 方法名以 add 开头 → 全局 AOP 自动加 REQUIRED 事务
public void add(@Valid OpsWorkshopSaveParam param) {
    checkSaveParam(param);
    OpsWorkshopEntity entity = BeanUtil.copyProperties(param, OpsWorkshopEntity.class);
    opsWorkshopMapper.insert(entity);
    // 无需 @Transactional，AOP 已覆盖
}

// ✅ deleteBatch → 匹配 delete* 规则
public void deleteBatch(List<Long> ids) { ... }

// ✅ batchSave → 匹配 batch* 规则
public void batchSave(List<?> entityList) { ... }
```

#### 🔴 P0 审查重点：方法名不匹配规则表 → 没有事务保护

```java
// ❌ 危险：方法名 "save" 不在规则表中 → 无事务！
public void save(OpsWorkshopSaveParam param) {  // save* 未被覆盖！
    opsWorkshopMapper.insert(entity1);
    opsWorkshopMapper.insert(entity2);  // 如果这里失败，entity1 不会回滚
}

// ❌ 危险：方法名 "remove" 不在规则表中 → 无事务！
public void removeAll(List<Long> ids) { ... }

// ❌ 危险：方法名 "handle" 不在规则表中 → 无事务！
public void handleOrder(Long orderId) {
    updateStatus(orderId);
    createRecord(orderId);  // 失败不回滚
}

// ✅ 修复方案 1：改用规则表内的方法名前缀
public void deleteAll(List<Long> ids) { ... }        // 匹配 delete*
public void executeHandleOrder(Long orderId) { ... }  // 匹配 execute*

// ✅ 修复方案 2：手动加 @Transactional 覆盖
@Transactional(rollbackFor = Exception.class)
public void handleOrder(Long orderId) { ... }
```

#### ⚠️ 需要显式加 `@Transactional` 的场景

```java
// 1. 需要 REQUIRES_NEW（独立事务）— 全局 AOP 只配了 REQUIRED
@Transactional(propagation = Propagation.REQUIRES_NEW, rollbackFor = Exception.class)
public void tryCreate(...) { ... }

// 2. 方法名不在规则表中但包含写操作
@Transactional(rollbackFor = Exception.class)
public void processOrder(Long orderId) { ... }  // process* 不在规则表中

// 3. 同类内部调用需要走事务（AOP 代理不生效场景）
// 通过自注入代理解决：
@Resource
private OpsSerialNoCreatorServiceImpl self;

public void doWork() {
    self.tryCreate(...);  // 通过代理调用，事务注解生效
}
```

#### ⚠️ get*/find* 是只读事务（NOT_SUPPORTED）的影响

```java
// ⚠️ 以 get/find 开头的方法被设为只读 + NOT_SUPPORTED
// 如果 get/find 方法中有写操作，会出问题！

// ❌ 错误：find 开头但内部有写操作
public List<OpsWorkshopDTO> findAndSync(OpsWorkshopFindParam param) {
    List<OpsWorkshopDTO> list = findList(param);
    syncToExternalSystem(list);    // 如果这里有 DB 写入 → 不在事务中！
    return list;
}

// ✅ 修复：含写操作的方法不要用 get/find/export 开头
public List<OpsWorkshopDTO> executeSync(OpsWorkshopFindParam param) { ... }
```

### 5.3 事务注意事项

- **全局 AOP 优先级**：`@Transactional` 注解和全局 AOP 同时存在时，`@Transactional` 优先（更具体的切面胜出）
- **超时 5 秒**：写操作事务超时为 5000ms，长耗时操作（如大批量导入）需注意
- **同类内部调用**：不走 AOP 代理，全局事务和 `@Transactional` 都不生效，需用自注入（`self`）模式
- **Controller 层无事务**：切入点只匹配 service 包，Controller 中的方法不受事务保护

## 6. 批量操作规范

### 6.1 批量插入

```java
// ✅ 正确：分批插入，每批 100 条
if (CollUtil.isNotEmpty(collect)) {
    int batchInt = 100;
    int listSize = collect.size();
    int toIndex = batchInt;
    for (int i = 0; i < collect.size(); i += batchInt) {
        if (i + batchInt > listSize) {
            toIndex = listSize - i;
        }
        List<OpsWorkshopEntity> lists = collect.subList(i, i + toIndex);
        this.saveBatch(lists);
    }
}
```

### 6.2 批量删除

```java
// ✅ 正确：逻辑删除（设置 enable=0）
@Override
public void deleteBatch(List<Long> ids) {
    OpsWorkshopEntity entity = new OpsWorkshopEntity();
    // 1：校验是否有子节点
    // ... 校验逻辑
    // 2：循环删除
    entity.setEnable(BaseConstant.FALSE);
    ids.forEach(id -> {
        entity.setId(id);
        if (opsWorkshopMapper.deleteById(entity) < 1) {
            throw new CommonException(ErrorCodeConstant.FAILED_TO_DELETE);
        }
    });
}
```

**Review 要点**：
- ✅ 删除前检查关联数据（子节点、被引用记录）
- ✅ 使用逻辑删除（`enable=0`），不使用物理删除
- ✅ 批量操作检查返回影响行数
- ❌ 不检查就直接删除
- ❌ 使用 `DELETE FROM` 物理删除
