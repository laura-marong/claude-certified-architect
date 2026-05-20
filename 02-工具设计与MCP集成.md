# Claude 认证架构师考试 — 领域 2：工具设计与 MCP 集成（权重：约 20%）

## 概述

本领域涵盖有效工具设计与 MCP 服务器集成，包括工具描述最佳实践、结构化错误响应、工具分发、MCP 配置以及 Claude 内置工具。

---

## d2.1 工具描述最佳实践

### 概要

编写清晰有效的工具描述，帮助 Claude 正确选择和使用工具。需包含输入格式、示例、边界情况与边缘条件。

### 优秀工具描述的要素

- 在工具描述中包含输入格式规范及示例
- 明确边缘情况和边界条件，使模型能正确处理
- 清晰的参数描述，包含预期类型、范围与约束
- 工具描述是写给模型的文档——越详细越好

### 应避免的反模式

- 模糊的工具描述，使模型对何时或如何使用工具产生歧义
- 缺少边缘情况文档，导致意外行为

### 深度解析

工具描述是 Claude 决定**何时**以及**如何**使用工具的主要机制。可以将其视为专门为模型编写的文档。

**优秀工具描述的组成：**

- **清晰目的**：一句话说明工具的功能
- **输入规范**：精确的类型、格式、范围与约束
- **示例**：展示常见场景下的预期输入/输出对
- **边缘情况**：记录空输入、无效数据、边界值等情况下的行为
- **何时不应使用**：明确工具边界以防误用

一个模糊的描述（如"搜索客户"）会迫使 Claude 去猜测。详细描述则消除所有歧义。

### 代码示例 — 良好设计的工具定义

```json
{
  "name": "lookup_customer",
  "description": "Search for a customer by email, phone, or account ID. Returns a customer profile including name, account status, and order history summary. Input: Pass exactly one of email, phone, or account_id. Email must contain @. Phone must be in E.164 format (e.g. +15551234567). Account ID must start with ACC-. Returns: Customer object or empty array if not found. Note: an empty result means the customer does not exist - this is not an error.",
  "input_schema": {
    "type": "object",
    "properties": {
      "email": {
        "type": "string",
        "description": "Customer email address (must contain @)"
      },
      "phone": {
        "type": "string",
        "description": "Phone in E.164 format, e.g. +15551234567"
      },
      "account_id": {
        "type": "string",
        "description": "Account ID starting with ACC-, e.g. ACC-12345"
      }
    }
  }
}
```

### 对比：反模式 vs 正确做法

| 反模式 ❌ | 正确做法 ✅ |
|-----------|------------|
| `name: "search"`，`description: "搜索一些东西"`，输入仅有 `query`（字符串），用途不明 | `name: "lookup_customer"`，description 明确说明搜索方式（邮箱/电话/账户ID）、格式要求、返回值及空结果含义 |

### 考试提示

考试会呈现不同质量的工具描述。正确答案始终具有最详细的描述，包含输入格式、示例、边缘情况和边界文档。

---

## d2.2 结构化错误响应

### 概要

设计错误响应，使 Agent 获得足够信息以决定恢复或升级。使用结构化字段，如 `isError`、`errorCategory`、`isRetryable`。

### 错误响应设计模式

- **isError 标志**：向 Agent 明确标识工具执行失败
- **errorCategory**：对错误分类（如 `"validation"`、`"auth"`、`"not_found"`、`"rate_limit"`）
- **isRetryable**：告知 Agent 再次尝试同一调用是否可能成功
- **结构化错误上下文**：包含尝试了什么操作、什么环节失败了

### 应避免的反模式

- 使用"操作失败"之类的通用错误信息，隐藏了有价值的上下文
- 通过返回空结果来悄悄压制错误，将其伪装为成功
- 未能区分访问失败与真正空结果

### 深度解析

当工具失败时，错误响应必须给予 Agent 足够的信息以决定下一步：重试、尝试替代方案还是升级。

