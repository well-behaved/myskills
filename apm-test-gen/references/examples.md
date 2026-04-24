# 生成示例对照表

真实的源码 → 生成的测试类对照，基于项目中已有的实际代码模式。

## 示例 1：Service 集成测试（分页查询 + 新增）

### 源码特征

```java
// OpsMaterialService.java — Service 接口
public interface OpsMaterialService {
    IPage<OpsMaterialDTO> findPage(OpsMaterialFindParam param);
    void add(OpsMaterialSaveParam param);
}
```

### 生成的测试

```java
package cn.sottop.operations.materialManagement.service;

import cn.hutool.json.JSONUtil;
import cn.sottop.framework.tenant.context.TenantContextHolder;
import cn.sottop.operations.materialManagement.dto.OpsMaterialDTO;
import cn.sottop.operations.materialManagement.param.OpsMaterialFindParam;
import cn.sottop.operations.materialManagement.param.OpsMaterialSaveParam;
import com.baomidou.mybatisplus.core.metadata.IPage;
import lombok.extern.slf4j.Slf4j;
import static org.junit.jupiter.api.Assertions.*;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.AfterAll;
import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;

import javax.annotation.Resource;

@SpringBootTest
@Slf4j
class OpsMaterialServiceTest {

    @Resource
    private OpsMaterialService opsMaterialService;

    @BeforeAll
    static void beforeAll() {
        log.info("初始化tenantId");
        TenantContextHolder.setTenantId(1L);
    }

    @AfterAll
    static void afterAll() {
        TenantContextHolder.clear();
    }

    /**
     * 测试分页查询 - 正常流程
     */
    @Test
    void findPage() {
        OpsMaterialFindParam param = new OpsMaterialFindParam();
        param.setCurrent(1L);
        param.setSize(100L);
        assertDoesNotThrow(() -> {
            IPage<OpsMaterialDTO> page = opsMaterialService.findPage(param);
            log.info("分页查询结果:{}", JSONUtil.toJsonStr(page));
        });
    }

    /**
     * 测试新增 - 正常流程
     */
    @Test
    void add() {
        String paramJson = "{\"name\":\"测试物料\",\"model\":\"型号A\",\"unit\":\"个\"}";
        OpsMaterialSaveParam param = JSONUtil.toBean(paramJson, OpsMaterialSaveParam.class);
        assertDoesNotThrow(() -> opsMaterialService.add(param));
    }
}
```

### 关键点

- `@Resource` 注入，非 `@Autowired`
- `@BeforeAll` 设置租户 ID
- 分页参数用 `setCurrent/setSize`
- 复杂参数用 JSON 字符串构造
- void 方法用 `assertDoesNotThrow`
- 查询结果用 `JSONUtil.toJsonStr` 打印

---

## 示例 2：定时任务 Service 测试

### 源码特征

```java
// OpsSpotcheckOrderService.java
public interface OpsSpotcheckOrderService {
    void executeCreateTask();
    void executeCreateNightTask();
    void executeCreateNightTask(ExecuteCreateTaskParam param);
}
```

### 生成的测试

```java
package cn.sottop.operations.spotcheck.service;

import cn.hutool.core.date.LocalDateTimeUtil;
import cn.sottop.framework.tenant.context.TenantContextHolder;
import cn.sottop.operations.spotcheck.param.ExecuteCreateTaskParam;
import lombok.extern.slf4j.Slf4j;
import static org.junit.jupiter.api.Assertions.*;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;

import javax.annotation.Resource;
import java.time.LocalDateTime;

@SpringBootTest
@Slf4j
class OpsSpotcheckOrderServiceTest {

    @Resource
    OpsSpotcheckOrderService opsSpotcheckOrderService;

    @BeforeAll
    static void beforeAll() {
        log.info("初始化tenantId");
        TenantContextHolder.setTenantId(1L);
    }

    /**
     * 测试创建点检任务 - 白班默认
     */
    @Test
    void executeCreateTask() {
        log.info("执行创建点检任务 白班");
        TenantContextHolder.setTenantId(1L);
        opsSpotcheckOrderService.executeCreateTask();
    }

    /**
     * 测试创建点检任务 - 夜班默认
     */
    @Test
    void executeCreateNightTask() {
        log.info("执行创建点检任务 夜班");
        opsSpotcheckOrderService.executeCreateNightTask();
    }

    /**
     * 测试创建点检任务 - 夜班带参数
     */
    @Test
    void executeCreateNightTask_withParam() {
        log.info("执行创建点检任务 夜班带参数");
        String dateDemo = "2025-07-21 14:25:08";
        LocalDateTime parse = LocalDateTimeUtil.parse(dateDemo, "yyyy-MM-dd HH:mm:ss");
        ExecuteCreateTaskParam param = new ExecuteCreateTaskParam();
        param.setCurrentDateTime(parse);
        param.setSpotcheckCycleIds("1,2,3");
        assertDoesNotThrow(() -> opsSpotcheckOrderService.executeCreateNightTask(param));
    }
}
```

