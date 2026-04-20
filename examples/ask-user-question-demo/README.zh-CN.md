# AskUserQuestionTool Demo

Spring Boot 控制台应用，演示 [AskUserQuestionTool](../../spring-ai-agent-utils/docs/AskUserQuestionTool.md)：让 AI 智能体在执行过程中向用户提出澄清性问题。

## 概述

本示例展示智能体如何通过带多选项的提问，交互式收集用户偏好、澄清模糊指令并辅助实现决策。

应用以控制台聊天方式运行。当智能体需要更多信息才能完成请求时，会自动调用 `AskUserQuestionTool`，以预定义选项提问，或允许自由文本输入。

## 功能

- 基于控制台的交互式智能体对话
- 需要澄清时自动弹出提问
- 支持单选与多选
- 支持在选项之外输入自由文本
- 对话记忆，在多轮交互中保持上下文

## 前置条件

- Java 17 或更高
- Maven 3.6 或更高
- Anthropic API 密钥（或配置其它 AI 服务商）

## 配置

将 API 密钥设为环境变量：

```bash
export ANTHROPIC_API_KEY=your-api-key-here
```

### 其它 AI 服务商

示例默认使用 Anthropic Claude，可在 [application.properties](src/main/resources/application.properties) 中取消注释相应段落以切换：

- **OpenAI**：取消注释 OpenAI SDK 配置与依赖
- **Google GenAI**：取消注释 Google GenAI 配置与依赖

## 运行示例

在项目根目录：

```bash
cd examples/ask-user-question-demo
mvn spring-boot:run
```

或打包后运行 JAR：

```bash
mvn clean package
java -jar target/chat-demo2-0.1.0-SNAPSHOT.jar
```

## 用法

启动后会看到问候并等待输入：

```
I am your assistant.

USER:
```

### 示例交互

**示例 1：选择技术栈**

```
USER: Help me choose a database for my new web application

ASSISTANT: I'd be happy to help you choose a database. Let me ask you a few questions to understand your needs better.

Database Type: What type of database do you prefer?
  1. PostgreSQL - Powerful open-source relational database with ACID compliance
  2. MongoDB - Flexible document-oriented NoSQL database
  3. Redis - In-memory key-value store for caching and real-time data
  (Enter a number, or type custom text)
1

Scale: What scale are you expecting?
  1. Small - < 1000 users, simple queries
  2. Medium - 1000-100K users, moderate complexity
  3. Large - > 100K users, high performance needs
  (Enter a number, or type custom text)
2

ASSISTANT: Based on your requirements, PostgreSQL is an excellent choice...
```

**示例 2：多选**

```
USER: I'm building a REST API. What features should I include?

ASSISTANT: Let me help you decide on the features.

Features: Which features do you want to enable?
  1. Authentication - User login and registration with JWT
  2. Database - PostgreSQL integration with JPA
  3. Caching - Redis caching layer for performance
  4. Monitoring - Health checks and metrics endpoints
  (Enter numbers separated by commas, or type custom text)
1,2,4

ASSISTANT: Great choices! I'll help you set up authentication, database integration, and monitoring...
```

## 工作原理

演示通过控制台问答处理器使用 `AskUserQuestionTool`：

1. 智能体判断需要用户输入，调用 `AskUserQuestionTool`
2. 工具通过自定义 `handleQuestions` 传入问题列表
3. 在控制台展示编号选项
4. 用户输入选择（数字或自由文本）
5. 解析答案并返回给智能体
6. 智能体根据收集到的信息继续执行

实现要点见 [Application.java](src/main/java/org/springaicommunity/agent/Application.java)：

- **约 55–58 行**：工具配置与自定义问答处理器
- **约 81–106 行**：控制台展示问题与收集答案
- **约 108–125 行**：解析响应（将数字映射为选项文案或使用自由文本）

## 自定义

### 关闭答案校验

默认会校验所有问题都已回答。若允许部分回答，构建工具时设置 `answersValidation(false)`（见 [Application.java 第 57 行附近](src/main/java/org/springaicommunity/agent/Application.java#L57)）。

### 调整对话记忆

示例保留最近 500 条消息（约第 64 行）。可按需修改：

```java
MessageChatMemoryAdvisor.builder(
    MessageWindowChatMemory.builder()
        .maxMessages(100)  // 修改此值
        .build()
).build()
```

### 更换 AI 服务商

在 [application.properties](src/main/resources/application.properties#L7) 中修改模型与服务商配置，并同步调整 `pom.xml` 依赖。

## 延伸阅读

- [AskUserQuestionTool 文档](../../spring-ai-agent-utils/docs/AskUserQuestionTool.md)
- [Spring AI 文档](https://docs.spring.io/spring-ai/reference/)
- [Claude Agent SDK - User Input](https://platform.claude.com/docs/en/agent-sdk/user-input#question-format)

## 项目结构

```
ask-user-question-demo/
├── pom.xml                           # Maven 配置
├── src/
│   └── main/
│       ├── java/
│       │   └── org/springaicommunity/agent/
│       │       └── Application.java  # 主程序与问答处理
│       └── resources/
│           └── application.properties # 配置
├── README.md
└── README.zh-CN.md
```
