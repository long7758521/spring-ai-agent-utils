# Code Agent Demo

基于 Spring AI 与 [spring-ai-agent-utils](../../spring-ai-agent-utils/README.zh-CN.md) 的交互式 AI 编程助手。支持文件操作、Shell 执行、网页访问、MCP 集成与可扩展技能。

## 概述

命令行 AI 助手具备：

- **代码操作**：读、写、编辑文件；使用 grep 搜索
- **Shell 执行**：同步或后台运行命令
- **网页检索**：搜索并抓取网页内容，由 AI 摘要
- **任务管理**：用待办列表跟踪多步操作
- **用户交互**：执行过程中提问并收集答案
- **技能系统**：从 Markdown 加载自定义能力
- **MCP 集成**：连接 Model Context Protocol 服务（含 AirBnB 演示）
- **多模型支持**：Anthropic Claude、OpenAI GPT 或 Google Gemini

## 工具

- **[AskUserQuestionTool](../../spring-ai-agent-utils/docs/AskUserQuestionTool.md)** — 执行过程中的交互式问答
- **[SkillsTool](../../spring-ai-agent-utils/docs/SkillsTool.md)** — 从 Markdown 加载自定义技能
- **[ShellTools](../../spring-ai-agent-utils/docs/ShellTools.md)** — 执行 Shell 命令
- **[FileSystemTools](../../spring-ai-agent-utils/docs/FileSystemTools.md)** — 文件读/写/编辑
- **[GrepTool](../../spring-ai-agent-utils/docs/GrepTool.md)** — 基于正则的代码搜索
- **[SmartWebFetchTool](../../spring-ai-agent-utils/docs/SmartWebFetchTool.md)** — AI 驱动的网页内容提取
- **[BraveWebSearchTool](../../spring-ai-agent-utils/docs/BraveWebSearchTool.md)** — 网页搜索
- **[TodoWriteTool](../../spring-ai-agent-utils/docs/TodoWriteTool.md)** — 任务跟踪
- **MCP 工具** — 通过 Model Context Protocol 集成外部工具

## 前置条件

- Java 17+
- Maven 3.6+
- 至少一个 AI 服务商的 API 密钥：
  - `ANTHROPIC_API_KEY`（Claude，推荐）
  - `OPENAI_API_KEY`（GPT）
  - `GOOGLE_CLOUD_PROJECT`（Gemini）
- 可选：`BRAVE_API_KEY`（网页搜索）

## 快速开始

```bash
# 1. 设置环境变量
export ANTHROPIC_API_KEY=your-key-here
export BRAVE_API_KEY=your-brave-key  # 可选

# 2. 运行
cd examples/code-agent-demo
mvn spring-boot:run
```

示例交互：
```
> USER: Search for TODO comments in Java files
> ASSISTANT: [Uses GrepTool to search]

> USER: What tools do I have available?
> ASSISTANT: [Lists configured tools]
```

## 配置

主要设置在 [application.properties](src/main/resources/application.properties)：

```properties
# Active provider (uncomment one in pom.xml)
spring.ai.anthropic.api-key=${ANTHROPIC_API_KEY}
spring.ai.anthropic.chat.options.model=claude-sonnet-4-5-20250929

# Skills location (classpath resource)
agent.skills.paths=classpath:/.claude/skills

# MCP servers
spring.ai.mcp.client.stdio.servers-configuration=classpath:/mcp-servers-config.json

# Model metadata for system prompt
agent.model=claude-sonnet-4-5-20250929
agent.model.knowledge.cutoff=2025-01-01
```

### 切换模型

1. 编辑 [pom.xml](pom.xml) — 注释/取消注释所需服务商
2. 在 [application.properties](src/main/resources/application.properties) 中配置对应项
3. 重新构建：`mvn clean install`

### 自定义技能

在 [src/main/resources/.claude/skills/](src/main/resources/.claude/skills/) 添加 Markdown：

```markdown
---
name: my-skill
description: When to use this skill
allowed-tools: Read, Bash
---
Skill prompt instructions here...
```

详见 [SkillsTool 文档](../../spring-ai-agent-utils/docs/SkillsTool.md)。

## 架构

### ChatClient 配置

[Application.java:51-94](src/main/java/org/springaicommunity/agent/Application.java#L51-L94) 中配置 ChatClient：

```java
ChatClient chatClient = chatClientBuilder
    .defaultSystem(systemPrompt)                        // MAIN_AGENT_SYSTEM_PROMPT_V2.md
    .defaultToolCallbacks(mcpToolCallbackProvider)      // MCP tools
    .defaultToolCallbacks(skillsTool)                   // Skills
    .defaultTools(AskUserQuestionTool, TodoWriteTool,   // Agent tools
                  ShellTools, FileSystemTools,
                  SmartWebFetchTool, BraveWebSearchTool, GrepTool)
    .defaultAdvisors(ToolCallAdvisor,                   // Tool execution
                     MessageChatMemoryAdvisor)          // 500-message memory
    .build();
```

### 要点

- **系统提示词**：[MAIN_AGENT_SYSTEM_PROMPT_V2.md](../../spring-ai-agent-utils/src/main/resources/prompt/MAIN_AGENT_SYSTEM_PROMPT_V2.md) — 智能体行为总配置
- **MCP 集成**：自动连接已配置的 MCP 服务（含 AirBnB 示例）
- **用户交互**：`AskUserQuestionTool` 支持执行过程中的多选题
- **日志**：可选 `MyLoggingAdvisor` 用于调试（当前已注释）

## 使用示例

**代码搜索**
```
> USER: Find all classes extending SpringBootApplication
[Uses GrepTool with pattern]
```

**交互式提问**
```
> USER: Help me choose a testing framework
> ASSISTANT: [Uses AskUserQuestionTool to present options: JUnit, TestNG, etc.]
```

**多步任务**
```
> USER: Create a DateUtil class, write tests, and run them
[Uses TodoWriteTool to plan → FileSystemTools to write → ShellTools to test]
```

**网页研究**
```
> USER: Find latest Spring AI advisor patterns
[Uses BraveWebSearchTool + SmartWebFetchTool]
```

## 项目结构

```
code-agent-demo/
├── src/main/java/.../agent/
│   ├── Application.java           # ChatClient 与主循环
│   └── MyLoggingAdvisor.java      # 可选调试 Advisor
├── src/main/resources/
│   ├── .claude/skills/            # 自定义技能目录
│   ├── application.properties     # 配置
│   └── mcp-servers-config.json    # MCP 服务定义
└── pom.xml                        # 依赖
```

## 工作流程

1. 应用启动，用工具与 Advisor 构建 ChatClient
2. 用户在控制台循环中输入提示
3. 智能体按系统提示处理请求
4. 选择并调用合适工具
5. 将结果综合为回复
6. 对话记忆保留上下文（500 条消息）

## 自定义

**调整记忆**：
```java
MessageWindowChatMemory.builder().maxMessages(1000).build()
```

**工具配置**：
```java
BraveWebSearchTool.builder(apiKey).resultCount(20).build()
SmartWebFetchTool.builder(client).maxContentLength(150_000).build()
```

**启用调试日志**：
取消注释 [Application.java:90-93](src/main/java/org/springaicommunity/agent/Application.java#L90-L93) 中的 `MyLoggingAdvisor`。

## 资源

- [spring-ai-agent-utils 文档](../../spring-ai-agent-utils/README.zh-CN.md)
- [工具文档](../../spring-ai-agent-utils/docs/)
- [Spring AI 文档](https://docs.spring.io/spring-ai/reference/)

## 许可证

Apache License 2.0
