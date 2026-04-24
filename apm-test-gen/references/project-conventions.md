# 项目测试约定

从现有测试代码中提炼的项目特有规范。生成测试时必须严格遵循。

## 项目架构

- **构建工具**：Maven 多模块
- **框架**：Spring Boot + MyBatis-Plus
- **认证**：Sa-Token（非 Spring Security）
- **多租户**：`TenantContextHolder.setTenantId(1L)` 线程级上下文
- **测试框架**：JUnit 5（`spring-boot-starter-test`），部分使用 Mockito
- **Java 版本**：**JDK 1.8** — 禁止使用任何 Java 9+ 语法

## 模块结构

```
apm-backend/
├── apm-framework/           # 框架层（工具类、starter）
│   ├── apm-starter-log/     # 日志组件
│   └── ...
├── apm-module-operations/   # 业务模块（主要测试集中在此）
│   ├── apm-module-operations-api/
│   └── apm-module-operations-biz/
│       └── src/test/java/cn/sottop/operations/
│           ├── BaseTest.java           # 集成测试基类
│           └── {domain}/service/       # 各业务域测试
├── apm-module-system/
├── apm-module-log/
└── ...
```

## 测试文件位置

测试类放在 `src/test/java/` 下，与源码**相同包路径**：

```
源码：src/main/java/cn/sottop/operations/mold/service/OpsMoldStokeService.java
测试：src/test/java/cn/sottop/operations/mold/service/OpsMoldStokeServiceTest.java
```

## 命名约定

| 项目 | 约定 |
|------|------|
| 测试类名 | `{源类名}Test` 或 `{源类名}LogicTest`（逻辑测试） |
| 测试方法名 | **驼峰命名**，与被测方法同名（`findPage`）或加场景后缀（`findPageWithBizModule`） |
| 类访问级别 | 包级别（无 `public`），除非继承 BaseTest |
| 注解 | 类级别 `@Slf4j`（可选），`@SpringBootTest` 或 `@ExtendWith(MockitoExtension.class)` |

### 方法命名示例

```java
// ✅ 正确：驼峰命名
void findPage() { }
void findPageWithBizModule() { }
void findChangeLogByIdsMajorCodeTranslated() { }
void saveNullParamThrows() { }
void getByIdNotExists() { }

// ✅ 正确：Suite / 场景测试有序前缀，仍用驼峰
void s01EquipmentTypeAdd() { }
void s02EquipmentTypeUpdate() { }
void s03EquipmentTypeFindPage() { }

// ❌ 错误：下划线分隔（不符合项目规范）
void findPage_withBizModule() { }
void s01_equipmentType_add() { }
void s01_equipment_type_findPage() { }
```

## 依赖注入

```java
// ✅ 项目用 @Resource
@Resource
private XxxService xxxService;

// ❌ 不用 @Autowired
@Autowired
private XxxService xxxService;
```

## JDK 1.8 兼容性约束

项目使用 **JDK 1.8**，生成的测试代码必须严格遵守以下规则：

### 禁止使用的语法（Java 10+）

```java
// ❌ 禁止 var —— Java 10 才引入，JDK 1.8 编译报错
var list = service.findList(param);
var page = service.findPage(param);
var map  = service.findMap(ids);
```

### 必须显式声明类型

```java
// ✅ 必须写出完整类型
List<OpsEquipmentTypeDTO> list = service.findList(param);
IPage<OpsEquipmentTypeDTO> page = service.findPage(param);
Map<Long, OpsEquipmentTypeDTO> map = service.findMap(ids);
```

### 常用返回类型对照

| 方法名模式 | 返回类型 |
|-----------|---------|
| `findPage(...)` | `IPage<XxxDTO>` |
| `findList(...)` | `List<XxxDTO>` |
| `findMap(...)` | `Map<Long, XxxDTO>` 或 `Map<Long, List<XxxDTO>>` |
| `getById(...)` | `XxxDTO` |
| `getRelation()` | 具体 DTO 类型（查接口定义） |
| `getTree(...)` | `List<XxxTreeDTO>` |
| `findBizNameOptions(...)` | `List<XxxOptionDTO>` |

必要的 import（按需添加）：
```java
import com.baomidou.mybatisplus.core.metadata.IPage;
import java.util.List;
import java.util.Map;
```

## 断言风格

```java
// ✅ JUnit 5 原生断言
import static org.junit.jupiter.api.Assertions.*;

assertDoesNotThrow(() -> service.doSomething(param));
assertEquals(expected, actual);
assertNotNull(result);
assertTrue(condition);
assertThrows(BusinessException.class, () -> service.invalid(param));

// ❌ 不用 AssertJ
assertThat(result).isNotNull();
```

