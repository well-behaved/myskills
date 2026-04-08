# 项目架构与模块规范

## 1. 项目整体架构

```
apm-backend/                          # 根 POM（Maven 多模块）
├── apm-framework/                    # 公共框架层（跨模块复用的 Starter）
│   ├── apm-starter-log/              # 操作日志（AOP 切面）
│   ├── apm-starter-rate-limit/       # 接口限流（Lua + Redis）
│   └── apm-starter-interface-dock/   # 接口对接日志追踪
├── apm-module-system/                # 系统管理模块（用户/角色/部门/租户）
│   ├── apm-module-system-api/        # Feign 接口定义
│   └── apm-module-system-biz/        # 业务实现
├── apm-module-operations/            # 核心业务模块（设备/维保/点检/库存等）
│   ├── apm-module-operations-api/    # Feign 接口定义
│   └── apm-module-operations-biz/    # 业务实现
├── apm-module-log/                   # 审计日志模块
├── apm-module-quartz/                # 定时任务模块
├── apm-module-gateway/               # API 网关
├── apm-module-file/                  # 文件存储服务
└── lib/                              # 外部自研 SDK（system scope jar）
    ├── ops-common-1.0-SNAPSHOT.jar
    ├── ops-starter-security-1.0-SNAPSHOT.jar
    ├── ops-starter-mybatis-1.0-SNAPSHOT.jar
    ├── ops-starter-data-privilege-1.0-SNAPSHOT.jar
    ├── ops-starter-tenant-1.0-SNAPSHOT.jar
    ├── ops-starter-i18n-1.0-SNAPSHOT.jar
    └── ops-starter-mqtt-1.0-SNAPSHOT.jar
```

## 2. 模块拆分规则

### 2.1 api / biz 拆分（核心规则）

每个业务模块拆为 `*-api` 和 `*-biz` 两个子模块：

| 子模块 | 职责 | 可包含的内容 | 禁止包含 |
|--------|------|-------------|---------|
| `*-api` | 跨模块 RPC 接口定义 | Feign 接口、Fallback、RPC 用 DTO/Param/Enum | Controller、ServiceImpl、Mapper、Entity |
| `*-biz` | 业务实现 | Controller、Service、ServiceImpl、Mapper、Entity、DTO、Param | — |

**Review 要点**：
- ✅ `*-api` 中只有接口和数据契约，不含实现
- ❌ 在 `*-api` 中写 ServiceImpl 或直接访问数据库

### 2.2 模块间依赖方向

```
gateway → (不依赖其他业务模块，独立网关)
quartz → operations-api, system-api  （通过 Feign 调用）
operations-biz → system-api          （通过 Feign 调用 system 服务）
system-biz → (不依赖 operations)
log-biz → (独立运行)
file-biz → (独立运行)
所有 *-biz → apm-framework           （使用公共 Starter）
所有模块 → lib/ops-*                  （使用自研 SDK）
```

**Review 要点**：
- ✅ 跨模块调用必须通过 `*-api` 的 Feign 接口
- ❌ `*-biz` 模块之间直接依赖（如 operations-biz 直接 import system-biz 的类）
- ❌ 循环依赖

## 3. 包结构规范（feature-first + layer-second）

```
cn.sottop.{module}                    # 模块根包
├── {feature}/                        # 功能域（如 user, dept, spotcheck, maintenance）
│   ├── controller/                   # REST 控制器
│   │   └── XxxController.java
│   ├── service/                      # 服务接口
│   │   ├── XxxService.java
│   │   └── impl/                     # 服务实现
│   │       └── XxxServiceImpl.java
│   ├── mapper/                       # MyBatis Mapper 接口
│   │   └── XxxMapper.java
│   ├── entity/                       # 数据库实体
│   │   └── XxxEntity.java
│   ├── dto/                          # 数据传输对象（输出）
│   │   ├── XxxDTO.java
│   │   └── XxxExcelDTO.java
│   ├── param/                        # 请求参数对象（输入）
│   │   ├── XxxFindParam.java
│   │   └── XxxSaveParam.java
│   └── api/                          # 仅 *-api 模块使用
│       ├── XxxApi.java               # Feign 接口
│       ├── fallback/
│       │   └── XxxFallback.java
│       ├── dto/                      # RPC 用 DTO
│       └── param/                    # RPC 用 Param
```

**Review 要点**：
- ✅ 按功能域划分顶层包，而非按层划分
- ✅ 每个功能域下按层组织子包
- ❌ 在 controller/ 包下放 Service 类
- ❌ 跨功能域直接 import 内部实现（应通过 Service 接口调用）

