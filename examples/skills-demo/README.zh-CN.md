# Skills Demo

聚焦演示 SkillsTool 系统，用于构建自定义 AI 智能体能力。

## 概述

本示例展示如何创建与使用智能体技能：在 Markdown 中定义的专业能力，可包含辅助脚本、参考文档与自定义工具限制。

## 功能

- **ai-tutor** 技能 — 生成讲解技术概念的教学用 PDF
- **pdf** 技能 — 从内容生成 PDF
- 结合 YouTube 字幕做调研型内容
- 基于 classpath 加载技能（适合打包应用）
- Python 辅助脚本扩展能力

## 快速开始

### 前置条件
- Java 17+
- Maven 3.6+
- AI 服务商 API 密钥（Anthropic、OpenAI 或 Google）
- Python 3.9+（抓取 YouTube 字幕）

### 配置

1. 设置 API 密钥：
```bash
export ANTHROPIC_API_KEY=your-key-here
# 或 OpenAI: export OPENAI_API_KEY=your-key-here
# 或 Google: export GOOGLE_CLOUD_PROJECT=your-project-id
```

2. 可选 — 网页搜索：
```bash
export BRAVE_API_KEY=your-brave-key
```

3. 运行示例：
```bash
mvn spring-boot:run
```

## 工作原理

演示对单一提示的执行流程：
1. 请求解释某技术概念（强化学习）
2. 引用 YouTube 视频作为支撑材料
3. 自动调用 ai-tutor 技能
4. 用 Python 获取 YouTube 字幕
5. 生成教学向说明
6. 可创建 PDF 教程

### 示例代码

```java
var answer = chatClient.prompt("""
    Explain reinforcement learning in simple terms and use.
    Use required skills.
    Then use the Youtube video https://youtu.be/vXtfdGphr3c transcript
    to support your answer.
    """).call().content();
```

## 技能配置

技能从 `src/main/resources/.claude/skills/` 的 classpath 资源加载：

```
src/main/resources/.claude/skills/
├── ai-tutor/
│   ├── SKILL.md                    # 技能定义
│   ├── REFERENCE.md                # 参考文档
│   └── scripts/
│       └── get_youtube_transcript.py
└── pdf/
    ├── SKILL.md
    └── scripts/
```

在 `application.properties` 中配置：
```properties
agent.skills.dirs=classpath:/.claude/skills
```

## 创建自己的技能

1. 在 `.claude/skills/` 下新建目录
2. 添加带 YAML 头信息的 `SKILL.md`：

```markdown
---
name: my-skill
description: What the skill does and when to use it
allowed-tools: Read, Bash, Grep
model: claude-sonnet-4-5-20250929
---

# Skill Instructions
Your detailed prompt for the AI agent...
```

3. 可选：添加辅助脚本与参考文件

## 架构

演示使用：
- **SkillsTool** — 加载与管理智能体技能
- **ShellTools** — 执行 Shell（运行 Python 脚本）
- **FileSystemTools** — 读写文件（生成 PDF）
- **SmartWebFetchTool** — 抓取网页
- **BraveWebSearchTool** — 网页搜索

## 切换 AI 服务商

修改 `pom.xml` 与 `application.properties`：

**Anthropic Claude**（默认）：
```properties
spring.ai.anthropic.api-key=${ANTHROPIC_API_KEY}
spring.ai.anthropic.chat.options.model=claude-sonnet-4-5-20250929
```

**OpenAI**：
```properties
spring.ai.openai-sdk.api-key=${OPENAI_API_KEY}
spring.ai.openai-sdk.chat.options.model=gpt-5-mini-2025-08-07
```

**Google Gemini**：
```properties
spring.ai.google.genai.project-id=${GOOGLE_CLOUD_PROJECT}
spring.ai.google.genai.chat.options.model=gemini-3.1-pro-preview
```

## 延伸阅读

- [SkillsTool 文档](../../spring-ai-agent-utils/docs/SkillsTool.md)
- [示例总览](../README.zh-CN.md)
- [Spring AI 文档](https://docs.spring.io/spring-ai/reference/)

## 许可证

Apache License 2.0
