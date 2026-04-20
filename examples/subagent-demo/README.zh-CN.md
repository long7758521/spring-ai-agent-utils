# Sub-Agent Demo

演示如何使用 Spring AI 的 **Task 工具** 将任务委派给专业化子智能体。

## 概述

本示例展示如何配置主智能体，使其能将任务委派给专业化子智能体。每个子智能体拥有独立的上下文窗口、系统提示词与工具访问权限。

## 快速开始

1. 设置环境变量：
```bash
export GOOGLE_CLOUD_PROJECT=your-project-id
# 或使用 Anthropic/OpenAI（见 application.properties）
```

2. 运行应用：
```bash
./mvnw spring-boot:run
```

## 配置

### Task 工具设置

```java
var taskTool = TaskTool.builder()
    .subagentTypes(ClaudeSubagentType.builder()
        .skillsResources(skillPaths)
        .chatClientBuilder("default", chatClientBuilder.clone())
        .braveApiKey(braveApiKey)
        .build())
    .build();

ChatClient chatClient = chatClientBuilder
    .defaultToolCallbacks(taskTool)
    // ... other tools and advisors
    .build();
```

### 应用属性

```properties
agent.skills.paths=classpath:/skills
agent.tasks.paths=classpath:/agents
```

## 项目结构

```
src/main/resources/
├── agents/
│   └── spring-ai-expert.md    # 自定义子智能体定义
├── skills/
│   └── ai-tutor/              # 子智能体可用的技能
└── prompt/
    └── MAIN_AGENT_SYSTEM_PROMPT_V2.md
```

## 自定义子智能体示例

子智能体以带 YAML 头信息的 Markdown 文件定义（`agents/spring-ai-expert.md`）：

```markdown
---
name: spring-ai-expert
description: Use this agent when the user asks questions about Spring AI...
model: sonnet
---

You are a Spring AI Expert...
```

## 演示能力

- **TaskTool** — 通过可插拔的解析器与执行器启动并管理子智能体
- **ClaudeSubagentType** — 为 Claude 子智能体类型配置默认工具、技能与模型路由
- **自定义子智能体** — 领域专家（Spring AI）
- **技能集成** — 子智能体在系统提示词中预载技能内容
- **多模型支持** — 通过头信息字段 `model` 将子智能体路由到不同模型

## 依赖

- `spring-ai-agent-utils` — 含 Task 工具与 Claude 子智能体支持的核心库
- `spring-ai-starter-model-google-genai` — Google Gemini（可配置为 Anthropic/OpenAI）

## 相关文档

- [Task 工具文档](../../spring-ai-agent-utils/docs/TaskTools.md)
- [子智能体框架](../../spring-ai-agent-utils/docs/Subagent.md)
- [Sub-Agent A2A Demo](../subagent-a2a-demo) — 本地与远程 A2A 子智能体组合