## 4. 分层职责（严格边界）

| 层 | 类后缀 | 职责 | 允许的操作 | 禁止的操作 |
|----|--------|------|-----------|-----------|
| Controller | `XxxController` | HTTP 入口、参数接收、返回值包装 | 调用 Service、包装 `CommonResult` | 写业务逻辑、直接调 Mapper、事务管理 |
| Service 接口 | `XxxService` | 业务契约定义 | 定义方法签名、声明校验组 | 包含实现 |
| Service 实现 | `XxxServiceImpl` | 业务逻辑、事务管理 | 调用 Mapper、Bean 转换、校验、RPC | 直接写 SQL、操作 HTTP Request/Response |
| Mapper | `XxxMapper` | 数据访问抽象 | 继承 BaseMapper、自定义 SQL | 包含业务逻辑 |
| Entity | `XxxEntity` | 数据库映射 | 字段定义、表映射注解 | 包含业务方法 |
| DTO | `XxxDTO` | 输出数据承载 | 字段定义、展示用附加字段 | 包含业务逻辑 |
| Param | `XxxFindParam` / `XxxSaveParam` | 输入参数承载 | 字段定义、校验注解、@Query 注解 | 包含业务逻辑 |

## 5. Feign RPC 调用规范

### 5.1 接口定义（在 *-api 模块）

```java
@FeignClient(name = ServerNameConstant.OPERATIONS_SERVER_NAME, fallback = XxxFallback.class)
public interface XxxApi {
    String prefix = "/xxxDomain";

    @GetMapping(prefix + "/methodName")
    ReturnType methodName(@RequestParam("param") Type param);
}
```

### 5.2 Fallback 实现（在 *-api 模块的 fallback/ 包）

```java
@Slf4j
@Component
public class XxxFallback implements XxxApi {
    @Override
    public ReturnType methodName(Type param) {
        log.error("RPC调用失败，XxxApi.methodName远程调用失败");
        return null; // 或默认值
    }
}
```

**Review 要点**：
- ✅ 每个 Feign 接口必须有对应的 Fallback
- ✅ Fallback 中必须有 `log.error` 记录失败
- ✅ 使用 `ServerNameConstant` 定义服务名
- ❌ Feign 接口放在 `*-biz` 模块

## 6. 配置管理规范

### 6.1 配置文件结构

```
src/main/resources/
├── bootstrap.yaml           # Nacos 连接配置（按 profile 区分）
├── application.yaml         # 本地配置（最小化，大部分在 Nacos）
├── logback/                 # 日志配置
│   ├── xiaomi.xml           # 生产环境
│   └── xiaomiTest.xml       # 测试环境
├── mapper/                  # MyBatis XML 映射文件
│   └── {feature}/
│       └── XxxMapper.xml
└── generator/               # 代码生成器配置
```

### 6.2 配置注入方式

```java
// ✅ 正确：使用 @Value 注入，提供默认值
@Value("${system.user.defaultPassword:admin123}")
private String defaultPassword;

// ✅ 正确：动态刷新配置
@RefreshScope
@Service
public class XxxServiceImpl { ... }

// ❌ 错误：硬编码配置值
private String password = "admin123";
```

## 7. 外部 SDK 使用规范

`lib/` 目录下的自研 SDK 以 system scope 引入：

| SDK | 职责 | 提供的关键类/注解 |
|-----|------|-----------------|
| `ops-common` | 通用工具 | `CommonResult`, `CommonException`, `ErrorCodeConstant`, `BaseEntity`, `BaseDTO` |
| `ops-starter-security` | 认证安全 | `SecurityUtils.getUserId()`, `SecurityUtils.getUserDetail()` |
| `ops-starter-mybatis` | MyBatis 增强 | `MybatisQueryHelper`, `@Query`, `BaseFindParam`, `BaseSaveParam` |
| `ops-starter-data-privilege` | 数据权限 | `@DataPrivilege` |
| `ops-starter-tenant` | 多租户 | `TenantBaseEntity`, `TenantContextHolder` |
| `ops-starter-i18n` | 国际化 | `ErrorCodeConstant`（i18n 错误码） |
| `ops-starter-mqtt` | MQTT 通信 | MQTT 客户端封装 |

**Review 要点**：
- ✅ 使用 SDK 提供的基类和工具方法
- ❌ 重复造轮子（如自己写分页工具、自己写统一返回包装）
