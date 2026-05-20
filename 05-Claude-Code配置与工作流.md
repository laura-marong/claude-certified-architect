# Claude 认证架构师考试 — 领域 3：Claude Code 配置与工作流（权重：约 20%）

## 概述

本领域涵盖为开发工作流配置 Claude Code，包括 CLAUDE.md 层级体系、自定义命令与技能、计划模式、迭代优化、CI/CD 集成与批处理。

---

## d3.1 — CLAUDE.md 层级与配置

### 概要

理解项目级、用户级和目录级设置在 Claude Code 层级配置体系中如何相互作用。

### 配置层级

- **用户级**（`~/.claude/CLAUDE.md`）：适用于所有项目的个人偏好，不共享
- **项目级**（`.claude/CLAUDE.md`）：通过版本控制共享的团队配置
- **目录级**（任意目录下的 `CLAUDE.md`）：限定于该目录及其子目录的规则
- **@import 语法**：引入外部 markdown 文件以实现模块化配置
- **`.claude/rules/` 目录**：存放按主题分类的规则文件，便于组织管理

### 应避免的反模式

- 将所有配置堆积在一个巨大的 CLAUDE.md 中，而不是利用模块化规则
- 不理解用户级、项目级和目录级配置之间的优先级关系

### 深度解析

系统使用三个层级，将其合并应用。更具体的配置覆盖更通用的配置：**目录级 > 项目级 > 用户级**。

**模块化配置**的核心是将规则拆分为按主题分类的文件，而非一个单体的 CLAUDE.md。`@import` 指令引入外部规则文件，而 `.claude/rules/` 目录中的所有内容自动加载。规则可使用 YAML frontmatter 中的 `paths` glob 模式实现路径级作用域。这种方式使规则更易查找和维护，并允许不同团队成员负责不同的规则文件。

### 代码示例 — 文件结构

```
~/.claude/
  CLAUDE.md                    # User-level (personal use, not shared)
    "Use vim keybindings"
    "Prefer dark theme output"

project/
  .claude/
    CLAUDE.md                  # Project-level (shared via git)
      "Use TypeScript strict mode"
      "Follow ESLint airbnb config"
      "@import ./rules/api-design.md"
    rules/
      typescript.md            # Auto-loaded rule files
      testing.md               # Auto-loaded rule files
      api-design.md            # Imported by CLAUDE.md
    commands/
      review.md                # /review slash command
      deploy.md                # /deploy slash command
    skills/
      refactor/
        SKILL.md               # Refactor skill (forked context)

  src/
    api/
      CLAUDE.md                # Directory-level (scoped rules)
        "All endpoints must validate auth token"
        "Use Zod schemas for request validation"
```

### 对比：反模式 vs 正确做法

| 反模式 ❌ | 正确做法 ✅ |
|-----------|------------|
| 800 行的单体 `.claude/CLAUDE.md`，将 vim 键绑定等个人偏好混入项目配置，加上目录级 API 规则，全部揉在一起 | 按关注点和作用域拆分——项目级存放团队标准和 `@import` 引用，用户级包含个人编辑器/终端偏好，目录级限定 API 相关的认证规则，`.claude/rules/` 存放测试规则等按主题分类的文件 |

### 考试提示

个人偏好属于用户级配置。团队标准放在项目级。模块特定规则放在目录级。任何将个人偏好放入项目配置的答案都是错误的。

---

## d3.2 — 自定义命令与技能

### 概要

创建自定义斜杠命令和技能来为团队扩展 Claude Code 的能力。

### 命令与技能体系

- **自定义斜杠命令**：存储在 `.claude/commands/` 中，是团队可共享的快捷方式
- **技能（Skills）**：位于 `.claude/skills/` 中，使用 `SKILL.md` 文件定义复杂、可复用的行为
- **SKILL.md frontmatter 字段**：`context: fork`、`allowed-tools`、`argument-hint`
- **路径特定规则**：使用 YAML frontmatter 中的 `paths` glob 模式实现针对性配置

### 应避免的反模式

- 当技能（提供分支上下文）更适合时，却选择了命令
- 在技能中未指定 `allowed-tools`，导致工具访问权限过于宽泛

### 深度解析

Claude Code 提供两种服务于不同目的的扩展机制。

