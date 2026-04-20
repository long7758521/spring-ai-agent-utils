# Spring AI Agent Utils

<img style="display: block; margin: auto;" align="left" src="./docs/spring-ai-agent-utils-logo.png" width="200" />

面向 AI 智能体的 Spring AI 库，带来受 Claude Code 启发的工具与技能。

[Spring AI](https://docs.spring.io/spring-ai/reference/2.0-SNAPSHOT/index.html) Agent Utils 将 [Claude Code](https://code.claude.com/docs/en/settings#tools-available-to-claude) 的核心能力以 Spring AI 工具形式重新实现，支持文件操作、Shell 执行、网页访问、任务管理与可扩展技能等智能体工作流。

## Agentic 工具集

实现任意智能体行为所需的核心工具。

#### 核心工具

- **[AgentEnvironment](docs/AgentEnvironment.md)** — 动态智能体上下文工具，向系统提示词提供运行时环境与 Git 仓库状态
- **[FileSystemTools](docs/FileSystemTools.md)** — 精确控制的读、写、编辑文件
- **[ShellTools](docs/ShellTools.md)** — 带超时、后台进程与正则输出过滤的 Shell 执行
- **[GrepTool](docs/GrepTool.md)** — 纯 Java grep，支持正则、glob 与多种输出模式
- **[GlobTool](docs/GlobTool.md)** — 按 glob 快速匹配文件名
- **[SmartWebFetchTool](docs/SmartWebFetchTool.md)** — 带缓存的 AI 网页内容摘要
- **[BraveWebSearchTool](docs/BraveWebSearchTool.md)** — 支持域名过滤的网页搜索

#### 用户反馈

- **[AskUserQuestionTool](docs/AskUserQuestionTool.md)** — 执行过程中以多选项向用户澄清问题

#### 智能体技能

- **[SkillsTool](docs/SkillsTool.md)** — 用带 YAML 头信息的 Markdown 定义可复用、可组合的知识模块以扩展能力

#### 任务编排与多智能体

- **[TodoWriteTool](docs/TodoWriteTool.md)** — 带状态跟踪的结构化任务管理
- **[TaskTools](docs/TaskTools.md)** — 分层自主子智能体，将复杂任务委派给拥有独立上下文的专家智能体

这些工具可单独使用，但组合后更能体现智能体行为：SkillsTool 常与 FileSystemTools、ShellTools 配合完成领域工作流；BraveWebSearchTool 与 SmartWebFetchTool 提供真实世界信息；TaskTools 将工作委派给配备不同工具子集的子智能体。


## 安装

**Maven（推荐 — 使用 BOM）：**
```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springaicommunity</groupId>
            <artifactId>spring-ai-agent-utils-bom</artifactId>
            <version>0.7.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <dependency>
        <groupId>org.springaicommunity</groupId>
        <artifactId>spring-ai-agent-utils</artifactId>
    </dependency>
</dependencies>
```

**Maven（直接依赖）：**
```xml
<dependency>
    <groupId>org.springaicommunity</groupId>
    <artifactId>spring-ai-agent-utils</artifactId>
    <version>0.7.0</version>
</dependency>
```

_查看最新版本：_ [![](https://img.shields.io/maven-central/v/org.springaicommunity/spring-ai-agent-utils.svg?label=Maven%20Central)](https://central.sonatype.com/artifact/org.springaicommunity/spring-ai-agent-utils)

> **说明：** 需要 Spring AI `2.0.0-M4` 或更高版本。


## 快速开始

```java
@SpringBootApplication
public class Application {

    @Bean
    CommandLineRunner demo(ChatClient.Builder chatClientBuilder,
        @Value("${BRAVE_API_KEY}") String braveApiKey,
        @Value("${agent.skills.paths}") List<Resource> skillPaths,
        @Value("classpath:/prompt/MAIN_AGENT_SYSTEM_PROMPT_V2.md") Resource agentSystemPrompt) {

        return args -> {
            // Configure Task tool with Claude sub-agents
            var taskTool = TaskTool.builder()
                .subagentTypes(ClaudeSubagentType.builder()
                    .chatClientBuilder("default", chatClientBuilder.clone())
                    .skillsResources(skillPaths)
                    .braveApiKey(braveApiKey)
                    .build())
                .build();

            ChatClient chatClient = chatClientBuilder
                // Main agent prompt
                .defaultSystem(p -> p.text(agentSystemPrompt) // system prompt
                    .param(AgentEnvironment.ENVIRONMENT_INFO_KEY, AgentEnvironment.info())
                    .param(AgentEnvironment.GIT_STATUS_KEY, AgentEnvironment.gitStatus())
                    .param(AgentEnvironment.AGENT_MODEL_KEY, "claude-sonnet-4-5-20250929")
                    .param(AgentEnvironment.AGENT_MODEL_KNOWLEDGE_CUTOFF_KEY, "2025-01-01"))

                // Sub-Agents
                .defaultToolCallbacks(taskTool)

                // Skills
                .defaultToolCallbacks(SkillsTool.builder()
                    .addSkillsResources(skillPaths)
                    .build())

                // Core Tools
                .defaultTools(
                    ShellTools.builder().build(),
                    FileSystemTools.builder().build(),
                    GrepTool.builder().build(),
                    GlobTool.builder().build(),
                    SmartWebFetchTool.builder(chatClientBuilder.clone().build()).build(),
                    BraveWebSearchTool.builder(braveApiKey).build())

                // Task orchestration
                .defaultTools(TodoWriteTool.builder().build())

                // User feedback tool (use CommandLineQuestionHandler for CLI apps)
                .defaultTools(AskUserQuestionTool.builder()
                    .questionHandler(new CommandLineQuestionHandler())
                    .build())

                // Advisors
                .defaultAdvisors(
                    ToolCallAdvisor.builder().conversationHistoryEnabled(false).build(), // Tool Calling
                    MessageChatMemoryAdvisor.builder(MessageWindowChatMemory.builder().maxMessages(500).build()).build()) // Memory

                .build();

            String response = chatClient
                .prompt("Search for Spring AI documentation and summarize it")
                .call()
                .content();
        };
    }
}
```

### 示例

五个示例覆盖不同场景：

- **[code-agent-demo](../examples/code-agent-demo)** — 全功能 AI 编程助手、交互 CLI、全部工具、对话记忆与多模型
- **[skills-demo](../examples/skills-demo)** — SkillsTool 与自定义技能、辅助脚本
- **[subagent-demo](../examples/subagent-demo)** — 分层子智能体、自定义 Spring AI 专家子智能体与 TaskTools
- **[subagent-a2a-demo](../examples/subagent-a2a-demo)** — 本地 Claude 子智能体与远程 A2A 协议智能体组合
- **[ask-user-question-demo](../examples/ask-user-question-demo)** — 控制台聊天，演示 AskUserQuestionTool 单选/多选与自定义答案处理

详细搭建、配置与用法见 [示例 README](../examples/README.zh-CN.md)。


## 智能体工具详解

### 核心工具

### AgentEnvironment

通过动态系统提示词参数，向 AI 智能体提供运行时环境与 Git 上下文，使智能体具备环境感知能力。

[**完整文档 →**](docs/AgentEnvironment.md)

**简短示例：**
```java
import org.springaicommunity.agent.utils.AgentEnvironment;

@Value("${agent.model:Unknown}")
String agentModel;

@Value("${agent.model.knowledge.cutoff:Unknown}")
String agentModelKnowledgeCutoff;

@Value("classpath:/prompt/MAIN_AGENT_SYSTEM_PROMPT_V2.md")
Resource systemPrompt;

// Configure ChatClient with dynamic environment context
ChatClient chatClient = chatClientBuilder
    .defaultSystem(p -> p.text(systemPrompt)
        .param(AgentEnvironment.ENVIRONMENT_INFO_KEY, AgentEnvironment.info())
        .param(AgentEnvironment.GIT_STATUS_KEY, AgentEnvironment.gitStatus())
        .param(AgentEnvironment.AGENT_MODEL_KEY, agentModel)
        .param(AgentEnvironment.AGENT_MODEL_KNOWLEDGE_CUTOFF_KEY, agentModelKnowledgeCutoff))
    .defaultTools(/* your tools */)
    .build();
```

**系统提示词模板：** `src/main/resources/prompt/MAIN_AGENT_SYSTEM_PROMPT_V2.md`
```markdown
Here is useful information about the environment you are running in:
<env>
{ENVIRONMENT_INFO}
</env>
You are powered by the model: {AGENT_MODEL}

Assistant knowledge cutoff is {AGENT_MODEL_KNOWLEDGE_CUTOFF}.

{GIT_STATUS}
```

**application.properties：**
```properties
# AGENT CONFIGURATION
agent.model=claude-sonnet-4-5-20250929
agent.model.knowledge.cutoff=2025-09-29
```

**收益：**
- 智能体了解运行环境（操作系统、工作目录、日期）
- 具备 Git 感知：当前分支、未提交变更、最近提交
- 按模型配置知识截止日期
- 通过配置轻松支持多模型

### FileSystemTools

精确读、写、编辑文件。核心能力：Read（分页读）、Write（创建/覆盖）、Edit（带安全校验的字符串替换）。

[**完整文档 →**](docs/FileSystemTools.md)

**简短示例：**
```java
FileSystemTools fileTools = FileSystemTools.builder().build();

// Read a file
String content = fileTools.read("/path/to/file.txt", null, null);

// Edit with precise replacement
fileTools.edit(filePath, "oldValue", "newValue", null);
```

### ShellTools

支持后台进程的 Shell 执行。含 Bash（可选超时与后台）、BashOutput（正则过滤监控后台输出）、KillShell（优雅结束进程）。

[**完整文档 →**](docs/ShellTools.md)

**简短示例：**
```java
ShellTools shellTools = new ShellTools();

// Run command in background
String result = shellTools.bash("npm run dev", null, "Start dev server", true);
// Returns: "bash_id: shell_1234567890\n\nBackground shell started..."

// Monitor output with optional filtering
String output = shellTools.bashOutput("shell_1234567890", null);

// Kill background process
String killResult = shellTools.killShell("shell_1234567890");
```

### GrepTool

纯 Java grep，支持正则、glob 与多种输出模式，无需外部 ripgrep。

[**完整文档 →**](docs/GrepTool.md)

**简短示例：**
```java
GrepTool grepTool = GrepTool.builder().build();

// Search Java files for pattern
String result = grepTool.grep("public class.*", "./src", null,
    OutputMode.files_with_matches, null, null, null, null, null, "java", null, null, null);
```

### GlobTool

按文件名模式快速查找文件。纯 Java glob，结果可按修改时间排序。

[**完整文档 →**](docs/GlobTool.md)

**简短示例：**
```java
GlobTool globTool = GlobTool.builder().build();

// Find all Java files
String files = globTool.glob("**/*.java", "./src");

// Find TypeScript components
String components = globTool.glob("**/*Component.tsx", "./src");
```

### SmartWebFetchTool

带智能缓存与安全特性的 AI 网页抓取与摘要。抓取页面、HTML 转 Markdown，并按用户提示由 AI 提取相关信息。

[**完整文档 →**](docs/SmartWebFetchTool.md)

**简短示例：**
```java
// Build with required ChatClient
SmartWebFetchTool webFetch = SmartWebFetchTool.builder(chatClient)
    .maxContentLength(150_000)    // Optional: default 100KB
    .domainSafetyCheck(true)      // Optional: default true
    .maxRetries(2)                // Optional: default 2
    .build();

// Fetch and summarize web content
String result = webFetch.webFetch(
    "https://docs.spring.io/spring-ai/reference/",
    "What are the key features of Spring AI?"
);
```

### BraveWebSearchTool

基于 Brave Search API 的网页搜索，可选域名过滤。

[**完整文档 →**](docs/BraveWebSearchTool.md)

**简短示例：**
```java
// Build with API key
BraveWebSearchTool searchTool = BraveWebSearchTool.builder(apiKey)
    .resultCount(10)  // Optional: default 10
    .build();

// Search the web
String results = searchTool.webSearch(
    "Spring AI features 2025",
    null,  // allowedDomains (optional)
    null   // blockedDomains (optional)
);

// Or use search operators for efficiency
String results2 = searchTool.webSearch("Spring AI site:spring.io", null, null);
```
---

### 用户反馈

### AskUserQuestionTool

在智能体执行过程中向用户提出澄清问题。支持多选或自由文本，用于收集偏好、消除歧义与决策。

[**完整文档 →**](docs/AskUserQuestionTool.md)

**简短示例：**
```java
// For CLI applications, use the provided CommandLineQuestionHandler
AskUserQuestionTool askTool = AskUserQuestionTool.builder()
    .questionHandler(new CommandLineQuestionHandler())
    .build();

// Or implement a custom handler for web/GUI applications
AskUserQuestionTool customTool = AskUserQuestionTool.builder()
    .questionHandler(questions -> {
        // Display questions to user via your UI
        Map<String, String> answers = collectUserAnswers(questions);
        return answers;
    })
    .build();

// AI agent will automatically call this tool when it needs clarification
// Example: "Which framework should we use?" with options like React, Vue, Angular
```

**演示应用：**

完整控制台实现见 [ask-user-question-demo](../examples/ask-user-question-demo)，使用 `CommandLineQuestionHandler`。

---

### 智能体技能

### SkillsTool

用带 YAML 头信息的 Markdown 定义可复用、可组合的知识模块以扩展智能体能力。基于 [Claude Code Agent Skills](https://code.claude.com/docs/en/skills#agent-skills)，通过语义匹配触发专门任务。

[**完整文档 →**](docs/SkillsTool.md)

**简短示例：**
```java
// Register SkillsTool with skill directories
ChatClient chatClient = chatClientBuilder
    .defaultToolCallbacks(SkillsTool.builder()
        .addSkillsDirectory(".claude/skills")
        .build())
    .defaultTools(FileSystemTools.builder().build())  // For reading reference files
    .defaultTools(new ShellTools())       // For executing scripts
    .build();
```

**创建技能：** `.claude/skills/my-skill/SKILL.md`
```markdown
---
name: my-skill
description: What this skill does and when to use it. Include trigger keywords.
---

# My Skill
Instructions for the AI agent to follow...
```
---

### 任务编排与子智能体

### TodoWriteTool

面向 AI 编程会话的结构化任务列表。帮助智能体跟踪进度、组织复杂任务并展示执行可见性。

[**完整文档 →**](docs/TodoWriteTool.md)

**简短示例：**
```java
TodoWriteTool todoTool = TodoWriteTool.builder().build();

// Create and manage task list
Todos todos = new Todos(List.of(
    new TodoItem("Read configuration", Status.completed, "Reading configuration"),
    new TodoItem("Parse settings", Status.in_progress, "Parsing settings"),
    new TodoItem("Validate config", Status.pending, "Validating config")
));

todoTool.todoWrite(todos);
```

### TaskTools — 可扩展子智能体系统

将复杂多步任务委派给拥有独立上下文的专家子智能体。基于 [Claude Code sub-agents](https://code.claude.com/docs/en/sub-agents)，支持自主执行、领域专长与可插拔架构（多种子智能体类型）。

[**完整文档 →**](docs/TaskTools.md)

**简短示例：**
```java
import org.springaicommunity.agent.tools.task.TaskTool;
import org.springaicommunity.agent.tools.task.claude.ClaudeSubagentType;

// Configure Task tool with Claude sub-agents and multi-model support
var taskTool = TaskTool.builder()
    .subagentTypes(ClaudeSubagentType.builder()
        .chatClientBuilder("default", chatClientBuilder)
        .chatClientBuilder("opus", opusChatClientBuilder)     // Optional: for complex tasks
        .chatClientBuilder("haiku", haikuChatClientBuilder)   // Optional: for quick tasks
        .skillsResources(skillPaths)
        .build())
    .build();

// Build main chat client with Task tool
ChatClient chatClient = chatClientBuilder
    .defaultToolCallbacks(taskTool)
    .defaultTools(FileSystemTools.builder().build(), GrepTool.builder().build())
    .build();

// Agent automatically delegates to appropriate sub-agents
String response = chatClient
    .prompt("Explore the authentication module and explain how it works")
    .call()
    .content();
// Main agent recognizes exploration task and delegates to Explore sub-agent
```

**内置子智能体：**
- **general-purpose** — 复杂调研与多步任务，全工具
- **Explore** — 只读代码探索， thoroughness：quick/medium/very thorough
- **Plan** — 架构师型，设计方案、关键文件与权衡
- **Bash** — 终端与 git、构建等命令专家

**自定义子智能体：** `.claude/agents/code-reviewer.md`
```markdown
---
name: code-reviewer
description: Expert code reviewer. Use proactively after writing or modifying code.
tools: Read, Grep, Glob, Bash
disallowedTools: Edit, Write
model: sonnet
---

You are a senior code reviewer with expertise in code quality and security.

**Review Checklist:**
- Code clarity and readability
- Error handling and security
- Test coverage and performance

**Output Format:**
Organize feedback by priority: Critical Issues, Warnings, Suggestions.
```

**关键特性：**
- **多模型路由** — 按任务复杂度路由到 sonnet、opus、haiku 等
- **可扩展架构** — 可插拔 [子智能体 SPI](../spring-ai-agent-utils-common/README.zh-CN.md)，支持基于 Claude、[A2A](../spring-ai-agent-utils-a2a/README.zh-CN.md) 或自定义子智能体
- **独立上下文** — 各子智能体独立上下文，不污染主对话
- **工具过滤** — 通过 `tools` / `disallowedTools` 按子智能体限制工具
- **技能预载** — 通过头信息 `skills` 将技能注入子智能体系统提示词
- **后台执行** — 长任务可用 TaskOutputTool 异步运行
- **可恢复智能体** — 长周期调研可跨多轮继续


## 配置

**application.properties：**
```properties
# Model selection (supports Anthropic, OpenAI, Google)

# Anthropic Claude
spring.ai.anthropic.api-key=${ANTHROPIC_API_KEY}
spring.ai.anthropic.options.model=claude-sonnet-4-5-20250929

# or OpenAI 
spring.ai.openai-sdk.api-key=${OPENAI_API_KEY}
spring.ai.openai-sdk.chat.options.model=gpt-5-mini-2025-08-07
spring.ai.openai-sdk.chat.options.temperature=1.0

# or Google Gemini
spring.ai.google.genai.project-id=${GOOGLE_CLOUD_PROJECT}
spring.ai.google.genai.chat.options.model=gemini-3.1-pro-preview
spring.ai.google.genai.location=global

# Web tools (used by the BraveWebSearchTool )
brave.api.key=${BRAVE_API_KEY}
```

## 环境要求

- Java 17+
- Spring Boot 3.x / 4.x
- Spring AI 2.0.0-SNAPSHOT（> M1）

## 许可证

Apache License 2.0

## 相关模块

- [**spring-ai-agent-utils-common**](../spring-ai-agent-utils-common/README.zh-CN.md) — 共享子智能体 SPI（SubagentDefinition、SubagentResolver、SubagentExecutor、SubagentType）
- [**spring-ai-agent-utils-a2a**](../spring-ai-agent-utils-a2a/README.zh-CN.md) — 远程智能体编排的 A2A 协议子智能体实现
- [**spring-ai-agent-utils-bom**](../spring-ai-agent-utils-bom/pom.xml) — BOM，统一各模块版本

## 链接

- [GitHub 仓库](https://github.com/spring-ai-community/spring-ai-agent-utils)
- [Issue 跟踪](https://github.com/spring-ai-community/spring-ai-agent-utils/issues)
- [Spring AI 文档](https://docs.spring.io/spring-ai/reference/)
- 架构参考：
    - [Claude Code 文档](https://code.claude.com/docs/en/overview)
    - [Claude Code Agent Skills](https://code.claude.com/docs/en/skills#agent-skills)
    - [Claude Code Internals](https://agiflow.io/blog/claude-code-internals-reverse-engineering-prompt-augmentation/)
    - [Claude Code Skills](https://mikhail.io/2025/10/claude-code-skills/)
