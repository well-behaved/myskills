# 场景链式集成测试模式

同一期改动中存在业务关联的多个方法时，除了各自独立的测试类，还需生成一个**场景链式测试类**，
模拟真实业务流程（新增 → 修改 → 查询 → 删除）串联验证。

## 何时触发场景链式测试

满足以下条件时，生成额外的 `{功能名}ScenarioTest.java`：

| 条件 | 说明 |
|------|------|
| 同一 Service 内存在 ≥2 个 CRUD 操作 | 如同时有 `add` + `update` + `findPage` |
| 方法操作同一实体对象 | 方法参数/返回值中出现同一实体类（如 `OpsXxx`）|
| 用户明确要求生成场景测试 | "加场景测试"、"加联动测试"、"加流程测试"等 |

**不生成场景测试的情况：**
- 只有查询方法，没有写操作
- 各方法操作不同实体，没有关联
- 只生成了 1 个类且方法彼此独立

---

## 方法语义识别规则

通过方法名前缀识别语义，再推断执行顺序：

| 语义 | 方法名前缀 | 执行顺序 |
|------|-----------|---------|
| 新增 | `add`、`save`、`create`、`insert`、`batchAdd`、`batchSave` | 1（最先） |
| 修改 | `update`、`modify`、`edit`、`change` | 2 |
| 删除 | `delete`、`remove`、`del`、`batchDelete` | 最后（清理） |
| 查询 | `find`、`get`、`list`、`page`、`query`、`count` | 新增之后、删除之前 |
| 状态变更 | `enable`、`disable`、`submit`、`approve`、`reject` | 介于新增和查询之间 |

---

## 场景测试类模板

文件名：`{功能名}ScenarioTest.java`
路径：与主 Service 测试类同包（`src/test/java/{包路径}/`）

```java
package {与源码相同的包路径};

import cn.hutool.json.JSONUtil;
import cn.sottop.framework.tenant.context.TenantContextHolder;
import lombok.extern.slf4j.Slf4j;
import org.junit.jupiter.api.*;
import org.springframework.boot.test.context.SpringBootTest;

import javax.annotation.Resource;
import java.util.ArrayList;
import java.util.List;

import static org.junit.jupiter.api.Assertions.*;

/**
 * {功能名} 场景链式集成测试
 * 模拟完整业务流程：{step1} → {step2} → {step3}
 *
 * ⚠️ 数据隔离：使用 "SCENARIO_TEST_" 前缀 + 时间戳命名测试数据，@AfterAll 统一清理
 */
@SpringBootTest
@Slf4j
@TestMethodOrder(MethodOrderer.MethodName.class)
class {FunctionName}ScenarioTest {

    @Resource
    private {ServiceName} {serviceField};

    /** 记录本次测试创建的 ID，供 @AfterAll 清理 */
    private static final List<Long> CREATED_IDS = new ArrayList<Long>();

    /** 跨方法传递的实体 ID */
    private static Long testEntityId;

    @BeforeAll
    static void beforeAll() {
        log.info("初始化租户上下文");
        TenantContextHolder.setTenantId(1L);
    }

    @AfterAll
    static void afterAll() {
        // 清理测试数据（吞掉异常，不影响测试报告）
        if (!CREATED_IDS.isEmpty()) {
            try {
                // TODO: 替换为实际的删除方法
                // {serviceField}.deleteBatch(new ArrayList<Long>(CREATED_IDS));
                log.info("清理测试数据: {}", CREATED_IDS);
            } catch (Exception e) {
                log.warn("清理测试数据失败，请手动清理 IDs: {}", CREATED_IDS, e);
            }
        }
        TenantContextHolder.clear();
    }

    // ==================== Step 1: 新增 ====================

    /**
     * 场景第 1 步：新增 {实体名}，验证不报错
     */
    @Test
    void s01{Entity}Add() {
        String paramJson = "{\"name\":\"SCENARIO_TEST_{场景标识}_\" + System.currentTimeMillis()}";
        {SaveParamType} param = JSONUtil.toBean(paramJson, {SaveParamType}.class);

        assertDoesNotThrow(() -> {serviceField}.add(param));

        // 新增后查回 ID，供后续步骤使用
        // 若 add() 有返回值，直接 testEntityId = {serviceField}.add(param)
        {FindParamType} findParam = new {FindParamType}();
        findParam.setName(param.getName());
        // testEntityId = {serviceField}.findList(findParam).get(0).getId();
        log.info("s01 新增完成");
    }

    // ==================== Step 2: 修改 ====================

    /**
     * 场景第 2 步：修改刚新增的 {实体名}，验证修改生效
     */
    @Test
    void s02{Entity}Update() {
        assertNotNull(testEntityId, "需先执行 s01 新增步骤");

        {UpdateParamType} param = new {UpdateParamType}();
        param.setId(testEntityId);
        // param.setXxx("修改后的值");

        assertDoesNotThrow(() -> {serviceField}.update(param));
        log.info("s02 修改完成 id={}", testEntityId);
    }

    // ==================== Step 3: 查询验证 ====================

    /**
     * 场景第 3 步：查询验证数据一致性
     */
    @Test
    void s03{Entity}FindPage() {
        assertNotNull(testEntityId, "需先执行 s01 新增步骤");

        assertDoesNotThrow(() -> {
            // 单条查询
            // {DtoType} result = {serviceField}.getById(testEntityId);
            // assertNotNull(result);
            // assertEquals("修改后的值", result.getXxx());

            // 或分页查询
            {FindParamType} param = new {FindParamType}();
            param.setCurrent(1L);
            param.setSize(20L);
            // IPage<{DtoType}> page = {serviceField}.findPage(param);
            // assertNotNull(page);
            log.info("s03 查询完成");
        });
    }

    // ==================== Step 4: 删除 / 清理 ====================

    /**
     * 场景第 4 步：删除，验证数据被清理（兼做测试数据清理）
     */
    @Test
    void s04{Entity}Delete() {
        if (testEntityId == null) {
            log.warn("testEntityId 为 null，跳过删除步骤");
            return;
        }

        assertDoesNotThrow(() -> {
            // {serviceField}.delete(testEntityId);
            log.info("s04 删除完成 id={}", testEntityId);
        });

        // 从清理列表中移除（已手动删除，无需 @AfterAll 二次删除）
        CREATED_IDS.remove(testEntityId);
    }
}
```