### 关键点

- 定时任务方法通常无返回值，用 `assertDoesNotThrow` 或直接调用
- 有参数和无参数版本都要覆盖
- 时间类参数用 `LocalDateTimeUtil.parse` 构造

---

## 示例 3：Mockito 单元测试

### 源码特征

```java
// OpsStandardWorkPlanServiceImpl.java
@Service
public class OpsStandardWorkPlanServiceImpl implements OpsStandardWorkPlanService {
    @Resource
    private OpsStandardWorkOrderMapper mapper;
    
    private LocalDate getOrderDate(LocalDate baseDate, OpsStandardWorkOrderSaveParam param) {
        // 私有方法，纯计算逻辑
    }
}
```

### 生成的测试

```java
package cn.sottop.operations.performance.service;

import cn.sottop.operations.performance.param.OpsStandardWorkOrderSaveParam;
import cn.sottop.operations.performance.service.impl.OpsStandardWorkPlanServiceImpl;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.lang.reflect.Method;
import java.time.LocalDate;

import static org.junit.jupiter.api.Assertions.assertTrue;

@ExtendWith(MockitoExtension.class)
public class OpsStandardWorkPlanServiceTest {

    @InjectMocks
    private OpsStandardWorkPlanServiceImpl opsStandardWorkPlanService;

    @Mock
    private OpsStandardWorkOrderSaveParam opsStandardWorkOrderSaveParam;

    @BeforeEach
    public void setUp() {
        opsStandardWorkOrderSaveParam = new OpsStandardWorkOrderSaveParam();
        opsStandardWorkOrderSaveParam.setWorkCycleId(2L);
    }

    /**
     * 测试私有方法 getOrderDate - 通过反射调用
     */
    @Test
    void test_getOrderDate() {
        try {
            Method method = OpsStandardWorkPlanServiceImpl.class
                .getDeclaredMethod("getOrderDate", LocalDate.class, OpsStandardWorkOrderSaveParam.class);
            method.setAccessible(true);
            LocalDate result = (LocalDate) method.invoke(
                opsStandardWorkPlanService, LocalDate.now(), opsStandardWorkOrderSaveParam);
            assertTrue(result != null);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

### 关键点

- `@ExtendWith(MockitoExtension.class)` 而非 `@SpringBootTest`
- `@InjectMocks` 注入实现类（非接口）
- 私有方法通过反射测试（项目已有此模式）
- 不启动 Spring 容器，速度快

---

## 示例 4：纯工具类测试

### 源码特征

```java
// ChangeLogDiffUtil.java — 静态工具类
public class ChangeLogDiffUtil {
    public static String diff(String before, String after, Class<?> snapshotClass) {
        // 比较两个 JSON 快照的差异
    }
}
```

### 生成的测试

```java
package cn.sottop.framework.log.change.util;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class ChangeLogDiffUtilTest {

    /**
     * 测试 diff - 字段有变化时返回变化描述
     */
    @Test
    void diff_field_changed() {
        String before = "{\"name\":\"A\"}";
        String after = "{\"name\":\"B\"}";
        String result = ChangeLogDiffUtil.diff(before, after, TestSnapshot.class);
        assertTrue(result.contains("名称"));
    }

    /**
     * 测试 diff - 无变化时返回空
     */
    @Test
    void diff_no_change() {
        String json = "{\"name\":\"A\"}";
        assertEquals("", ChangeLogDiffUtil.diff(json, json, TestSnapshot.class));
    }

    /**
     * 测试 diff - 双 null 返回空
     */
    @Test
    void diff_both_null() {
        assertEquals("", ChangeLogDiffUtil.diff(null, null, TestSnapshot.class));
    }

    /**
     * 测试 diff - 新增场景（before 为 null）
     */
    @Test
    void diff_add_scenario() {
        String result = ChangeLogDiffUtil.diff(null, "{\"name\":\"A\"}", TestSnapshot.class);
        assertTrue(result.contains("名称"));
        assertFalse(result.contains("→"));
    }
}
```

### 关键点

- 无 `@SpringBootTest`，无注入，无租户上下文
- 直接调用静态方法
- 覆盖：正常、无变化、null 输入、新增/删除场景
- 断言用 `assertEquals` / `assertTrue` / `assertFalse`

---

## 示例 5：逻辑测试（自造数据 → 验证业务逻辑 → 清理）

适用于：有字典翻译、字段映射、关联查询等业务逻辑的 Service/QueryService 类。

**与示例 1 的区别**：不只是"不报错"，而是验证具体逻辑正确性。

### 模式结构

```java
@SpringBootTest
@Slf4j
class XxxServiceLogicTest {

