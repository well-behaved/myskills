# My Skills

> 一套面向 APM 后端项目的 AI 技能集合，涵盖代码审查、测试生成、自动修复、需求到交付的全链路流程编排。

## APM 系列 Skills

| Skill | 说明 |
|-------|------|
| `apm-code-review` | APM 后端项目代码审查编排器，自动确定范围、分批派发、汇总输出报告。 |
| `apm-auto-fix` | APM 项目自动修复循环。调用 `apm-code-review` 扫描代码，自动修复可安全修复的 P1/P2 规范问题（F01-F08），修复后再次调用审查验证，循环直至零缺陷。P0 级业务逻辑问题仅上报不自动修复。支持 git worktree 并行模式。 |
| `apm-test-gen` | 根据 git diff（commit / 分支对比 / 指定模块/文件）自动生成符合项目规范的 Spring Boot 测试类。支持 Service 集成测试（@SpringBootTest）和控制器单元测试（Mockito），含增量式生成和追加模式。 |
| `apm-test-pipeline` | APM 项目测试全流程自动化流水线。完整链路：扫描测试代码 → 补全缺失测试 → 运行测试 → 分析失败 → 自动修复 → 循环验证 → 最终报告。内置判断业务逻辑缺陷还是测试代码缺陷，执行 2 轮修复循环。 |

## 工具类 Skills

| Skill | 说明 |
|-------|------|
| `branch-diff-context` | 在 session 开始时快速理解分支的改动全貌。自动执行 git diff，提取变更文件、代码结构、改动摘要（新增/修改/删除分类、核心业务逻辑说明），用于辅助代码审查或 review 对话的上下文预热。 |

## RFLP 流程 Skills

> RFLP（Requirements → Framework → Logic → Polish）：从需求到交付的完整研发流程，每步都有对应 skill 编排。

| Skill | 说明 |
|-------|------|
| `rflp-arch` | 为新项目或已有项目生成全景图，一页纸看清模块结构与串联关系。 |
| `rflp-prd` | 将一个想法打磨成经打分验证的 PRD，设计决策 + 场景/规格，不到 9 分不往下走。 |
| `rflp-trd` | 从确认的 PRD 输出技术方案（L 内容 + UT 规格 + 实现计划），任务粒度 2-5 分钟。 |
| `rflp-code` | 按技术方案逐 task 执行编码，带验证和 review 检查点。 |
| `rflp-finish` | 验证所有 task 完成度，合并 RFL 文档，处理分支收尾。 |
| `rflp-test` | 用压力场景对 skill 做 TDD 测试，发现薄弱点并修补。 |

## 安装使用

将本仓库克隆到 OpenCode agent skills 目录：

```bash
# Linux / macOS
git clone https://github.com/well-behaved/myskills ~/.agents/skills

# Windows
git clone https://github.com/well-behaved/myskills %USERPROFILE%\.agents\skills
```

Skills 会在 OpenCode 启动时自动加载，agent 根据触发描述自动匹配调用。