## 事务说明（重要）

项目通过 `TransactionalAopConfig` 对 `cn.sottop..*.service..*.*(..)` 统一织入事务，规则如下：

| 方法前缀 | 事务传播 | 测试影响 |
|---------|---------|---------|
| `add*` `update*` `delete*` `batch*` 等 | REQUIRED（提交） | 每步执行后**立即提交**，下一步能读到 ✅ |
| `find*` `get*` | SUPPORTS 只读 | 加入已有事务或无事务执行，均可读到已提交数据 ✅ |

**结论：场景链式测试（新增→查询→修改→删除）不会有事务隔离问题。**
每个 `@Test` 方法独立调用 Service，事务各自提交，数据在下一步可见。

### 禁止在场景测试类上加 `@Transactional`

```java
// ❌ 禁止：会导致所有步骤共享同一事务，测试结束后整体回滚
// 让人误以为新增成功但查不到数据
@SpringBootTest
@Transactional   // ← 禁止
class XxxScenarioTest { }

// ✅ 正确：不加 @Transactional，每步各自提交
@SpringBootTest
class XxxScenarioTest { }
```

### 清理方法命名要匹配事务切点

`@AfterEach` / `s04Delete` 里调用的删除方法必须以事务切点覆盖的前缀开头，否则异常不回滚：

| 调用方式 | 是否在事务切点内 |
|---------|----------------|
| `service.deleteBatch(ids)` | ✅ `delete*` 已覆盖 |
| `service.batchDelete(ids)` | ✅ `batch*` 已覆盖 |
| `service.removeById(id)` | ❌ `remove*` 未覆盖，无事务 |

若 Service 只有 `remove*` 前缀的删除方法，清理时用 `assertDoesNotThrow` 包裹，并在 catch 里打 warn 日志提示手动清理。

---

| 场景 | 传递方式 |
|------|---------|
| add() 有返回值（Long id）| 直接 `testEntityId = service.add(param)` |
| add() 无返回值（void）| 新增后调 `findList()` + 名称匹配找回 id |
| 多个实体需传递 | 用 `private static final Map<String, Long> ENTITY_IDS = new LinkedHashMap<String, Long>()` |

```java
// add() 无返回值时，通过查询找回 id 的示例
private static Long findIdByName(XxxService service, String name) {
    XxxFindParam p = new XxxFindParam();
    p.setName(name);
    List<XxxDTO> list = service.findList(p);
    for (XxxDTO dto : list) {
        if (name.equals(dto.getName())) {
            return dto.getId();
        }
    }
    throw new IllegalStateException("新增后找不到记录: " + name);
}
```

---

## 测试数据命名规范

- 格式：`"SCENARIO_TEST_{业务标识}_" + System.currentTimeMillis()`
- 目的：时间戳避免并发冲突；前缀便于手动排查清理
- 示例：`"SCENARIO_TEST_物料管理_1720000000000"`

---

## @AfterAll 清理策略

```java
@AfterAll
static void afterAll() {
    if (!CREATED_IDS.isEmpty()) {
        try {
            xxxService.deleteBatch(new ArrayList<Long>(CREATED_IDS));
        } catch (Exception e) {
            log.warn("清理失败，请手动删除 IDs: {}", CREATED_IDS, e);
        }
    }
    TenantContextHolder.clear();
}
```

**注意**：`@AfterAll` 方法中**不能直接引用** `@Resource` 注入的字段（Spring 实例非静态），
需要通过**静态辅助方法**或在 `@AfterEach` + 实例方法中清理：

```java
// ✅ 用 @AfterEach（实例方法，可访问 @Resource 字段）
@AfterEach
void cleanup() {
    if (!CREATED_IDS.isEmpty()) {
        try { xxxService.deleteBatch(new ArrayList<Long>(CREATED_IDS)); }
        catch (Exception e) { log.warn("清理失败", e); }
        CREATED_IDS.clear();
    }
}
```

---

## 场景测试 vs 独立测试对比

| 维度 | 独立测试（XxxServiceTest）| 场景测试（XxxScenarioTest）|
|------|--------------------------|--------------------------|
| 目标 | 每个方法隔离验证 | 业务流程串联验证 |
| 依赖 | 各测试独立 | 上一步的输出是下一步的输入 |
| 数据 | 使用已有数据或最小构造 | 自建 → 修改 → 清理 |
| 失败影响 | 互不影响 | s01 失败则 s02/s03 跳过 |
| 适用场景 | 每次 CI 跑 | 功能验收、联调确认 |

---

## 与 Suite 的关系

- **场景测试不放入 Suite** — 有数据副作用，不适合与烟雾测试混跑
- Suite 放无副作用的查询/枚举类测试
- 场景测试单独运行：`mvn test -Dtest=XxxScenarioTest`
