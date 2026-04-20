# Spring AI Agent Utils A2A

面向 [Spring AI Agent Utils](../spring-ai-agent-utils/README.zh-CN.md) 项目的 [A2A（Agent-to-Agent）](https://google.github.io/A2A/) 协议子智能体实现。

本模块使 AI 智能体能够通过 A2A 协议将任务委派给远程智能体。它实现 [子智能体 SPI](../spring-ai-agent-utils-common/README.zh-CN.md)，因此 A2A 智能体可与本地 Claude 子智能体一样，通过同一套 `TaskTool` 被发现和调用。

## 工作原理

```
TaskTool
  │
  ├── SubagentReference("http://host:port/path", "A2A")
  │         │
  │         ▼
  │   A2ASubagentResolver
  │     ── fetches /.well-known/agent-card.json
  │     ── returns A2ASubagentDefinition (wraps AgentCard)
  │
  └── A2ASubagentExecutor
        ── sends message via JSON-RPC transport
        ── waits for task completion (60s timeout)
        ── extracts text from response artifacts
```

## 组件

| 类 | 说明 |
|----|------|
| `A2ASubagentDefinition` | 将 A2A 的 `AgentCard` 包装为 `SubagentDefinition`（kind = `"A2A"`） |
| `A2ASubagentResolver` | 从 well-known 端点拉取 agent card 并创建定义 |
| `A2ASubagentExecutor` | 向远程智能体发送文本消息并返回响应 |

## 用法

### 将 A2A 子智能体与本地 Claude 子智能体一同注册

```java
import org.springaicommunity.agent.common.task.subagent.SubagentReference;
import org.springaicommunity.agent.common.task.subagent.SubagentType;
import org.springaicommunity.agent.subagent.a2a.A2ASubagentDefinition;
import org.springaicommunity.agent.subagent.a2a.A2ASubagentExecutor;
import org.springaicommunity.agent.subagent.a2a.A2ASubagentResolver;

var taskTools = TaskTool.builder()
    // Local Claude subagents
    .subagentTypes(ClaudeSubagentType.builder()
        .chatClientBuilder("default", chatClientBuilder)
        .build())

    // Remote A2A subagent
    .subagentReferences(new SubagentReference("http://localhost:10001/myagent", A2ASubagentDefinition.KIND))
    .subagentTypes(new SubagentType(new A2ASubagentResolver(), new A2ASubagentExecutor()))

    .build();
```

### 仅使用 A2A 子智能体

```java
var taskTools = TaskTool.builder()
    .subagentReferences(
        new SubagentReference("http://host-a:10001/agent-a", A2ASubagentDefinition.KIND),
        new SubagentReference("http://host-b:10002/agent-b", A2ASubagentDefinition.KIND))
    .subagentTypes(new SubagentType(new A2ASubagentResolver(), new A2ASubagentExecutor()))
    .build();
```

### 自定义 agent card 路径

默认情况下，解析器从 `<uri>/.well-known/agent-card.json` 获取 agent card。可自定义路径：

```java
var resolver = new A2ASubagentResolver("/custom/agent-card.json");
```

## 智能体发现

解析器要求每个兼容 A2A 的智能体在其 well-known 端点暴露 [Agent Card](https://google.github.io/A2A/#/documentation?id=agent-card)。卡片提供智能体名称、描述与能力，用于填充 `SubagentDefinition` 并向 `TaskTool` 注册。

## 安装

**Maven：**
```xml
<dependency>
    <groupId>org.springaicommunity</groupId>
    <artifactId>spring-ai-agent-utils-a2a</artifactId>
    <version>0.7.0</version>
</dependency>
```

上述依赖会传递引入 [A2A Java SDK](https://github.com/a2aproject/a2a-java-sdk) 客户端与 JSON-RPC 传输。

## 示例

完整示例见 [subagent-a2a-demo](../examples/subagent-a2a-demo)，演示本地 Claude 子智能体与远程 A2A 智能体的组合。

## 环境要求

- Java 17+
- Spring AI 2.0.0-SNAPSHOT（> M1）
- 运行中的、兼容 A2A 的智能体服务

## 许可证

Apache License 2.0