## 租户与登录上下文

### 方式 1：@BeforeAll 静态初始化（多数测试采用）

```java
@BeforeAll
static void beforeAll() {
    log.info("初始化tenantId");
    TenantContextHolder.setTenantId(1L);
}
```

### 方式 2：继承 BaseTest（需要 Sa-Token 登录模拟时）

```java
class XxxServiceTest extends BaseTest {
    @BeforeEach
    void setUp() {
        loginWithToken("真实Token字符串");
    }
}
```

> BaseTest 已包含 `@SpringBootTest(webEnvironment = RANDOM_PORT)`，子类不需要重复标注。

### 方式 3：SatokenHolder 禁用登录校验（Feign 调用场景）

```java
@BeforeAll
static void beforeAll() {
    TenantContextHolder.setTenantId(1L);
    SatokenHolder.setTenantId(1L);
    SatokenHolder.setUserLoginEnabled(Boolean.FALSE);
    RequestContextHolder.setRequestAttributes(null);
}
```

## 选择上下文方式的决策

```
被测方法内部是否校验登录态？
├─ 否 → 方式 1（仅设租户 ID）
├─ 是，但可以绕过 → 方式 3（禁用登录校验）
└─ 是，必须模拟真实登录 → 方式 2（继承 BaseTest）
```

## 测试数据

- 直接硬编码测试数据（项目中没有 test fixtures / builder 模式）
- 对于复杂参数，用 JSON 字符串 + `JSONUtil.toBean()` 构造：

```java
String paramJson = "{\"name\":\"测试\",\"model\":\"型号A\"}";
XxxSaveParam param = JSONUtil.toBean(paramJson, XxxSaveParam.class);
```

- 分页参数统一设置：

```java
param.setCurrent(1L);
param.setSize(100L);
```

## 日志

- 使用 `@Slf4j` + `log.info()` 输出测试过程信息
- 查询结果用 `JSONUtil.toJsonStr()` 序列化后打印

```java
log.info("查询结果:{}", JSONUtil.toJsonStr(page));
```

## 事务约定

项目通过 `TransactionalAopConfig` 对所有 `cn.sottop..*.service..*.*(..)` 统一织入声明式事务：

- 写操作（`add*`/`update*`/`delete*`/`batch*` 等）→ `REQUIRED`，异常自动回滚
- 读操作（`find*`/`get*`）→ `SUPPORTS` 只读

**测试类不加 `@Transactional`**（项目所有测试类均无此注解）：
- 每个 `@Test` 方法各自提交事务，步骤间数据可见
- 加了反而导致整体回滚，场景测试结果异常

## @AfterEach 清理

- 集成测试继承 BaseTest 时自动清理上下文
- 独立测试类如果设了 `TenantContextHolder`，应在 `@AfterEach` 或 `@AfterAll` 中清理：

```java
@AfterAll
static void afterAll() {
    TenantContextHolder.clear();
}
```

## 不测试的类型

| 类型 | 原因 |
|------|------|
| Entity | 纯数据类，getter/setter 无逻辑 |
| DTO / VO | 数据传输对象 |
| Param | 请求参数对象 |
| Enum / Constant | 枚举和常量定义 |
| Config | 配置类 |
| Mapper 接口 | MyBatis-Plus 自动实现，测通过 Service 间接覆盖 |

## 禁止的测试行为

```java
// ❌ 禁止：不传任何条件直接查全表
service.findPage(new XxxFindParam());               // param 无任何过滤条件
service.findList(new XxxFindParam());

// ❌ 禁止：更新时不设 id，导致全表更新
XxxSaveParam param = new XxxSaveParam();
param.setStatus("1");                               // 只设业务字段，不设 id
service.update(param);                              // 全量更新！

// ❌ 禁止：删除时传空列表/null，可能触发全表删除
service.deleteBatch(Collections.emptyList());
service.deleteBatch(null);

// ❌ 禁止：delete 方法不传 id
service.delete(null);
```

**正确做法：写操作必须明确操作目标**

```java
// ✅ 更新：必须设置 id
XxxSaveParam param = new XxxSaveParam();
param.setId(testEntityId);     // 必须有 id
param.setStatus("1");
service.update(param);

// ✅ 删除：必须传具体 id 列表
service.deleteBatch(Collections.singletonList(testEntityId));

// ✅ 查询：分页加合理的 size 上限
XxxFindParam param = new XxxFindParam();
param.setCurrent(1L);
param.setSize(20L);            // 不用 100+，避免全表扫描
```