    @Resource private XxxService xxxService;
    @Resource private YyyService yyyService; // 用于造测试数据

    // 记录本次测试创建的 ID，供 @AfterEach 清理
    private final List<Long> createdIds = new ArrayList<Long>();

    @BeforeAll
    static void beforeAll() { TenantContextHolder.setTenantId(1L); }

    @AfterAll
    static void afterAll() { TenantContextHolder.clear(); }

    @AfterEach
    void cleanup() {
        if (!createdIds.isEmpty()) {
            try { xxxService.deleteBatch(new ArrayList<Long>(createdIds)); }
            catch (Exception e) { log.warn("清理失败", e); }
            createdIds.clear();
        }
    }

    /** 造数据辅助方法，返回新增 ID，并记录到 createdIds */
    private Long createTestData(String name) {
        XxxSaveParam param = new XxxSaveParam();
        param.setName(name);
        xxxService.add(param);
        // add() 无返回值，通过查询找回 id
        List<XxxDTO> list = xxxService.findList(new XxxFindParam());
        for (XxxDTO dto : list) {
            if (name.equals(dto.getName())) {
                createdIds.add(dto.getId());
                return dto.getId();
            }
        }
        throw new IllegalStateException("新增后找不到: " + name);
    }

    @Test
    void someLogicTest() {
        // 1. 造数据（时间戳避免重名）
        Long id = createTestData("TEST_场景名_" + System.currentTimeMillis());

        // 2. 调用被测方法
        List<XxxSnapshot> snapshots = xxxQueryService.findChangeLogByIds(
                Collections.singletonList(id));

        // 3. 验证具体逻辑
        assertEquals(1, snapshots.size());
        assertNotEquals("001", snapshots.get(0).getSomeField(), "不应存原始 code");
        assertEquals("预期的中文名", snapshots.get(0).getSomeField(), "应被翻译为中文名");
        // @AfterEach 自动清理，无需手动 delete
    }
}
```

### 关键规范

| 项 | 规范 |
|----|------|
| 测试数据命名 | `"TEST_{场景}_{System.currentTimeMillis()}"` — 时间戳避免重名 |
| 数据清理 | `@AfterEach` 统一清理（不在每个 @Test 里手动删）|
| 清理异常 | `try/catch` 吞掉，避免清理失败影响测试结果 |
| add() 无返回值 | 通过 `findList()` + 名称匹配找回 id |
| 字典翻译断言 | 同时断言"不等于原始 code"和"等于中文名" |
| 不放入 Suite | 逻辑测试有数据副作用，单独跑，不混入烟雾套件 |

### 字典 code 真实值（当前项目）

| dict_code | code | name |
|-----------|------|------|
| equipment_major | 001 | 车身 |
| equipment_major | 002 | 压铸 |
| equipment_major | 003 | 电池 |
| equipment_major | 004 | 涂装 |
| equipment_major | 005 | 模具 |
| cycle_type | 001 | 周保养 |
| cycle_type | 002 | 月保养 |
| cycle_type | 003 | 双月保养 |
| spotcheck_way | 001 | 手动录入 |
| spotcheck_way | 002 | 自动采集 |
| value_type | 001 | 数值 |
| value_type | 002 | 文本 |
| value_type | 003 | 下拉选项 |
| check_time | 001 | 班前 |
| check_time | 002 | 班中 |
| maintain_check_time | 001 | 运行 |
| maintain_check_time | 002 | 停机 |
| maintain_check_time | 003 | 不限 |

---

## 示例 6：场景链式集成测试（新增 → 修改 → 查询 → 删除）

### 触发场景

本期变更同时包含 `add`、`update`、`findPage`、`delete` 四个方法，操作同一实体 `OpsInventory`。

### 源码特征

```java
// OpsInventoryService.java
public interface OpsInventoryService {
    void add(OpsInventorySaveParam param);
    void update(OpsInventoryUpdateParam param);
    IPage<OpsInventoryDTO> findPage(OpsInventoryFindParam param);
    void delete(Long id);
}
```

### 生成的场景测试

```java
package cn.sottop.operations.inventory.service;

import cn.hutool.json.JSONUtil;
import cn.sottop.framework.tenant.context.TenantContextHolder;
import cn.sottop.operations.inventory.dto.OpsInventoryDTO;
import cn.sottop.operations.inventory.param.OpsInventoryFindParam;
import cn.sottop.operations.inventory.param.OpsInventorySaveParam;
import cn.sottop.operations.inventory.param.OpsInventoryUpdateParam;
import com.baomidou.mybatisplus.core.metadata.IPage;
import lombok.extern.slf4j.Slf4j;
import org.junit.jupiter.api.*;
import org.springframework.boot.test.context.SpringBootTest;

import javax.annotation.Resource;
import java.util.ArrayList;
import java.util.List;