**自定义命令**（`.claude/commands/`）：简单的斜杠命令，如 `/review`、`/deploy`、`/test`。以 markdown 文件形式定义指令，在**当前会话上下文**中执行（同一个上下文窗口）。最适合快速的一次性操作。通过版本控制与团队共享。

**技能**（`.claude/skills/`）：复杂的多步骤可复用行为，定义为带有 YAML frontmatter 的 `SKILL.md` 文件。可以**分支上下文**（隔离执行，不污染主会话）并**限制工具访问**（通过 `allowed-tools`）。适用于需要隔离的复杂操作。

**SKILL.md frontmatter 字段说明：**

- `context: fork` — 在隔离上下文中运行
- `allowed-tools` — 限制技能可使用的工具（如 `[Read, Edit, Grep]`）
- `argument-hint` — 描述预期的参数

**决策指南：**

- **命令**："运行 lint 并显示错误"——简单，无需隔离
- **技能**："重构此模块以使用依赖注入"——复杂，需要上下文隔离

### 代码示例 — `.claude/skills/refactor/SKILL.md`

```yaml
---
context: fork
allowed-tools:
  - Read
  - Edit
  - Grep
argument-hint: "file or directory to refactor"
---
```

技能随后列出步骤：分析当前结构、识别 SOLID 违规、规划方案、使用 Edit（绝不用 Write）进行增量修改，并验证每次更改。关键规则包括：绝不删除现有测试、保留所有公共 API 签名、为重构后的函数添加 JSDoc 注释。

### 对比：反模式 vs 正确做法

| 反模式 ❌ | 正确做法 ✅ |
|-----------|------------|
| 为复杂重构定义命令，探索所有文件并重组代码库——它在主上下文中运行，用探索噪声污染主上下文，且没有工具限制 | 使用带 `context: fork` 的技能实现隔离，限制 `allowed-tools`（Read、Edit、Grep），将所有探索保持在分支内 |

### 考试提示

当任务需要上下文隔离或工具限制时，正确答案涉及使用技能而非命令。注意正确答案中的 `context: fork` 和 `allowed-tools`。

---

## d3.3 — 计划模式与迭代优化

### 概要

使用计划模式处理复杂任务，以及通过迭代优化模式逐步提高输出质量。

### 规划与迭代策略

- **计划模式（Plan mode）**：先思考后行动——对复杂的多步骤任务很有价值
- **直接执行（Direct execution）**：适用于定义明确、简单的任务
- **迭代优化模式**：具体示例、TDD 迭代、面试模式
- **TDD 迭代**：先写测试，实现功能，然后优化直到测试通过

### 应避免的反模式

- 对简单、定义明确的任务使用计划模式（不必要的开销）
- 对需要先进行架构思考的复杂任务跳过规划

### 深度解析

计划模式指示 Claude 在执行前先思考和规划方案。它对复杂任务至关重要，对简单任务则是浪费。

**何时使用计划模式**：跨多文件的架构变更、涉及多个相互关联组件的任务、错误代价高昂的场景、需要设计决策的新功能实现。

**何时使用直接执行**：简单且定义明确的任务（如修复拼写错误、添加日志语句）、范围清晰的单文件修改、正确方案显而易见的任务。

**迭代优化模式包括：**

- **具体示例**：提供期望输出的具体示例，以指导格式和风格
- **TDD 迭代**：编写失败测试（定义预期行为）→ 实现使其通过 → 运行测试验证 → 在保持测试绿色的同时优化 → 下一个需求重复此循环
- **面试模式**：让 Claude 在开始前提出澄清性问题，适用于需求模糊的场景

TDD 迭代循环在每个步骤中给 Claude 一个具体、可验证的目标，相比普通的"实现这个功能"请求，显著提高输出质量。

### 代码示例 — TDD 迭代工作流

1. **先写测试**：为 `getUserById` 定义测试，覆盖返回包含 id/name/email 的正常结果、对缺失用户抛出 `NotFoundError`、验证 id 为正整数
2. **运行测试（应失败）**：测试失败，因为 `getUserById` 尚未定义
3. **实现使其通过**：Claude 实现函数以满足所有测试
4. **再次运行测试（应通过）**：三项测试全部通过
5. **优化**：添加输入清理和连接池，同时保持测试绿色

