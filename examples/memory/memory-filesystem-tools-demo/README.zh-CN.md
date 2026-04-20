# Memory Filesystem Tools Demo

演示如何结合通用 **`FileSystemTools`** 与 **`ShellTools`**，在 Spring AI 中为智能体实现**长期记忆**。跨会话记忆模式与 [memory-tools-demo](../memory-tools-demo/README.zh-CN.md) 相同，但不使用专用记忆工具 —— 智能体用与其它文件操作相同的 `Read`、`Write`、`Edit`、`Bash` 自行管理记忆文件。

## 与 memory-tools-demo 的差异

两个示例都实现同一长期记忆思路：类型化记忆文件、`MEMORY.md` 索引、两步保存流程。区别在于**智能体使用的工具**：

| | [memory-tools-demo](../memory-tools-demo/README.zh-CN.md) | **本示例** |
|---|---|---|
| **工具** | 专用 `AutoMemoryTools`（6 类面向记忆的操作） | 通用 `FileSystemTools` + `ShellTools` |
| **路径模型** | 相对路径，沙箱限制在 memories 根目录 | 绝对路径，可访问完整文件系统 |
| **安全** | 内置穿越防护、拒绝绝对路径 | 无沙箱 —— 信任智能体自行留在记忆目录 |
| **记忆目录** | 在 `AutoMemoryTools.builder()` 中配置 | 通过系统提示注入 `{MEMORIES_ROOT_DIERCTORY}` |
| **灵感来源** | [Claude API SDK Memory Tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool) | [Claude Code auto memory](https://code.claude.com/docs/en/memory) |

**适合本方案时：**
- 智能体已有 `FileSystemTools` / `ShellTools` 处理其它任务，不想依赖 `AutoMemoryTools`
- 希望智能体在记忆之外拥有完整文件系统能力（例如读源码再写相关记忆）
- 希望直接对齐 Claude Code 的做法

**更适合 memory-tools-demo 时：**
- 希望记忆操作隔离在沙箱中，不影响其它文件
- 更偏好语义明确的工具名（`MemoryCreate`、`MemoryView`…）而非通用 `Write` / `Read`

## 概述

智能体用与其它工作相同的 `Read`、`Write`、`Edit` 管理记忆。系统提示（`AUTO_MEMORY_FILESYSTEM_TOOLS_SYSTEM_PROMPT.md`）说明存储位置与文件结构，无需专用记忆工具。

```
Session 1:                              Session 2（新 JVM 进程）:
  User: "I prefer concise answers"        Agent reads MEMORY.md（通过 Read）
  Agent: Write → feedback_style.md   →    Agent: already knows your preference
         Edit  → MEMORY.md（索引）         "Here's the short version..."
```

## 快速开始

### 前置条件

- Java 17+
- Maven 3.6+
- 任一 AI 服务商 API 密钥

### 配置

1. 在 `src/main/resources/application.properties` 中增加记忆目录属性：
```properties
agent.memory.dir=${user.home}/.spring-ai-agent/memory-filesystem-tools-demo/memory
```

2. 设置 API 密钥（默认 Anthropic）：
```bash
export ANTHROPIC_API_KEY=your-key-here
# 或 Google: export GOOGLE_CLOUD_PROJECT=your-project-id
```

3. 运行示例：
```bash
mvn spring-boot:run
```

4. 开始对话：
```
USER: My name is Alice. I'm a backend engineer and I prefer short answers.
USER: We're migrating from PostgreSQL to CockroachDB this quarter.
USER: exit
```

5. 再次运行 — 智能体回忆已存储内容：
```
USER: What do you know about me?
ASSISTANT: You're Alice, a backend engineer. You prefer short answers.
           You're migrating from PostgreSQL to CockroachDB this quarter.
```

## 项目结构

```
memory-filesystem-tools-demo/
├── src/main/java/org/springaicommunity/skills/
│   ├── Application.java                          # 装配 FileSystemTools + ShellTools 与聊天循环
│   └── MyLoggingAdvisor.java                     # 将工具调用与响应记录到 stdout
├── src/main/resources/
│   ├── application.properties                    # 模型配置
└── target/
    └── memory/                                   # 运行时写入的记忆文件
        ├── MEMORY.md                             # 所有记忆条目的索引
        └── user_profile.md                       # 示例：user 类型记忆
```

`classpath:/prompt/MAIN_AGENT_SYSTEM_PROMPT_V2.md` 从 `spring-ai-agent-utils` 依赖 jar 加载。`AUTO_MEMORY_FILESYSTEM_TOOLS_SYSTEM_PROMPT.md` 位于本示例目录内。

## 工作原理

### 应用装配

```java
ChatClient chatClient = chatClientBuilder
    .defaultSystem(p -> p
        .text(mainPrompt + "\n\n" + memoryToolsPrompt)
        .param("MEMORIES_ROOT_DIERCTORY", memoryDir))   // tells the agent where to write
    .defaultTools(
        ShellTools.builder().build(),          // Bash — for ls, mkdir, etc.
        FileSystemTools.builder().build())     // Read, Write, Edit — memory file operations
    .defaultAdvisors(
        ToolCallAdvisor.builder().build(),
        MyLoggingAdvisor.builder().build())
    .build();
```

占位符 `{MEMORIES_ROOT_DIERCTORY}` 注入系统提示，使智能体在每次 `Write` / `Read` / `Edit` 时知道绝对路径。与 `memory-tools-demo` 不同，此处**没有**强制沙箱 —— 智能体按约定遵循提示。

### 记忆生命周期

智能体决定保存记忆时执行两次文件操作：

**步骤 1 — `Write`**：创建带 YAML 头信息的类型化 `.md` 文件：
```markdown
---
name: user profile
description: Alice — backend engineer, prefers short answers
type: user
---

Backend engineer named Alice.
Prefers concise, direct responses without trailing summaries.
```

**步骤 2 — `Edit`**：向 `MEMORY.md` 追加指针行（若不存在则 `Write`）：
```markdown
- [User Profile](user_profile.md) — Alice, backend engineer, prefers short answers
```

下次会话中，智能体对 `MEMORY.md` 调用 `Read` 加载索引，再对相关记忆文件 `Read`。

### 记忆类型

| 类型 | 何时保存 | 示例 |
|---|---|---|
| `user` | 用户分享背景、目标或偏好 | 姓名、角色、沟通风格 |
| `feedback` | 用户纠正智能体或确认做法 | 「不要在结尾再总结」 |
| `project` | 项目决策、截止日、约束 | 迁移目标、冻结日期 |
| `reference` | 指向外部系统 | Linear 看板、Grafana 仪表盘 |

### 系统提示的作用

何时保存、如何组织文件、如何维护 `MEMORY.md`、哪些不应保存 —— 全部由 `AUTO_MEMORY_FILESYSTEM_TOOLS_SYSTEM_PROMPT.md` 驱动，没有专用工具语义可依赖，**提示即主要约束**。

## 配置

```properties
# application.properties

## AI provider（示例为 Anthropic；其它见「切换 AI 服务商」）
spring.ai.anthropic.api-key=${ANTHROPIC_API_KEY}
spring.ai.anthropic.chat.options.model=claude-sonnet-4-5-20250929

## Agent metadata shown to the model in the system prompt
agent.model=claude-sonnet-4-5-20250929
agent.model.knowledge.cutoff=2025-01-01

## Memory directory — must be set; persists across restarts
agent.memory.dir=${user.home}/.spring-ai-agent/memory-filesystem-tools-demo/memory
```

> **注意：** `agent.memory.dir` 无默认值，必须显式配置，否则应用无法启动。

## 切换 AI 服务商

编辑 `pom.xml` 取消注释所需 starter，并更新 `application.properties`：

**Google Gemini:**
```properties
spring.ai.google.genai.project-id=${GOOGLE_CLOUD_PROJECT}
spring.ai.google.genai.chat.options.model=gemini-3.1-pro-preview
agent.model=gemini-3.1-pro-preview
agent.model.knowledge.cutoff=Unknown
```

**OpenAI:**
```properties
spring.ai.openai-sdk.api-key=${OPENAI_API_KEY}
spring.ai.openai-sdk.chat.options.model=gpt-5-mini-2025-08-07
agent.model=gpt-5-mini-2025-08-07
agent.model.knowledge.cutoff=2025-08-07
```

## 相关文档

- [memory-tools-demo](../memory-tools-demo/README.zh-CN.md) — 相同模式，使用专用、沙箱化的 `AutoMemoryTools`
- [AutoMemoryTools 文档](../../../spring-ai-agent-utils/docs/AutoMemoryTools.md) — 专用工具方案的完整说明
- [FileSystemTools 文档](../../../spring-ai-agent-utils/docs/FileSystemTools.md) — 本示例使用的文件工具
- [Claude Code — Memory](https://code.claude.com/docs/en/memory) — 本示例直接遵循的模式
- [Claude API SDK — Memory Tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool) — 专用工具规范
