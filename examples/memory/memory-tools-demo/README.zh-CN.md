# Memory Tools Demo

演示如何结合 Spring AI 使用 **`AutoMemoryTools`** 为 AI 智能体实现**长期记忆**。智能体能在多次独立对话之间记住与用户、项目和行为反馈相关的事实；若无持久化，这些信息会在会话结束后丢失。

## 概述

本示例运行控制台聊天循环，智能体可在对话之间持久化并回忆知识。每当学到值得保留的内容（用户偏好、项目决策、行为纠正等），它会写入带类型的记忆文件，并在 `MEMORY.md` 索引中登记。下次启动时，智能体先读索引，再按需加载相关记忆后作答。

```
Session 1:                          Session 2（新 JVM 进程）:
  User: "I prefer concise answers"    Agent reads MEMORY.md
  Agent: saves feedback_style.md  →   Agent: already knows your preference
         updates MEMORY.md            "Here's the short version..."
```

这是与会话**对话历史**并存的**长期记忆**：

| | 对话历史 | AutoMemoryTools（长期） |
|---|---|---|
| 范围 | 当前会话 | 跨会话持久 |
| 存储 | 进程内（`ChatMemory`） | 磁盘文件 |
| 内容 | 完整消息往来 | 筛选后值得保留的事实 |
| 管理方 | Spring AI advisors | 智能体通过工具调用 |

## 快速开始

### 前置条件

- Java 17+
- Maven 3.6+
- 任一 AI 服务商 API 密钥

### 配置

1. 设置 API 密钥（默认配置为 Google GenAI）：
```bash
export GOOGLE_CLOUD_PROJECT=your-project-id
# 或使用 Anthropic — 见下文「切换 AI 服务商」
```

2. 运行示例：
```bash
mvn spring-boot:run
```

3. 开始对话：
```
USER> My name is Alice. I'm a backend engineer and I prefer short answers.
USER> Remember that we're migrating from PostgreSQL to CockroachDB this quarter.
USER> exit
```

4. 再次运行 — 智能体回忆已写入的内容：
```
USER> What do you know about me?
ASSISTANT> You're Alice, a backend engineer. You prefer short answers.
           You're migrating from PostgreSQL to CockroachDB this quarter.
```

## 项目结构

```
memory-tools-demo/
├── src/main/java/org/springaicommunity/agent/
│   ├── Application.java          # 主程序：装配 AutoMemoryTools 与聊天循环
│   └── MyLoggingAdvisor.java     # 将工具调用与响应打印到 stdout
├── src/main/resources/
│   └── application.properties    # 模型与记忆目录配置
```

`classpath:/prompt/MAIN_AGENT_SYSTEM_PROMPT_V2.md` 与 `classpath:/prompt/AUTO_MEMORY_TOOLS_SYSTEM_PROMPT.md` 从 `spring-ai-agent-utils` 依赖 jar 加载，无需本地副本。

## 工作原理

### 应用装配

```java
AutoMemoryTools memoryTools = AutoMemoryTools.builder()
    .memoriesDir(memoryDir)          // from application.properties
    .build();

ChatClient chatClient = chatClientBuilder
    .defaultSystem(p -> p
        .text(mainPrompt + "\n\n" + memoryToolsPrompt)
        .param("MEMORIES_ROOT_DIERCTORY", memoryDir))
    .defaultTools(
        memoryTools,                 // long-term memory
        TodoWriteTool.builder().build())
    .defaultAdvisors(
        ToolCallAdvisor.builder().build(),
        MyLoggingAdvisor.builder().build())
    .build();
```

### 记忆生命周期

智能体决定保存记忆时会自动发起两次工具调用：

**步骤 1 — `MemoryCreate`**：写入带 YAML 头信息的类型化 `.md` 文件：
```markdown
---
name: user profile
description: Alice — backend engineer, prefers short answers
type: user
---

Backend engineer named Alice.
Prefers concise, direct responses without trailing summaries.
```

**步骤 2 — `MemoryInsert`**：向 `MEMORY.md` 追加指针：
```markdown
- [User Profile](user_profile.md) — Alice, backend engineer, prefers short answers
```

下次会话中，智能体调用 `MemoryView("MEMORY.md", null)` 加载索引，再对相关条目调用 `MemoryView("user_profile.md", null)`。

### 记忆类型

| 类型 | 何时保存 | 示例 |
|---|---|---|
| `user` | 用户分享背景、目标或偏好 | 姓名、角色、沟通风格 |
| `feedback` | 用户纠正智能体或确认方案 | 「不要在结尾再总结」 |
| `project` | 项目决策、截止日、约束 | 迁移目标、冻结日期 |
| `reference` | 指向外部系统的引用 | Linear 看板、Grafana 仪表盘 |

## 配置

```properties
# application.properties

## AI provider
spring.ai.google.genai.project-id=${GOOGLE_CLOUD_PROJECT}
spring.ai.google.genai.chat.options.model=gemini-3.1-pro-preview

## Agent metadata (shown to the model in the system prompt)
agent.model=gemini-3.1-pro-preview
agent.model.knowledge.cutoff=Unknown

## Memory directory — persists across restarts
agent.memory.dir=${user.home}/.spring-ai-agent/memory-tools-demo/memory
```

`agent.memory.dir` 使用 Spring 的 `${user.home}`，可在任意机器解析到正确主目录。目录在首次运行时自动创建。

## 切换 AI 服务商

在 `pom.xml` 中取消注释所需 starter，并更新 `application.properties`：

**Anthropic Claude:**
```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-anthropic</artifactId>
</dependency>
```
```properties
spring.ai.anthropic.api-key=${ANTHROPIC_API_KEY}
spring.ai.anthropic.chat.options.model=claude-sonnet-4-6
agent.model=claude-sonnet-4-6
agent.model.knowledge.cutoff=2025-08-01
```

**OpenAI:**
```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-openai-sdk</artifactId>
</dependency>
```
```properties
spring.ai.openai-sdk.api-key=${OPENAI_API_KEY}
spring.ai.openai-sdk.chat.options.model=gpt-5-mini-2025-08-07
agent.model=gpt-5-mini-2025-08-07
agent.model.knowledge.cutoff=2025-08-07
```

## 相关文档

- [AutoMemoryTools 文档](../../../spring-ai-agent-utils/docs/AutoMemoryTools.md) — API、安全模型与系统提示说明
- [Claude Code — Memory](https://code.claude.com/docs/en/memory) — 本示例实现的基于文件的记忆模式
- [Claude API SDK — Memory Tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool) — 专用工具规范（本示例操作与之对应）
- [TodoWriteTool 文档](../../../spring-ai-agent-utils/docs/TodoWriteTool.md) — 本示例中包含的另一工具
- 同类示例（通用文件工具实现长期记忆）：[memory-filesystem-tools-demo](../memory-filesystem-tools-demo/README.zh-CN.md)