### 考试提示

对于复杂的多文件任务，使用计划模式。对于简单修复，使用直接执行。TDD 迭代——写测试、实现、验证——是首选的优化模式。

---

## d3.4 — CI/CD 集成与批处理

### 概要

使用 `-p` 标志和结构化输出将 Claude Code 集成到 CI/CD 管道中，同时利用批处理实现成本优化。

### CI/CD 与批处理模式

- **`-p` 标志**：在非交互模式下运行 Claude Code，用于 CI/CD 管道
- **`--output-format json`**：获取结构化输出以供自动化处理
- **`--json-schema`**：强制执行特定的输出 schema
- **CI 中的会话上下文隔离**：将生成器和审查器的上下文分开
- **Message Batches API**：24 小时处理窗口，节省 50% 成本
- **`custom_id`**：在批处理期间追踪单个请求

### 应避免的反模式

- 在 CI/CD 管道中使用交互模式
- 同会话自审查，这会保留推理上下文并产生偏见
- 在代码审查管道中未能隔离生成器和审查器会话

### 深度解析

CI/CD 集成依赖 `-p` 标志实现非交互执行，配合结构化输出标志实现自动化处理。

**关键 CI/CD 标志**：`-p` 启用非交互模式（CI 必须使用）。`--output-format json` 提供结构化 JSON 输出以供解析。`--json-schema` 强制执行特定的输出结构。

**代码审查的会话隔离**：编写代码的生成器会话必须与审查器会话完全分离。如果审查器在同一会话中运行，它会保留生成器的推理过程，产生确认偏见。

**批处理**：通过 Message Batches API 在 24 小时内处理（非阻塞），相比同步请求节省 50% 成本，每个请求分配一个 `custom_id` 用于追踪，适合夜间审计、每周审查和非紧急分析。不应将其用于阻塞式 PR 审查或实时反馈。

### 代码示例 — `ci-review.yml` 管道

一个由 Pull Request 触发的 GitHub Actions 工作流：检出代码，使用 `-p` 标志以非交互模式运行 Claude Code。审查内容包括：超过 50 行的函数、异步操作中缺少错误处理、硬编码的凭证或 API 密钥、新函数缺少单元测试。结果使用 `--output-format json` 返回结构化 JSON，配合 `--json-schema` 定义输出对象，包含一个 issue 数组，每个 issue 指定 file、severity 和 description。

### 对比：反模式 vs 正确做法

| 反模式 ❌ | 正确做法 ✅ |
|-----------|------------|
| 同一会话中自审查——在会话 A 中生成代码，然后恢复同一会话执行审查命令。审查器保留生成器的推理上下文，产生确认偏见 | 分离会话——代码生成在会话 A 中完成，然后一个全新的会话 B 执行审查，不带任何生成阶段的上下文 |

### 考试提示

三个关键的 CI/CD 事实：始终使用 `-p` 进行非交互模式；绝不在同一会话中自审查；对非紧急审查使用 Batch API 以实现 50% 的成本节省。

---

## 领域 3 考试提示汇总

1. 熟记 CLAUDE.md 层级：用户级 → 项目级 → 目录级（具体覆盖通用）
2. 理解何时使用计划模式 vs 直接执行
3. CI/CD 自动化必须使用 `-p` 标志配合 `--output-format json`
4. Batch API 可节省 50% 成本——掌握同步与批处理的决策标准

---

## 相关考试场景

- **场景 2 — 使用 Claude Code 生成代码**：为团队工作流配置 Claude Code，测试 CLAUDE.md 设置、计划模式、斜杠命令和迭代优化
- **场景 4 — 开发者生产力**：使用 Claude Agent SDK 搭配内置工具和 MCP 服务器构建开发者工具
- **场景 5 — Claude Code 用于 CI/CD**：管道集成测试，考察 `-p` 标志使用、结构化输出、Batch API 和多轮代码审查

## 导航

- 上一领域：Tool Design & MCP（工具设计与 MCP）
- 下一领域：Claude API Integration（Claude API 集成）

---

> **来源**：[Claude Certified Architect — Claude Code Config Domain](https://claudecertifications.com/claude-certified-architect/domains/claude-code-config)
