# Spring Boot 分层测试模板

根据被测类的类型，选择对应的测试模板。

## 决策树

```
被测类有 Spring 注解（@Service/@Component/@RestController）？
├─ 是 → 有复杂依赖注入（3+ 个 @Resource 字段）？
│   ├─ 是 → 被测方法是否可以不启动容器测试？
│   │   ├─ 是（纯逻辑，不查库）→ 模板 B：Mockito 单元测试
│   │   └─ 否（需要数据库/中间件）→ 模板 A：Service 集成测试
│   └─ 否（依赖少）→ 模板 A：Service 集成测试
└─ 否（无注解）→ 模板 C：纯单元测试
```

## 模板 A：Service 集成测试（@SpringBootTest）

适用：Service / ServiceImpl / Manager / Handler 等需要 Spring 容器的类。

```java
package {与源码相同的包路径};

import cn.sottop.framework.tenant.context.TenantContextHolder;
import lombok.extern.slf4j.Slf4j;
import static org.junit.jupiter.api.Assertions.*;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.AfterAll;
import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;

import javax.annotation.Resource;
// 按需导入其他依赖

@SpringBootTest
@Slf4j
class {ClassName}Test {

    @Resource
    private {ClassName} {fieldName};

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
     * 测试 {methodName} - 正常流程
     */
    @Test
    void {methodName}() {
        // Arrange - 构造参数
        {ParamType} param = new {ParamType}();
        // 设置必要字段...

        // Act & Assert
        assertDoesNotThrow(() -> {fieldName}.{methodName}(param));
    }

    /**
     * 测试 {methodName} - {边界/异常场景描述}
     */
    @Test
    void {methodName}_{scenario}() {
        // Arrange - 构造边界参数
        // Act & Assert
    }
}
```

### 模板 A 变体：需要登录态

当被测方法内部校验 Sa-Token 登录态时，继承 BaseTest：

```java
package {包路径};

import cn.sottop.operations.BaseTest;
import lombok.extern.slf4j.Slf4j;
import static org.junit.jupiter.api.Assertions.*;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import javax.annotation.Resource;

@Slf4j
class {ClassName}Test extends BaseTest {

    @Resource
    private {ClassName} {fieldName};

    @BeforeEach
    void setUp() {
        // 如需模拟登录：loginWithToken("token字符串");
        // BaseTest 已含 @SpringBootTest 和 @AfterEach 清理
    }

    @Test
    void {methodName}() {
        // ...
    }
}
```

### 模板 A 变体：Feign 调用场景

当被测方法通过 Feign 调用其他服务时：

```java
@BeforeAll
static void beforeAll() {
    TenantContextHolder.setTenantId(1L);
    SatokenHolder.setTenantId(1L);
    SatokenHolder.setUserLoginEnabled(Boolean.FALSE);
    RequestContextHolder.setRequestAttributes(null);
}
```

## 模板 B：Mockito 单元测试

适用：ServiceImpl 中的纯逻辑方法，可以 mock 掉数据库依赖。

```java
package {与源码相同的包路径};

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;
import static org.mockito.ArgumentMatchers.*;

@ExtendWith(MockitoExtension.class)
class {ClassName}Test {

    @InjectMocks
    private {ImplClassName} {fieldName};

    @Mock
    private {DependencyType} {dependencyField};

    @BeforeEach
    void setUp() {
        // 公共 mock 行为设置
    }

    /**
     * 测试 {methodName} - 正常流程
     */
    @Test
    void {methodName}() {
        // Arrange
        when({dependencyField}.{method}(any())).thenReturn({mockData});

        // Act
        {ReturnType} result = {fieldName}.{methodName}({args});

        // Assert
        assertNotNull(result);
        verify({dependencyField}).{method}(any());
    }
}
```

### 何时选 Mockito vs 集成测试

| 信号 | 选择 |
|------|------|
| 方法内只做计算/转换，不查库 | Mockito |
| 方法内有复杂 if/else 分支逻辑 | Mockito（便于覆盖各分支） |
| 方法内调 Mapper 查库且逻辑简单 | 集成测试（mock 成本高于启动容器） |
| 方法内调多个 Service 协作 | 集成测试 |
| 需要验证数据库实际写入结果 | 集成测试 |
| 测试私有方法的逻辑 | Mockito + 反射（参考项目已有模式） |

## 模板 C：纯单元测试

适用：工具类（Util）、枚举行为、注解验证等无 Spring 依赖的类。

```java
package {与源码相同的包路径};

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class {ClassName}Test {

    /**
     * 测试 {methodName} - {场景}
     */
    @Test
    void {methodName}_{scenario}() {
        // Arrange
        {InputType} input = {构造输入};

        // Act
        {ReturnType} result = {ClassName}.{methodName}(input);

        // Assert
        assertEquals({expected}, result);
    }

    /**
     * 测试 {methodName} - 空输入
     */
    @Test
    void {methodName}_null_input() {
        // 边界 case
        assertDoesNotThrow(() -> {ClassName}.{methodName}(null));
        // 或
        assertThrows(IllegalArgumentException.class, () -> {ClassName}.{methodName}(null));
    }
}
```

## 测试方法覆盖策略

对每个公开方法，生成的测试应覆盖：

| 方法类型 | 必须覆盖 | 建议覆盖 |
|----------|----------|----------|
| 查询方法（find/get/list/page） | 正常查询返回结果 | 空结果、参数缺失 |
| 新增方法（add/create/save） | 正常新增 | 重复数据、必填字段缺失 |
| 更新方法（update/modify） | 正常更新 | 数据不存在 |
| 删除方法（delete/remove） | 正常删除 | 数据不存在 |
| 定时任务方法（execute*） | 正常执行不报错 | 带参数执行 |
| 工具方法（静态方法） | 正常输入输出 | null 输入、边界值 |

## 断言选择指南

| 场景 | 断言 |
|------|------|
| void 方法，验证不抛异常 | `assertDoesNotThrow(() -> ...)` |
| 有返回值，验证非空 | `assertNotNull(result)` |
| 有返回值，验证具体值 | `assertEquals(expected, actual)` |
| 布尔判断 | `assertTrue(condition)` / `assertFalse(condition)` |
| 验证抛出特定异常 | `assertThrows(XxxException.class, () -> ...)` |
| 分页查询 | `assertNotNull(page)` + `log.info(JSONUtil.toJsonStr(page))` |
| 多个断言组合 | 直接顺序写，不用 `assertAll` |