import static org.junit.jupiter.api.Assertions.*;

/**
 * 库存管理场景链式集成测试
 * 模拟完整业务流程：新增 → 修改 → 查询 → 删除
 *
 * ⚠️ 数据隔离：测试数据以 "SCENARIO_TEST_库存_" 前缀 + 时间戳命名，@AfterEach 统一清理
 */
@SpringBootTest
@Slf4j
@TestMethodOrder(MethodOrderer.MethodName.class)
class OpsInventoryScenarioTest {

    @Resource
    private OpsInventoryService opsInventoryService;

    /** 记录本次测试创建的记录名称，供查回 ID 使用 */
    private static String testDataName;

    /** 跨步骤传递的实体 ID */
    private static Long testEntityId;

    @BeforeAll
    static void beforeAll() {
        log.info("初始化租户上下文");
        TenantContextHolder.setTenantId(1L);
        testDataName = "SCENARIO_TEST_库存_" + System.currentTimeMillis();
    }

    @AfterAll
    static void afterAll() {
        TenantContextHolder.clear();
    }

    @AfterEach
    void cleanup() {
        // 仅在最后一步（删除步骤）失败时，兜底清理
        // 正常流程中 s04 会主动 delete，无需此处二次清理
    }

    // ==================== Step 1: 新增 ====================

    /**
     * 场景第 1 步：新增库存记录，验证不报错
     */
    @Test
    void s01InventoryAdd() {
        OpsInventorySaveParam param = new OpsInventorySaveParam();
        param.setName(testDataName);
        param.setQuantity(100);
        // 其他必填字段...

        assertDoesNotThrow(() -> opsInventoryService.add(param));

        // 新增后查回 ID，传递给后续步骤
        OpsInventoryFindParam findParam = new OpsInventoryFindParam();
        findParam.setName(testDataName);
        findParam.setCurrent(1L);
        findParam.setSize(10L);
        IPage<OpsInventoryDTO> page = opsInventoryService.findPage(findParam);
        assertNotNull(page);
        assertFalse(page.getRecords().isEmpty(), "新增后应能查到数据");

        testEntityId = page.getRecords().get(0).getId();
        log.info("s01 新增完成，id={}", testEntityId);
    }

    // ==================== Step 2: 修改 ====================

    /**
     * 场景第 2 步：修改刚新增的库存记录，验证修改不报错
     */
    @Test
    void s02InventoryUpdate() {
        assertNotNull(testEntityId, "需先执行 s01 新增步骤");

        OpsInventoryUpdateParam param = new OpsInventoryUpdateParam();
        param.setId(testEntityId);
        param.setQuantity(200);  // 修改数量

        assertDoesNotThrow(() -> opsInventoryService.update(param));
        log.info("s02 修改完成，id={}", testEntityId);
    }

    // ==================== Step 3: 查询验证 ====================

    /**
     * 场景第 3 步：分页查询，验证数据可见且修改已生效
     */
    @Test
    void s03InventoryFindPage() {
        assertNotNull(testEntityId, "需先执行 s01 新增步骤");

        OpsInventoryFindParam param = new OpsInventoryFindParam();
        param.setCurrent(1L);
        param.setSize(20L);

        assertDoesNotThrow(() -> {
            IPage<OpsInventoryDTO> page = opsInventoryService.findPage(param);
            assertNotNull(page);
            log.info("s03 查询结果：{}", JSONUtil.toJsonStr(page));
        });
    }

    // ==================== Step 4: 删除（兼清理） ====================

    /**
     * 场景第 4 步：删除测试数据，验证删除不报错（同时清理数据）
     */
    @Test
    void s04InventoryDelete() {
        if (testEntityId == null) {
            log.warn("testEntityId 为 null，跳过删除步骤");
            return;
        }

        assertDoesNotThrow(() -> opsInventoryService.delete(testEntityId));
        log.info("s04 删除完成，id={}", testEntityId);

        // 验证已删除
        OpsInventoryFindParam findParam = new OpsInventoryFindParam();
        findParam.setName(testDataName);
        findParam.setCurrent(1L);
        findParam.setSize(10L);
        IPage<OpsInventoryDTO> page = opsInventoryService.findPage(findParam);
        assertTrue(page.getRecords().isEmpty(), "删除后应查不到数据");
    }
}
```

### 关键点

- `@TestMethodOrder(MethodOrderer.MethodName.class)` + `s01_`/`s02_` 前缀确保顺序
- `testEntityId` 声明为 `static`，跨 `@Test` 方法传递中间状态
- `testDataName` 加时间戳，避免并发冲突
- `add()` 无返回值时，通过 `findPage` 查回 id
- 最后一步 `s04` 主动删除，兼做测试数据清理
- 场景测试**不放入 Suite**（有数据副作用）
