## A2A 子智能体示例

本示例在分层智能体系统之上扩展对 [A2A（Agent-to-Agent）协议](https://google.github.io/A2A/) 的支持，支持与远程智能体通信。

### 概述

基础的 [subagent-demo](../subagent-demo) 使用 Markdown 定义的本地子智能体；本示例将本地 Claude 子智能体与远程 A2A 智能体结合。A2A 相关类位于独立的 [`spring-ai-agent-utils-a2a`](../../spring-ai-agent-utils-a2a/README.zh-CN.md) 模块。

### 核心组件

| 类 | 模块 | 作用 |
|----|------|------|
| `A2ASubagentDefinition` | `spring-ai-agent-utils-a2a` | 将 A2A `AgentCard` 包装为 `SubagentDefinition` |
| `A2ASubagentResolver` | `spring-ai-agent-utils-a2a` | 从 `/.well-known/agent-card.json` 获取智能体元数据 |
| `A2ASubagentExecutor` | `spring-ai-agent-utils-a2a` | 通过 JSON-RPC 向远程智能体发送任务 |
| `ClaudeSubagentType` | `spring-ai-agent-utils` | 为本地 Claude 子智能体配置工具、技能与模型路由 |

### 配置

```java
var taskTool = TaskTool.builder()
    // Local Claude subagents (from Markdown files)
    .subagentReferences(ClaudeSubagentReferences.fromResources(agentPaths))
    .subagentTypes(ClaudeSubagentType.builder()
        .skillsResources(skillPaths)
        .chatClientBuilder("default", chatClientBuilder.clone())
        .braveApiKey(braveApiKey)
        .build())

    // Remote A2A subagent
    .subagentReferences(new SubagentReference("http://localhost:10001/airbnb", A2ASubagentDefinition.KIND))
    .subagentTypes(new SubagentType(new A2ASubagentResolver(), new A2ASubagentExecutor()))

    .build();
```

### 前置条件

- 运行中的兼容 A2A 的智能体（例如 `http://localhost:10001/airbnb`）
- 智能体须在 well-known 端点暴露 agent card

### 依赖

- `spring-ai-agent-utils` — 含 Task 工具与 Claude 子智能体支持的核心库
- `spring-ai-agent-utils-a2a` — A2A 协议子智能体实现
- `spring-ai-starter-model-google-genai` — Google GenAI 模型（可配置）

### 运行

```bash
export GOOGLE_GENAI_API_KEY=your-key
mvn spring-boot:run
```

编排器会在启动时自动发现 A2A 智能体，并与内置 Claude 子智能体一并用于任务委派。

### 相关文档

- [Task 工具文档](../../spring-ai-agent-utils/docs/TaskTools.md)
- [子智能体框架](../../spring-ai-agent-utils/docs/Subagent.md)
- [spring-ai-agent-utils-a2a](../../spring-ai-agent-utils-a2a/README.zh-CN.md) — A2A 模块说明
- [Sub-Agent Demo](../subagent-demo) — 仅本地 Claude 子智能体示例
