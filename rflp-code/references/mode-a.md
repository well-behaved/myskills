# 模式 A: 批量执行

有 Phase 分组按 Phase 边界分批；否则每 3 个 task 一批：

1. 严格按技术方案每个 step 执行（注意区分模板 A/B/C 的不同流程）
2. 跑 task 指定的验证命令（AT/FT/UT），优先用技术方案里写的命令
3. 每个 task 提交时**使用 TRD 文件列表中的精确路径**，禁止 `git add .`
4. 一批完成后**先跑编译检查**：

```bash
# 在仓库根目录执行，跳过测试只验证编译
mvn -B -DskipTests compile
```

编译结果处理：
- **BUILD SUCCESS** → 继续暂停报告
- **BUILD FAILURE** → **不暂停，立即修复**：读编译错误定位文件和行号，修复后重新编译确认通过，再进入暂停报告

5. 编译通过后**暂停报告**：

```
这批完成了（Task X-Y）：
- Task X: 改动文件：[列出] — 测试：PASS — 编译：✅
- Task Y: 改动文件：[列出] — 测试：PASS — 编译：✅

⚠️ 上下文提示：如已执行较多 task，建议新开 session 后用 /rflp-code 继续，
进度已保存至 docs/plans/.progress.json。

有反馈吗？
```

6. 根据反馈调整，然后执行下一批
