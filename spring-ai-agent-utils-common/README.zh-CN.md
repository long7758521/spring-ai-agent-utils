# Spring AI Agent Utils Common

[Spring AI Agent Utils](../spring-ai-agent-utils/README.zh-CN.md) 项目的共享子智能体 SPI（服务提供者接口）。

本模块定义核心抽象，使不同的子智能体实现（基于 Claude、[A2A 协议](https://google.github.io/A2A/) 或自定义）能够接入 `TaskTool` 编排系统。

## SPI 接口

| 接口 | 说明 |
|------|------|
| `SubagentDefinition` | 定义子智能体的标识、描述、类型（kind）及引用元数据 |
| `SubagentResolver` | 将 `SubagentReference` 解析为完整的 `SubagentDefinition` |
| `SubagentExecutor` | 针对已解析的子智能体执行任务并返回结果 |
| `SubagentType` | 为特定 kind 将 `SubagentResolver` 与 `SubagentExecutor` 配对 |
| `SubagentReference` | 指向子智能体资源的轻量引用（URI + kind + 可选元数据） |
| `TaskCall` | 描述待执行任务的输入记录（提示词、子智能体类型、模型等） |

## 如何协同工作

```
SubagentReference ──► SubagentResolver ──► SubagentDefinition
                                                    │
                              TaskCall ──► SubagentExecutor ──► result
```

1. 将 `SubagentReference`（例如 classpath URI 或远程 URL）传给 `SubagentResolver`
2. 解析器加载并解析引用，得到包含完整元数据的 `SubagentDefinition`
3. 执行时将 `TaskCall` 与已解析的定义传给 `SubagentExecutor`
4. 执行器运行任务并返回字符串结果

## 实现自定义子智能体类型

```java
// 1. Define your subagent definition
public class MySubagentDefinition implements SubagentDefinition {
    @Override public String getName() { return "my-agent"; }
    @Override public String getDescription() { return "Does something useful"; }
    @Override public String getKind() { return "MY_KIND"; }
    @Override public SubagentReference getReference() { return this.ref; }
}

// 2. Implement the resolver
public class MySubagentResolver implements SubagentResolver {
    @Override
    public boolean canResolve(SubagentReference ref) {
        return ref.kind().equals("MY_KIND");
    }

    @Override
    public SubagentDefinition resolve(SubagentReference ref) {
        // Load and parse the reference into a full definition
        return new MySubagentDefinition(ref);
    }
}

// 3. Implement the executor
public class MySubagentExecutor implements SubagentExecutor {
    @Override
    public String getKind() { return "MY_KIND"; }

    @Override
    public String execute(TaskCall taskCall, SubagentDefinition subagent) {
        // Execute the task and return the result
        return "Result from my agent";
    }
}

// 4. Register with TaskTool
TaskTool.builder()
    .subagentReferences(new SubagentReference("my://agent-1", "MY_KIND"))
    .subagentTypes(new SubagentType(new MySubagentResolver(), new MySubagentExecutor()))
    .build();
```

## 安装

**Maven：**
```xml
<dependency>
    <groupId>org.springaicommunity</groupId>
    <artifactId>spring-ai-agent-utils-common</artifactId>
    <version>0.7.0</version>
</dependency>
```

> **说明：** 本模块通常作为 `spring-ai-agent-utils` 或 `spring-ai-agent-utils-a2a` 的传递依赖被引入。仅在独立模块中实现自定义子智能体类型时，才需要直接依赖本模块。

## 环境要求

- Java 17+
- Spring AI 2.0.0-SNAPSHOT（> M1）

## 许可证

Apache License 2.0
