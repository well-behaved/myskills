# My Skills

> 一套 AI Agent 技能集合，涵盖 APM 后端项目全链路（代码审查、测试生成、自动修复、接口文档）、需求到交付的 RFLP 流程编排、数据库诊断、线上运维归档，以及 skill 自身的创建与发现。

## APM 系列 Skills

| Skill | 说明 |
|-------|------|
| `apm-code-review` | APM 后端项目代码审查编排器，自动确定范围、分批派发、汇总输出报告。 |
| `apm-auto-fix` | APM 项目自动修复循环。调用 `apm-code-review` 扫描代码，自动修复可安全修复的问题，修复后再次审查验证，循环直至干净。P0 及有业务歧义的问题仅上报不自动修复。支持 git worktree 并行模式。 |
| `apm-test-gen` | 根据 git diff（commit / 分支对比 / 指定文件）自动生成符合项目规范的 Spring Boot 测试类。支持 Service 集成测试和控制器单元测试，含增量与追加模式。 |
| `apm-test-pipeline` | APM 项目测试全流程自动化流水线。扫描测试代码 → 补全缺失测试 → 运行 → 分析失败 → 自动修复 → 循环验证 → 最终报告。 |
| `apm-doc-gen` | 根据 git 分支差异自动生成 APM 前端接口文档，支持单/双分支对比、路径前缀过滤、JSON 示例。 |

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

## 工具类 Skills

| Skill | 说明 |
|-------|------|
| `branch-diff-context` | 新 session 快速理解分支改动全貌。自动执行 git diff，输出结构化改动摘要，为后续 review 或提问建立上下文。 |
| `db-debugger` | 结合代码逻辑与数据库状态诊断问题。查询日志、业务数据，关联 Java/Spring 代码逻辑分析。安全只读模式。 |
| `ops-sql-runbook` | 线上 SQL 操作手册归档。将数据库变更（刷数据、配置修改、数据修复等）记录为飞书文档，形成可复用、可追溯的操作流水。 |
| `receiving-code-review` | 收到 code review 反馈后使用，在实施建议前先进行技术严谨性验证，避免盲目同意或无脑执行。 |

## 元技能 Skills

| Skill | 说明 |
|-------|------|
| `skill-creator` | 创建和优化 agent skill。处理目录脚手架、SKILL.md 编写、frontmatter 优化、资源文件组织。 |
| `find-skills` | 帮助用户发现和安装 agent skill。当用户问「怎么做 X」「有没有 skill 能…」时自动触发。 |

## 安装使用

将本仓库克隆到 Claude Code agent skills 目录：

```bash
# Linux / macOS
git clone https://github.com/well-behaved/myskills ~/.claude/skills

# Windows
git clone https://github.com/well-behaved/myskills %USERPROFILE%\.claude\skills
```

Skills 会在 Claude Code 启动时自动加载，agent 根据触发描述自动匹配调用。
