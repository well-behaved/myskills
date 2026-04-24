# Surefire 报告解析规范

---

## 报告文件位置

Maven Surefire 插件默认将测试结果输出到：

```
{module}/target/surefire-reports/
├── TEST-cn.sottop.operations.mold.service.OpsMoldServiceTest.xml
├── TEST-cn.sottop.operations.mold.service.OpsMoldServiceTest.txt
└── ...
```

**优先读 XML**（结构化，便于解析）；XML 不存在或损坏时读 `.txt` 兜底。

---

## XML 报告结构

```xml
<?xml version="1.0" encoding="UTF-8"?>
<testsuite name="cn.sottop.operations.mold.service.OpsMoldServiceTest"
           tests="3" failures="1" errors="0" skipped="0" time="2.345">

  <!-- 通过的测试 -->
  <testcase name="findPage" classname="cn.sottop.operations.mold.service.OpsMoldServiceTest"
            time="1.234"/>

  <!-- 失败的测试（断言失败） -->
  <testcase name="addItem" classname="cn.sottop.operations.mold.service.OpsMoldServiceTest"
            time="0.456">
    <failure message="expected: &lt;3&gt; but was: &lt;0&gt;"
             type="org.opentest4j.AssertionFailedError">
org.opentest4j.AssertionFailedError: expected: &lt;3&gt; but was: &lt;0&gt;
    at cn.sottop.operations.mold.service.OpsMoldServiceTest.addItem(OpsMoldServiceTest.java:45)
    at java.base/java.lang.reflect.Method.invoke(Method.java:568)
    </failure>
  </testcase>

  <!-- 错误的测试（异常） -->
  <testcase name="updateStatus" classname="cn.sottop.operations.mold.service.OpsMoldServiceTest"
            time="0.123">
    <error message="Cannot invoke method getById because xxx is null"
           type="java.lang.NullPointerException">
java.lang.NullPointerException: Cannot invoke method getById because xxx is null
    at cn.sottop.operations.mold.service.impl.OpsMoldServiceImpl.updateStatus(OpsMoldServiceImpl.java:123)
    at cn.sottop.operations.mold.service.OpsMoldServiceTest.updateStatus(OpsMoldServiceTest.java:67)
    </error>
  </testcase>
</testsuite>
```

---

## 关键字段提取

| 字段 | XPath | 说明 |
|------|-------|------|
| 测试类全名 | `testsuite/@name` | 用于定位测试文件 |
| 总失败数 | `testsuite/@failures` + `testsuite/@errors` | 快速判断是否有失败 |
| 测试方法名 | `testcase/@name` | 失败的具体方法 |
| 失败类型 | `failure/@type` 或 `error/@type` | AssertionError / NPE 等 |
| 失败信息 | `failure/@message` 或 `error/@message` | expected vs actual |
| 完整堆栈 | `failure` 或 `error` 元素文本内容 | 用于归因分析 |

---

## 解析步骤

### Step 1：定位报告目录

根据本次运行的模块列表，查找所有 surefire-reports 目录：

```bash
# 查找所有模块的 surefire-reports
Get-ChildItem -Recurse -Filter "TEST-*.xml" -Path "." |
  Where-Object { $_.FullName -like "*surefire-reports*" -and $_.FullName -notlike "*.worktrees*" }
```

### Step 2：过滤有失败的报告

只处理 `failures > 0` 或 `errors > 0` 的 `<testsuite>`。

### Step 3：提取失败清单

对每个失败的 `<testcase>`，提取：

```json
{
  "testClass": "cn.sottop.operations.mold.service.OpsMoldServiceTest",
  "testMethod": "updateStatus",
  "failureType": "error",
  "exceptionType": "java.lang.NullPointerException",
  "message": "Cannot invoke method getById because xxx is null",
  "stackTrace": "java.lang.NullPointerException: ...\n    at cn.sottop.operations.mold.service.impl.OpsMoldServiceImpl.updateStatus(OpsMoldServiceImpl.java:123)\n    at ..."
}
```

### Step 4：定位源文件

从 `testClass` 推导对应的业务类和测试类路径：

```
测试类：cn.sottop.operations.mold.service.OpsMoldServiceTest
  → 测试文件：{module}/src/test/java/cn/sottop/operations/mold/service/OpsMoldServiceTest.java

堆栈中的业务类：cn.sottop.operations.mold.service.impl.OpsMoldServiceImpl
  → 业务文件：{module}/src/main/java/cn/sottop/operations/mold/service/impl/OpsMoldServiceImpl.java
```

模块推导规则：
```
cn.sottop.operations.** → apm-module-operations/apm-module-operations-biz
cn.sottop.system.**     → apm-module-system/apm-module-system-biz
cn.sottop.log.**        → apm-module-log/apm-module-log-biz
cn.sottop.framework.**  → apm-framework/apm-starter-log（或对应子模块）
```

---

## 控制台输出兜底解析

当 XML 不存在时（编译失败、容器启动失败），从 mvn test 控制台输出解析：

### 编译错误特征

```
[ERROR] COMPILATION ERROR :
[ERROR] /path/to/File.java:[45,12] cannot find symbol
  symbol:   method getXxx()
  location: class cn.sottop.xxx.XxxDTO
```

提取：文件路径 + 行号 + 错误描述

### 容器启动失败特征

```
[ERROR] Tests run: 0, Failures: 0, Errors: 1, Skipped: 0
...
Caused by: org.springframework.beans.factory.NoSuchBeanDefinitionException: No qualifying bean of type 'cn.sottop.xxx.XxxService'
```

→ 标记为"容器启动失败，人工处理"

---

## 输出格式（给修复流程使用）

解析完成后，输出结构化失败清单：

```
测试失败清单（共 N 个）：

[F1] OpsMoldServiceTest#updateStatus
  类型：NullPointerException（业务代码）
  根因文件：OpsMoldServiceImpl.java:123
  信息：Cannot invoke method getById because xxx is null
  堆栈：（省略，完整内容在 XML 中）

[F2] OpsMoldServiceTest#addItem
  类型：AssertionFailedError（测试断言）
  根因文件：OpsMoldServiceTest.java:45
  信息：expected: <3> but was: <0>
  分析：断言期望值可能与当前数据库状态不符

[人工处理] OpsMoldServiceTest#findPage
  类型：DataAccessException
  原因：数据库连接异常，非代码问题
```
