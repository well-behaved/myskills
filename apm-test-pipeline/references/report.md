# 报告模板

最终报告写入 `docs/test-pipeline-{日期}-{简短主题}.md`。

---

```markdown
# 测试流水线报告

> 执行时间：{yyyy-MM-dd HH:mm}
> 变更范围：{未提交变更 / feature/xxx 分支 / 最近N次提交}
> 执行轮次：{实际执行修复轮次} / 2

## 1. 执行概述

| 阶段 | 结果 |
|------|------|
| 变更文件扫描 | {N} 个业务类 |
| 测试类生成 | {N} 个新建 / {N} 个追加 / {N} 个已有跳过 |
| 第1轮测试 | {通过 N} / {失败 N} / {错误 N} |
| 第1轮修复 | 修复 {N} 个 / 回滚 {N} 个 / 人工处理 {N} 个 |
| 第2轮测试 | {通过 N} / {失败 N} / {错误 N} |
| 第2轮修复 | 修复 {N} 个 / 回滚 {N} 个 / 人工处理 {N} 个 |
| 最终结果 | ✅ 全绿 / ⚠️ 部分通过 / ❌ 有失败 |

---

## 2. 生成的测试类

| 测试类 | 模块 | 状态 | 测试方法数 |
|--------|------|------|-----------|
| `OpsMoldServiceTest.java` | operations-biz | 新建 | 3 |
| `OpsEquipmentTypeServiceTest.java` | operations-biz | 追加 2 个方法 | 5 |
| `ChangeLogDiffUtilTest.java` | apm-starter-log | 已有，跳过 | - |

---

## 3. 自动修复记录

### ✅ 修复成功

| 文件 | 修复内容 | 轮次 |
|------|---------|------|
| `OpsMoldServiceImpl.java:123` | 补充 null 检查 | 第1轮 |
| `OpsMoldServiceTest.java:45` | 更新断言期望值（3→0） | 第1轮 |

### ⚠️ 修复后回滚（lsp_diagnostics 有编译错误）

| 文件 | 编译错误 |
|------|---------|
| `XxxServiceImpl.java` | cannot find symbol: method getXxx() |

### ❌ 需人工处理

| 测试方法 | 失败类型 | 原因 |
|----------|---------|------|
| `OpsMoldServiceTest#findPage` | DataAccessException | 数据库连接异常，非代码问题 |
| `XxxServiceTest#batchAdd` | ApplicationContextException | Bean 加载失败，Spring 配置问题 |
| `YyyServiceTest#updateStatus` | AssertionError（修复2轮仍失败） | 超出轮次限制，业务逻辑复杂 |

---

## 4. 最终测试状态

```
✅ 通过：{N} 个
⚠️ 回滚未修复：{N} 个  
❌ 需人工处理：{N} 个
```

**总体评价**：[全部通过 ✅ / 需修改后通过 ⚠️ / 有阻塞问题 ❌]

**优先处理**：
1. {最紧急的人工处理项}
2. {次优先项}
```
