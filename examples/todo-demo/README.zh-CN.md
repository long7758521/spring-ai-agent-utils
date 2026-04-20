# TodoWrite Demo

演示在 Spring AI 智能体中使用 `TodoWriteTool` 进行结构化任务管理。该工具使大模型能在执行过程中创建、跟踪和更新任务列表，将隐式规划变为可观察的工作流。

## 演示内容

- **任务跟踪**：智能体将复杂请求拆成可跟踪的待办项
- **进度状态**：任务在 `pending` → `in_progress` → `completed` 间流转
- **事件驱动更新**：通过 Spring 应用事件实时展示进度
- **单任务进行中**：同一时间仅允许一项为 `in_progress`

## 运行示例

```bash
# 为所选模型服务商设置 API 密钥
export GOOGLE_CLOUD_PROJECT=your-project-id
# 或: export ANTHROPIC_API_KEY=your-key
# 或: export OPENAI_API_KEY=your-key

# 可选：启用网页搜索
export BRAVE_API_KEY=your-brave-key

# 运行
./mvnw spring-boot:run -pl examples/todo-demo
```

## 示例运行

**提示：**
```
Find the top 10 Tom Hanks movies, then group them in groups of 2,
and finally print the title name inverted (e.g. last char first).
Use TodoWrite to organize your tasks.
```

**进度输出：**
```
Progress: 0/2 tasks completed (0%)
  [→] Find the top 10 Tom Hanks movies using WebSearch
  [ ] Group movies into pairs, reverse titles, and print the result

Progress: 1/2 tasks completed (50%)
  [✓] Find the top 10 Tom Hanks movies using WebSearch
  [→] Group movies into pairs, reverse titles, and print the result
```

**结果：**
```
Group 1
  pmuG tserroF (Forrest Gump)
  nayR etavirP gnivaS (Saving Private Ryan)

Group 2
  eliM neerG ehT (The Green Mile)
  yrotS yoT (Toy Story)

Group 3
  yawA tsaC (Cast Away)
  31 ollopA (Apollo 13)
...
```

## 核心组件

| 组件 | 作用 |
|------|------|
| `TodoWriteTool` | Spring AI 任务列表管理工具 |
| `TodoUpdateEvent` | 待办变更时发布的应用事件 |
| `TodoProgressListener` | 监听事件并展示进度 |

## 相关资源

- [TodoWriteTool.java](../../spring-ai-agent-utils/src/main/java/org/springaicommunity/agent/tools/TodoWriteTool.java)
- [Task 工具文档](../../spring-ai-agent-utils/docs/TaskTools.md)
- [博文：Spring AI Agentic Patterns - TodoWrite](https://spring.io/blog/2026/01/20/spring-ai-agentic-patterns-3-todowrite)
