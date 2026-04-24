# 测试套件模式

每次批量生成测试类（≥2 个）后，同步在对应模块的 `suite` 包下生成一个套件类，方便一键运行所有测试。

## ⚠️ 重要：不要使用 @Suite / @SelectClasses

`@Suite` + `@SelectClasses` 来自 `junit-platform-suite`，**该 jar 不在 Spring Boot 2.x BOM 中**，
需要单独引入依赖，而项目内网环境无法拉取新 jar，**会导致编译报错**。

```java
// ❌ 禁止使用，编译失败（缺少 junit-platform-suite 依赖）
import org.junit.platform.suite.api.Suite;
import org.junit.platform.suite.api.SelectClasses;

@Suite
@SelectClasses({ XxxTest.class, YyyTest.class })
class XxxSuite {}
```

## ✅ 正确方案：@SpringBootTest 单类套件

套件本身就是一个普通的 `@SpringBootTest` 测试类，把所有测试方法直接写在里面。

**优点：**
- 零额外依赖，项目开箱即用
- 只启动一次 Spring 容器，比多个独立测试类跑得更快
- IDEA 右键即可运行，`mvn test -Dtest=XxxSuite` 同样支持

## 套件文件位置

```
src/test/java/{模块根包}/suite/{功能名}Suite.java
```

| 模块 | 根包 | 示例路径 |
|------|------|----------|
| apm-module-operations-biz | `cn.sottop.operations` | `cn/sottop/operations/suite/ChangeLogSuite.java` |
| apm-module-log-biz | `cn.sottop.log` | `cn/sottop/log/suite/LogChangeSuite.java` |
| apm-module-system-biz | `cn.sottop.system` | `cn/sottop/system/suite/XxxSuite.java` |
| apm-framework/apm-starter-log | `cn.sottop.framework.log` | `cn/sottop/framework/log/suite/XxxSuite.java` |

## 命名规则

- 套件类名：`{功能名}Suite`，取本次变更的业务主题
- 例：`ChangeLogSuite`、`MaintainOrderSuite`、`InventorySuite`
- **不要用** `AllTest`、`TestAll` 等无意义通用名

## 套件模板

```java
package {模块根包}.suite;

import cn.sottop.framework.tenant.context.TenantContextHolder;
// 按需 import 各 Service / Param 类
import lombok.extern.slf4j.Slf4j;
import org.junit.jupiter.api.*;
import org.springframework.boot.test.context.SpringBootTest;

import javax.annotation.Resource;
import java.util.Arrays;
import java.util.Collections;

import static org.junit.jupiter.api.Assertions.*;

/**
 * {功能描述}测试套件（{分支名} 分支新增）
 * 一次运行覆盖：{分组A} + {分组B}，共 N 个测试方法
 *
 * 运行方式：右键此类 → Run '{SuiteName}'
 */
@SpringBootTest
@Slf4j
@TestMethodOrder(MethodOrderer.MethodName.class)   // 按方法名排序，s01_/s02_... 保证顺序
class {SuiteName} {

    // ===== 注入被测 Service（按分组排列）=====
    @Resource private XxxService xxxService;
    @Resource private YyyService yyyService;

    @BeforeAll
    static void beforeAll() {
        TenantContextHolder.setTenantId(1L);
    }

    @AfterAll
    static void afterAll() {
        TenantContextHolder.clear();
    }

    // ==================== {分组A 描述} ====================

    @Test void s01{功能}{场景}() {
        // 测试逻辑
    }

    @Test void s02{功能}{场景}() {
        // 测试逻辑
    }

    // ==================== {分组B 描述} ====================

    @Test void s03{功能}{场景}() {
        // 测试逻辑
    }
}
```

### 方法命名规范

- 前缀 `s01`、`s02` 确保 `@TestMethodOrder(MethodName)` 按预期顺序执行
- 命名格式：**驼峰**，`s{序号}{业务对象}{场景}` —— 例：`s01EquipmentTypeFindPage`、`s05FaultLocationSnapshotNullIds`
- **禁止下划线分隔**，`s01_equipmentType_findPage` 是错误写法

## 跨模块处理

如果本次变更跨多个模块（如同时涉及 operations + log），每个模块**分别建一个套件**：

```
operations 模块 → cn.sottop.operations.suite.ChangeLogSuite
log 模块        → cn.sottop.log.suite.LogChangeSuite
```

**不要把跨模块的测试类混在同一个套件里**（会导致 Spring 上下文冲突）。

## pom 检查

生成套件前，检查目标模块的 pom 是否包含 `spring-boot-starter-test`：

```bash
grep "spring-boot-starter-test" {module}/pom.xml
```

若**没有**，在 `<dependencies>` 中补充：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```

## 何时跳过套件

只生成了 1 个测试类时，不需要建套件（直接运行单个测试类即可）。

## 真实示例

```java
package cn.sottop.operations.suite;

import cn.sottop.framework.tenant.context.TenantContextHolder;
import cn.sottop.operations.baseData.changelog.EquipmentTypeChangeLogQueryService;
import cn.sottop.operations.baseData.changelog.FaultLocationChangeLogQueryService;
import cn.sottop.operations.baseData.param.OpsEquipmentTypeFindParam;
import cn.sottop.operations.baseData.service.OpsEquipmentTypeService;
import lombok.extern.slf4j.Slf4j;
import org.junit.jupiter.api.*;
import org.springframework.boot.test.context.SpringBootTest;

import javax.annotation.Resource;
import java.util.Arrays;
import java.util.Collections;

import static org.junit.jupiter.api.Assertions.*;

/**
 * 变更日志功能测试套件（feature/xue/202604 分支新增）
 * 一次运行覆盖：ChangeLogQueryService + ServiceImpl，共 N 个测试方法
 *
 * 运行方式：右键此类 → Run 'ChangeLogSuite'
 */
@SpringBootTest
@Slf4j
@TestMethodOrder(MethodOrderer.MethodName.class)
class ChangeLogSuite {

    @Resource private EquipmentTypeChangeLogQueryService equipmentTypeChangeLogQueryService;
    @Resource private FaultLocationChangeLogQueryService faultLocationChangeLogQueryService;
    @Resource private OpsEquipmentTypeService opsEquipmentTypeService;

    @BeforeAll
    static void beforeAll() {
        TenantContextHolder.setTenantId(1L);
    }

    @AfterAll
    static void afterAll() {
        TenantContextHolder.clear();
    }

    // ==================== ChangeLogQueryService ====================

    @Test void s01EquipmentTypeSnapshotEmptyIds() {
        assertTrue(equipmentTypeChangeLogQueryService.findChangeLogByIds(Collections.emptyList()).isEmpty());
    }

    @Test void s02EquipmentTypeSnapshotNullIds() {
        assertTrue(equipmentTypeChangeLogQueryService.findChangeLogByIds(null).isEmpty());
    }

    @Test void s03EquipmentTypeSnapshotQuery() {
        assertDoesNotThrow(() -> equipmentTypeChangeLogQueryService.findChangeLogByIds(Arrays.asList(1L, 2L)));
    }

    @Test void s04FaultLocationSnapshotEmptyIds() {
        assertTrue(faultLocationChangeLogQueryService.findChangeLogByIds(Collections.emptyList()).isEmpty());
    }

    // ==================== ServiceImpl ====================

    @Test void s05EquipmentTypeFindPage() {
        OpsEquipmentTypeFindParam p = new OpsEquipmentTypeFindParam();
        p.setCurrent(1L); p.setSize(20L);
        assertDoesNotThrow(() -> assertNotNull(opsEquipmentTypeService.findPage(p)));
    }
}
```