**结构化错误响应字段：**

- **isError**：标识这是失败而非结果（true/false）
- **errorCategory**：错误类型分类（`"auth"`、`"not_found"`、`"rate_limit"`、`"timeout"`、`"validation"`）
- **isRetryable**：Agent 是否应重试（超时类为 true，认证失败类为 false）
- **context**：尝试了什么以及具体什么失败了

**关键区别——访问失败 vs 空结果（考试最高频考点之一）：**

- **访问失败**："我无法检查数据库" → `isError: true`（搜索**未**执行）
- **空结果**："我检查了数据库，什么都没找到" → `isError: false`（搜索**已**执行）

**绝不要通过返回空结果来悄悄压制访问失败。** 如果数据库宕机了，返回 `[]` 会让 Agent 认为客户不存在——这是灾难性的误解。

### 代码示例 — 错误响应设计

```json
{
  "access_failure_example": {
    "isError": true,
    "errorCategory": "timeout",
    "isRetryable": true,
    "context": {
      "attempted": "Lookup customer by email: user@example.com",
      "service": "customer-database",
      "timeout_ms": 5000,
      "suggestion": "Retry in 2s, or try lookup by account ID instead"
    }
  },
  "empty_result_example": {
    "isError": false,
    "customers": [],
    "metadata": {
      "searched_by": "email",
      "query": "user@example.com",
      "results_count": 0
    }
  }
}
```

### 对比：反模式 vs 正确做法

| 反模式 ❌ | 正确做法 ✅ |
|-----------|------------|
| 数据库宕机但返回 `{"customers": []}`——Agent 以为"没有客户"，但实际是"根本无法检查"——这是静默的灾难性错误 | 数据库宕机时明确报告：`isError: true`，`errorCategory: "timeout"`，`isRetryable: true`，包含上下文和建议——Agent 知道搜索失败了并可做出决策 |

### 考试提示

如果考题涉及工具连接外部服务失败，正确答案始终会区分访问失败与空结果。对失败的连接返回 `[]` 永远是错误的。

---

## d2.3 工具分发与选择

### 概要

跨 Agent 有效分发工具。理解工具数量对选择质量的影响以及如何限定工具访问范围。

### 工具分发策略

- **每 Agent 4–5 个工具最优**；过多工具（如 18 个）会降低选择质量
- **限定工具访问范围**：每个 Agent 仅获得与其任务相关的工具
- **tool_choice 选项**：`"auto"`（模型自行决定）、`"any"`（必须使用某工具）或强制指定工具
- **工具分组**：组织相关工具并分配给专业化的 Agent

### 应避免的反模式

- 给 Agent 分配 18+ 个工具，而其中仅 4–5 个与其任务相关
- 在任务明确时未使用 `tool_choice` 来约束工具选择

### 深度解析

分配给单个 Agent 的工具数量直接影响其选择正确工具的能力。研究表明，**每 Agent 4–5 个工具**是最优的。

**工具过多的弊端：**

- 18+ 个工具时，Claude 必须针对当前任务逐一评估每个工具
- 相似工具造成歧义（如 `search_customers` vs `find_customer` vs `lookup_user`）
- 选择准确性随选项空间增大而下降
- 更多工具描述消耗宝贵的上下文窗口空间

**解决方案：将工具分发到专业化子 Agent**

与其给一个 Agent 18 个工具，不如创建一个协调器，搭配 3–4 个子 Agent，每个拥有 4–5 个聚焦工具：

- **客户 Agent**：`lookup_customer`、`update_account`、`check_status`、`verify_identity`
- **订单 Agent**：`find_order`、`process_refund`、`update_shipping`、`track_package`
- **沟通 Agent**：`send_email`、`send_sms`、`create_ticket`、`escalate_human`
- **协调器**：`Task`（委派）、`summarize`、`format_response`

