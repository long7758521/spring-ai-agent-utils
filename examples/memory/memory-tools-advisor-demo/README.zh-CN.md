# memory-tools-advisor-demo

可运行的 Spring Boot 控制台智能体，在贴近实战的多 Advisor 配置下演示 [`AutoMemoryToolsAdvisor`](../../../spring-ai-agent-utils/docs/AutoMemoryToolsAdvisor.md)。

## 演示内容

- **`AutoMemoryToolsAdvisor`** — 双重条件触发合并：距上一轮超过 60 秒**或**用户消息包含 `"bye"` 时执行合并
- **`ToolCallAdvisor`** 配合 `disableInternalConversationHistory()` — 处理递归工具调用，将对话历史交给专用 Advisor
- **`MessageChatMemoryAdvisor`** 基于 `MessageWindowChatMemory`（100 条窗口）— 在基于文件的长期记忆之外提供短期会话记忆
- **`MyLoggingAdvisor`** — 自定义 Advisor，在开发时将工具调用与消息文本彩色打印到控制台，便于跟踪

## Advisor 栈

```
AutoMemoryToolsAdvisor   (HIGHEST_PRECEDENCE + 200)  ← 长期记忆
ToolCallAdvisor                                       ← 递归工具执行
MessageChatMemoryAdvisor                              ← 短期对话窗口
MyLoggingAdvisor         (order = 0)                 ← 开发用控制台日志
```

## 关键源文件

| 文件 | 作用 |
|---|---|
| [`Application.java`](src/main/java/org/springaicommunity/agent/Application.java) | Spring Boot 入口 — 构建 `ChatClient` 并运行控制台循环 |
| [`MyLoggingAdvisor.java`](src/main/java/org/springaicommunity/agent/MyLoggingAdvisor.java) | 开发用 Advisor：记录用户消息、工具调用与助手回复 |
| [`application.properties`](src/main/resources/application.properties) | 模型凭证、模型选择与记忆目录路径 |

## 合并（consolidation）触发条件

示例使用带状态 lambda，在两种条件下触发：

```java
Instant lastInteraction = Instant.now();

.memoryConsolidationTrigger((request, instant) -> {
    var previousInteraction = lastInteraction;
    lastInteraction = Instant.now();

    // Consolidate when more than 60 seconds have passed since the last turn
    if (instant.isAfter(previousInteraction.plusSeconds(60))) {
        return true;
    }

    // Also consolidate when the user says goodbye
    var userMessage = request.prompt().getLastUserOrToolResponseMessage().getText();
    return userMessage != null && userMessage.toLowerCase().contains("bye");
})
```

## 配置

记忆默认保存在 `${user.home}/.spring-ai-agent/memory-tools-demo/memory`。可在 `application.properties` 中修改：

```properties
agent.memory.dir=/path/to/your/memories
```

`application.properties` 中含已注释的 Anthropic 与 OpenAI SDK 段落。切换模型时，在 `pom.xml` 中取消注释对应 starter 并更新 `agent.model` 属性即可，无需改 Java。

## 运行

```bash
cd examples/memory/memory-tools-advisor-demo
ANTHROPIC_API_KEY=sk-... mvn spring-boot:run
```

启动后进入 REPL：输入消息并回车。输入 `bye` 可在结束会话前显式触发一次记忆合并。

## 相关示例

- [memory-tools-demo](../memory-tools-demo/README.zh-CN.md) — `AutoMemoryTools` 基础用法
- [spring-ai-agent-utils 文档](../../../spring-ai-agent-utils/README.zh-CN.md)
