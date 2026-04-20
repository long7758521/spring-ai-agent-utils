# Spring AI Agent Utils - 示例

本目录包含使用 spring-ai-agent-utils 库构建 AI 智能体的完整示例。

## 可用示例

### [Code Agent Demo](code-agent-demo)

功能完整的 AI 编程助手，带交互式命令行界面，灵感来自 Claude Code。

完整说明见 [Code Agent Demo README](code-agent-demo/README.zh-CN.md)。

---

### [Ask User Question Demo](ask-user-question-demo)

演示 `AskUserQuestionTool`：通过结构化问题与多选项，实现智能体与用户的交互式沟通。

完整说明见 [Ask User Question Demo README](ask-user-question-demo/README.zh-CN.md)。

---

### [Sub-Agent Demo](subagent-demo)

演示分层子智能体系统：使用 Markdown 定义的本地子智能体，以及 TaskTool 分发模式。

架构说明见 [Sub-Agent Demo README](subagent-demo/README.zh-CN.md)。

---

### [Sub-Agent A2A Demo](subagent-a2a-demo)

在子智能体系统上扩展 [A2A（Agent-to-Agent）协议](https://google.github.io/A2A/) 支持，通过 HTTP 将任务委派给远程智能体。

搭建步骤见 [Sub-Agent A2A Demo README](subagent-a2a-demo/README.zh-CN.md)。

---

### [Skills Demo](skills-demo)

聚焦演示 SkillsTool 系统及自定义技能开发。

完整说明见 [Skills Demo README](skills-demo/README.zh-CN.md)。

---

### [Todo Demo](todo-demo)

演示 `TodoWriteTool`：在智能体中进行结构化任务管理。展示大模型如何在执行过程中创建、跟踪和更新任务列表，并通过 Spring 应用事件实时展示进度。

完整说明见 [Todo Demo README](todo-demo/README.zh-CN.md)。

## 环境要求

所有示例需要：

- Java 17 或更高版本
- Maven 3.6+
- 至少一个 AI 服务商的 API 密钥（Anthropic、OpenAI 或 Google）
- 可选：Brave API 密钥（用于网页搜索）

### 构建

在项目根目录执行：

```bash
mvn clean install
```

## 文档

- [spring-ai-agent-utils 库](../spring-ai-agent-utils/README.zh-CN.md)
- [工具文档](../spring-ai-agent-utils/docs/)
- [Spring AI 参考文档](https://docs.spring.io/spring-ai/reference/)

## 许可证

Apache License 2.0