**tool_choice 参数**控制工具选择：

- `"auto"` — Claude 自行决定是否及使用哪个工具（默认）
- `"any"` — Claude 必须使用某工具（但可自行选择）
- `{"type": "tool", "name": "X"}` — 强制使用特定工具

### 代码示例 — 跨 Agent 分发工具

**错误做法：** 一个 Agent 拥有 18 个工具（`lookup_customer`、`update_account`、`verify_identity`、`find_order`、`process_refund`、`update_shipping`、`track_package`、`send_email`、`send_sms`、`create_ticket`、`escalate`、`search_kb`、`check_inventory`、`apply_coupon`、`schedule_callback`、`log_interaction`、`generate_report`、`update_preferences`）——选择质量严重下降！

**正确做法：** 协调器 + 专业化子 Agent：

- 协调器：3 个工具（`Task`、`summarize_results`、`format_response`）
- 客户 Agent：4 个工具
- 订单 Agent：4 个工具
- 沟通 Agent：4 个工具

### 考试提示

当考试呈现涉及大量工具的场景时，正确答案始终是将它们分布到各专业化子 Agent 中，每个拥有 4–5 个工具。

---

## d2.4 MCP 服务器配置

### 概要

配置 Model Context Protocol 服务器，实现项目级和用户级工具集成。理解 `.mcp.json` 与 `~/.claude.json` 配置的区别。

### MCP 配置要点

- **.mcp.json**：项目级 MCP 服务器配置（与团队共享）
- **~/.claude.json**：用户级 MCP 配置（个人工具）
- MCP 配置中的**环境变量展开**用于密钥管理
- MCP 服务器通过自定义工具和数据源扩展 Claude 的能力

### 应避免的反模式

- 在 `.mcp.json` 中硬编码密钥，而非使用环境变量展开
- 混合项目级和用户级配置而不理解优先级

### 深度解析

**Model Context Protocol (MCP)** 服务器通过自定义工具和数据源扩展 Claude 的能力。配置分为两个层级：

**配置文件：**

- **.mcp.json**（项目级）——通过版本控制共享，团队工具，项目特定集成
- **~/.claude.json**（用户级）——个人使用，不共享，个人 API 密钥

**安全性：环境变量展开**

绝不要在配置文件中硬编码密钥。使用 `${ENV_VAR}` 语法：

- 密钥不会进入版本控制
- 每位开发者使用自己的凭据
- CI/CD 可注入环境特定的值

**MCP 服务器可提供：**

- **工具（Tools）**：Claude 可调用的自定义函数（如 Jira 集成、数据库查询）
- **资源（Resources）**：静态数据或文档（如 API 规范、模式文档）
- **提示（Prompts）**：针对常见任务的预构建提示模板

### 代码示例 — 项目级 MCP 配置

```json
{
  "mcpServers": {
    "jira": {
      "command": "npx",
      "args": ["@company/jira-mcp-server"],
      "env": {
        "JIRA_URL": "${JIRA_URL}",
        "JIRA_TOKEN": "${JIRA_TOKEN}"
      }
    },
    "postgres": {
      "command": "npx",
      "args": ["@company/pg-mcp-server", "--read-only"],
      "env": {
        "DATABASE_URL": "${DATABASE_URL}"
      }
    }
  }
}
```

### 对比：反模式 vs 正确做法

| 反模式 ❌ | 正确做法 ✅ |
|-----------|------------|
| 在配置文件中硬编码 `"JIRA_TOKEN": "sk-abc123-实际密钥"`——一旦提交到 git 即泄露密钥！ | 使用环境变量展开：`"JIRA_TOKEN": "${JIRA_TOKEN}"`——密钥保留在环境中，不进入代码 |

### 考试提示

如果某个考试答案在 `.mcp.json` 中硬编码了 API 密钥，那永远是错误的。正确做法使用 `${ENV_VAR}` 进行密钥管理。

---

## d2.5 内置工具

### 概要

了解 Claude 的内置工具：Read、Write、Edit、Bash、Grep 和 Glob。知道何时为不同任务使用何种工具。

### 内置工具及其用例

- **Read**：读取文件内容以理解代码或数据
- **Write**：从零创建新文件
- **Edit**：对现有文件进行精准修改
- **Bash**：执行 shell 命令，用于构建、测试和系统操作
- **Grep**：在代码库的多个文件中搜索模式
- **Glob**：查找匹配模式的文件，用于发现和导航

### 应避免的反模式

- 本应用 Edit 精准修改现有文件时，却使用了 Write
- 当 Read/Write/Edit 可用时，却使用 Bash 执行文件操作

### 深度解析

Claude Code 附带 6 个内置工具。了解**何时使用每个工具**是本领域的高频考点。

**六大内置工具：**

- **Read** — 读取文件内容（理解代码、检查数据）
- **Write** — 从零创建新文件（仅用于新文件！）
- **Edit** — 对现有文件进行精准修改
- **Bash** — 执行 shell 命令（测试、构建、安装）
- **Grep** — 跨文件搜索文本模式
- **Glob** — 查找匹配名称模式的文件

**关键区别：**

1. **Write vs Edit**：Write 用于**仅创建新文件**。Edit 用于**修改现有文件**。Write 会替换整个文件。
2. **Bash vs 内置工具**：绝不使用 Bash 执行已有专用工具的操作。不要用 `cat file.txt` 代替 Read。
3. **Grep vs Glob**：Grep 搜索**文件内部**的内容模式。Glob 搜索**文件名**的路径模式。

### 代码示例 — 正确工具选择

| 任务 | 正确做法 | 错误做法 |
|------|---------|---------|
| "读取配置文件" | `Read("config.json")` | `Bash("cat config.json")` |
| "创建新测试文件" | `Write("tests/new-test.ts", content)` | `Bash("echo '...' > tests/new-test.ts")` |
| "修复 server.ts 第42行的 bug" | `Edit("server.ts", old_text, new_text)` | `Write("server.ts", entire_file_content)` |
| "查找所有 getUserById 的用法" | `Grep("getUserById", "src/")` | `Bash("grep -r 'getUserById' src/")` |
| "查找所有 TypeScript 测试文件" | `Glob("**/*.test.ts")` | `Bash("find . -name '*.test.ts'")` |
| "运行测试套件" | `Bash("npm test")` | （无内置替代方案——此处用 Bash 正确） |

### 考试提示

考试频繁呈现 Agent 使用 Bash 进行文件操作的场景。正确答案始终使用专用内置工具（Read、Write、Edit、Grep、Glob）而非 Bash 等效命令。

---

## 领域 2 考试提示汇总

1. 每 Agent 保持 4–5 个工具以获得最佳选择质量
2. 结构化错误响应至关重要——始终包含 `isError`、`errorCategory`、`isRetryable`
3. 理解 `.mcp.json`（项目级）与 `~/.claude.json`（用户级）的区别
4. 内置工具：知道何时使用 Grep vs Glob vs Read

---

## 相关考试场景

- **场景 1 — 客户支持解决 Agent**：设计一个 AI 驱动的客户支持 Agent，处理咨询、解决问题和升级复杂案件。考察 Agent SDK 使用、MCP 工具和升级逻辑
- **场景 4 — 开发者生产力与 Claude**：使用 Claude Agent SDK 及内置工具和 MCP 服务器构建开发者工具。考察工具选择、代码库探索和代码生成工作流

## 导航

- 上一领域：Agentic Architecture（智能体架构）
- 下一领域：Claude Code Config（Claude Code 配置）

---

> **来源**：[Claude Certified Architect — Tool Design & MCP Domain](https://claudecertifications.com/claude-certified-architect/domains/tool-design-mcp)
