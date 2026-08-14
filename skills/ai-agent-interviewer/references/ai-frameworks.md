# AI 框架对比

> 版本基准：2026 年 8 月。选型前务必核对官方最新版本号。

## 版本现状速览（2026-08）

| 框架 | 当前稳定版 | 1.0 GA 时间 | 备注 |
|------|-----------|------------|------|
| LangChain | langchain-core 1.4.x | 2025-10-22（与 LangGraph 同日） | AgentExecutor 已废弃 |
| LangGraph | 1.2.x | 2025-10-22 | 生产级 Agent 编排运行时 |
| LangChain4j | 1.17.x / 1.18.x | 2025-05 | 迭代极快，约一月一个 minor |
| Spring AI | 2.0.0（2026-06-12 GA） | 2025-05 | 2.0 面向 Boot 4.x；Boot 3.5 维护线为 1.1.x |
| Spring AI Alibaba | 1.1.2.x | — | 主流实践仍搭配 Spring AI 1.1.x + Boot 3.5.x |

## 一、LangChain（Python）

### 核心架构（1.x 时代）

LangChain 1.0 于 2025-10-22 GA，是一次架构级重构：

| 模块 | 职责 | 1.0 变化 |
|------|------|---------|
| Models | LLM/ChatModel 抽象，统一调用接口 | 保留，支持 Standard Content Blocks（跨厂商统一的推理轨迹/引用/工具调用表示） |
| Prompts | PromptTemplate 管理 | 保留 |
| LCEL | `prompt \| model \| parser` 管道语法 | 仍可用，但不再是推荐默认（推荐 create_agent） |
| Agents | 基于 Tool Calling 的自主决策 | **create_agent 成为新标准**，AgentExecutor 废弃 |
| Memory | 对话记忆管理 | **旧 Memory 类全部废弃**，改为 LangGraph checkpointer + middleware |
| Tools | 工具定义和执行 | 保留 `@tool` 装饰器 |
| Retrievers | RAG 检索器抽象 | 保留 |
| Middleware | Agent 循环前后钩子 | **1.0 新增**，一等公民 |

#### create_agent（新标准，底层是 LangGraph 运行时）
```python
from langchain.agents import create_agent
from langchain.tools import tool
from langgraph.checkpoint.memory import InMemorySaver

@tool
def get_weather(city: str) -> str:
    """获取指定城市的天气"""
    return fetch_weather(city)

agent = create_agent(
    model="openai:gpt-5.6",
    tools=[get_weather],
    system_prompt="你是一个天气助手",
    checkpointer=InMemorySaver(),  # 配置后即有持久化
)
result = agent.invoke(
    {"messages": [{"role": "user", "content": "北京天气"}]},
    config={"configurable": {"thread_id": "1"}},
)
```

#### Middleware（替代旧 Memory 的上下文管理）
```python
from langchain.agents.middleware import SummarizationMiddleware

agent = create_agent(
    model=model,
    tools=[],
    middleware=[
        SummarizationMiddleware(
            model=model,
            trigger=("tokens", 4000),   # 超过 4000 token 触发摘要压缩
            keep=("messages", 20),      # 保留最近 20 条消息
        )
    ],
)
```

#### AgentExecutor（已废弃，仅面试需了解）
```python
# ⚠️ langchain-classic 中的遗留 API，官方建议 2026-12 前迁移到 create_agent
from langchain_classic.agents import create_tool_calling_agent, AgentExecutor

agent = create_tool_calling_agent(llm, tools, prompt)
executor = AgentExecutor(agent=agent, tools=tools, verbose=True)
```
- 面试要点：AgentExecutor 是黑盒 Python 循环，无持久化、难拦截；create_agent 是图结构（LangGraph 运行时），middleware 可插入、checkpointer 原生支持、结构化输出在主循环内完成

#### 旧 Memory 模块（已废弃，仅面试需了解）
| 类型 | 特点 | 替代方案 |
|------|------|---------|
| ConversationBufferMemory | 保留全部对话，token 增长快 | checkpointer + SummarizationMiddleware |
| ConversationBufferWindowMemory | 只保留最近 K 轮 | SummarizationMiddleware 的 `keep` 参数 |
| ConversationSummaryMemory | 摘要历史对话 | SummarizationMiddleware |
| VectorStoreRetrieverMemory | 向量检索相关记忆 | 自研长期记忆层（框架至今没有好答案） |

**Memory 的坑（面试高频）**：
- BufferMemory 会导致 token 爆炸
- SummaryMemory 摘要质量依赖 LLM，可能丢失关键信息
- 不支持精细的上下文压缩策略（如分档水位）——这正是 1.0 用 middleware 重做的原因，但 middleware 也只是"触发式摘要"，离精细压缩仍有距离

### 何时用 LangChain
- ✅ 快速原型开发，不想自己写抽象层
- ✅ 需要 RAG + Agent 组合的复杂场景
- ✅ 团队不熟悉 Agent 内部机制，用框架降低门槛
- ❌ 需要精细控制上下文压缩、工具执行安全
- ❌ 需要极致性能（框架开销）
- ❌ 需要完全定制 Agent 循环的场合（此时直接用 LangGraph 或自研）

## 二、LangGraph（Python）

### 核心概念

LangChain 团队推出的**状态图运行时**，1.0 GA 于 2025-10-22。截至 2026 年是安装量最大的 Agent 框架（约 3450 万月度 PyPI 下载），Klarna、Uber、LinkedIn、Replit 等约 400 家公司在生产环境使用。

**关键认知**：LangChain 1.x 的 `create_agent` 底层就跑在 LangGraph 运行时上。两者是分层关系，不是替代关系——简单 Agent 用 create_agent，需要显式控制图拓扑时直接用 StateGraph。

#### 与 LangChain 的区别
| 维度 | LangChain（create_agent） | LangGraph（StateGraph） |
|------|--------------------------|------------------------|
| 执行模型 | 预置 Agent 循环 | 状态图（有向图 + 条件边） |
| 状态管理 | 内置（thread_id 会话） | 显式 State 对象 |
| 控制流 | 隐式（框架循环） | 显式（边 + 条件分支） |
| 适合场景 | 简单工具调用 Agent | 循环/复杂路由/多子图/HITL |

#### State Graph
```python
from langgraph.graph import StateGraph, END
from typing import TypedDict, Annotated
import operator

class AgentState(TypedDict):
    messages: Annotated[list, operator.add]
    next_step: str

graph = StateGraph(AgentState)
graph.add_node("research", research_node)
graph.add_node("draft", draft_node)
graph.add_node("review", review_node)

graph.add_edge("research", "draft")
graph.add_conditional_edges("draft", should_review, {
    "yes": "review",
    "no": END
})
graph.add_edge("review", "draft")  # 循环：审稿不过重新写

app = graph.compile(checkpointer=checkpointer)  # 生产换 Postgres checkpointer
result = app.invoke({"messages": [HumanMessage("写一篇关于 AI Agent 的文章")]},
                    config={"configurable": {"thread_id": "1"}})
```

#### 核心特性
- **循环执行**：支持循环边，适合 ReAct 模式
- **条件分支**：`add_conditional_edges` 根据状态动态路由
- **持久化**：`checkpointer` 保存中间状态，支持恢复；换 Postgres checkpointer 即得持久化、可恢复执行和"时间旅行"调试，图代码零改动
- **人机协作**：`interrupt` 暂停等待人工审批
- **多 Agent**：多个 Agent 节点组成图，支持协作与 handoff

#### 1.0 迁移注意
- `create_react_agent` 已废弃 → 用 `langchain.agents.create_agent`
- `MessageGraph`、`ValidationNode` 已废弃
- 最低 Python 3.10

### 适用场景
- 复杂多步骤工作流（研究 → 起草 → 审核 → 修改循环）
- 需要"人在回路"（Human-in-the-loop）的审批流程
- 需要 durable execution（服务器重启后从断点恢复）
- 多 Agent 协作编排

## 三、LangChain4j（Java）

### 定位

LangChain 的 Java 生态对应实现。**1.0 GA 于 2025 年 5 月**，此后迭代极快（约一月一个 minor），截至 2026 年中已到 1.17/1.18，是 Java AI 框架中迭代最快的。

### 核心模块
| 模块 | 对应 LangChain | 说明 |
|------|---------------|------|
| ChatLanguageModel | Models | 20+ LLM 提供商统一 API |
| Prompt | Prompts | 模板管理 |
| AiServices | Chains | 声明式接口绑定 |
| Tools | Tools | `@Tool` 注解，支持参数默认值 |
| ChatMemory | Memory | MessageWindowChatMemory 等 |
| EmbeddingModel / EmbeddingStore | Retrievers | 30+ 向量库 |
| langchain4j-agentic | LangGraph（部分） | 1.15 起独立的 Agentic 模块 |
| MCP | — | McpClient/McpToolProvider，支持 stdio/HTTP/WebSocket/Docker 传输 |

#### AiServices（核心抽象）
```java
interface Assistant {
    String chat(String userMessage);
}

Assistant assistant = AiServices.builder(Assistant.class)
    .chatLanguageModel(model)
    .chatMemory(MessageWindowChatMemory.withMaxMessages(20))
    .tools(weatherTool, calculatorTool)
    .build();

String result = assistant.chat("北京天气怎么样？");
```

#### 工具定义
```java
@Tool("获取指定城市的天气")
public String getWeather(@P("城市名") String city) {
    return fetchWeather(city);
}
```

#### langchain4j-agentic 模块（1.15+）
- `@Agent` 注解 + `AgenticServices` 构建有状态工作流
- 内置多 Agent 编排模式：Sequential（顺序）、Loop（循环）、Voting（投票，1.14）、Blackboard（黑板，1.16）、Debate（辩论，1.17）
- AgenticScope 共享状态，执行状态可持久化恢复（1.13）
- A2A 协议支持（Google Agent-to-Agent，1.16+）
- 工具补偿机制（1.17）：工具失败时自动调用声明的补偿操作

#### Guardrails API
- 输护栏：拦截用户消息（防注入、脱敏）
- 出护栏：校验模型响应
- 生产环境必备的双向防护

### 特点
- **注解驱动**：`@Tool`、`@P` 注解定义工具，类似 Spring 的风格
- **类型安全**：Java 强类型，编译期检查
- **框架集成**：Spring Boot / Quarkus / Micronaut / Helidon 均有支持
- **成熟度**：核心能力（RAG/工具调用/AI Services）已稳定，Agentic 模块部分功能仍带 beta 后缀

### 适用场景
- Java/JVM 技术栈的企业级应用
- 需要 MCP 协议和 A2A 多 Agent 互操作的企业系统
- 需要强类型和编译期检查的场景

## 四、Spring AI（Java）

### 定位

Spring 官方 AI 框架。**1.0 GA 于 2025 年 5 月，2.0.0 GA 于 2026-06-12**。2.0 面向 Spring Boot 4.x / Framework 7.0（全代码库 JSpecify 空安全、Jackson 3）；Spring Boot 3.5 对应的维护线为 1.1.x。

### 2.0 架构级升级（面试重点）
- 1.x：每个 ChatModel 实现内部各有一套私有工具调用循环，无法在循环上拦截、记账或替换执行策略
- 2.0：**工具调用循环提升为 Advisor 链上的一等公民**，同一套递归 Advisor 机制同时驱动工具循环、结构化输出重试和评估循环
- 模型接入收敛为官方 vendor SDK 路线：OpenAI、Anthropic、Bedrock、Google GenAI、Mistral、DeepSeek、Ollama（Anthropic 模块可指向任意兼容端点，常用于接入国产模型）

### 核心概念

#### ChatClient
```java
@RestController
public class ChatController {

    private final ChatClient chatClient;

    public ChatController(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }

    @GetMapping("/chat")
    public String chat(@RequestParam String message) {
        return chatClient.prompt()
            .user(message)
            .call()
            .content();
    }
}
```

#### Tool Calling（注意：`.functions()` 已改名 `.tools()`）
```java
@Bean
public Function<WeatherRequest, WeatherResponse> weatherFunction() {
    return request -> fetchWeather(request.city());
}

// 调用
chatClient.prompt()
    .user("北京天气怎么样？")
    .tools("weatherFunction")   // 1.0 GA 后 .functions() 已改为 .tools()
    .call()
    .content();
```

#### Advisor（理解 Spring AI 架构的入口）
- Advisor 链类似 Servlet Filter，可拦截 prompt 和响应
- 内置：ChatMemoryAdvisor（记忆）、RetrievalAugmentationAdvisor（RAG）、SafeGuardAdvisor（护栏）
- 2.0 中工具循环也跑在 Advisor 机制上

#### RAG 支持
```java
@Bean
public ChatClient chatClient(ChatClient.Builder builder, VectorStore vectorStore) {
    return builder
        .defaultRetrievalAugmenter(DefaultRetrievalAugmenter.builder()
            .vectorStore(vectorStore)
            .build())
        .build();
}
```

#### MCP 一等公民支持
- Boot Starter + Java 注解即可暴露/消费 MCP 工具

### 特点
- **Spring 风格**：Bean 注入、自动配置、starters
- **生态集成**：与 Spring Boot / Spring Cloud 无缝集成
- **向量存储**：30+ VectorStore（Redis/Pinecone/Milvus/PGVector 等）
- **Chat Memory**：JDBC/Cassandra/MongoDB/Redis 持久化

### Spring AI vs LangChain4j

| 维度 | Spring AI | LangChain4j |
|------|-----------|-------------|
| 出身 | Spring 官方（Broadcom/VMware） | 社区驱动 |
| 风格 | Spring Bean + Advisor 链 | 注解驱动 + AiServices |
| 生态 | Spring 生态深度集成 | 独立框架，Boot/Quarkus/Micronaut 都支持 |
| 成熟度 | 2.0 GA（2026-06），Boot 4.x；Boot 3.x 用 1.1.x | 1.17+，迭代最快 |
| 多 Agent | 基础（2.0 工具循环 Advisor 化） | agentic 模块 + A2A，模式更丰富 |
| 适用 | Spring Boot 深度用户 | 任何 Java 项目 |

## 五、Spring AI Alibaba（Java）

### 定位

阿里巴巴基于 Spring AI 扩展的 AI 框架，集成通义系模型和阿里云服务。当前稳定版 1.1.2.x（2026 年）。**注意：主流实践仍搭配 Spring AI 1.1.x + Boot 3.5.x 使用，对 Spring AI 2.0 / Boot 4 的适配进度以官方版本映射为准。**

### 核心扩展

#### 通义千问集成
```java
// application.yml
spring:
  ai:
    dashscope:
      api-key: ${DASHSCOPE_API_KEY}
      chat:
        options:
          model: qwen-plus
```

#### Agent Framework 与 Graph（对标 LangGraph）
- Graph 模块借鉴 LangGraph，提供 StateGraph 编排
- ReactAgent / CompiledGraph 开箱即用
- Human-in-the-loop 已进入框架层
- 1.1.2.0（2026-02）加入 Agent Skills、多智能体并行、并行条件边

#### 额外能力
| 能力 | 说明 |
|------|------|
| 通义千问模型 | qwen-plus / qwen-max / qwen-turbo / qwen3.7 系列 |
| DashScope Embedding | text-embedding-v3/v4 |
| 百炼平台集成 | 阿里云百炼模型管理 |
| RAG 增强 | GraphRAG / 多模态 RAG |
| Agent 编排 | Agent Framework + 可视化编排平台 |
| A2A 与 Nacos | 1.1.2.x 加入 A2A 协议和 Nacos 服务发现集成 |
| 阿里云 OSS | 文档存储集成 |

### 适用场景
- 使用阿里云生态的企业
- 需要通义千问模型的项目
- Java 栈但想要 LangGraph 风格 Graph 编排 + 开箱即用可视化平台的团队
- 国内合规要求的项目

## 六、框架对比总结

| 维度 | LangChain | LangGraph | LangChain4j | Spring AI | Spring AI Alibaba |
|------|-----------|-----------|-------------|-----------|-----------------|
| 语言 | Python | Python | Java | Java | Java |
| 定位 | 通用 Agent 框架 | 生产级编排运行时 | Java Agent 库 | Spring AI 集成 | 阿里扩展（含 Graph） |
| 控制流 | create_agent 循环 | 状态图 | 线性/agentic 模块 | Advisor 链 | StateGraph/ReactAgent |
| 成熟度 | 1.0 GA | 1.2.x，生产大规模验证 | 1.17+ | 2.0 GA | 1.1.2.x |
| 学习曲线 | 中等 | 较陡 | 低 | 低（Spring 用户） | 低 |
| 生产可用 | ✅（middleware 体系） | ✅（durable execution） | ✅ | ✅ | ⚠️ 依赖 Boot 3.x 线 |

### 何时用框架 vs 自研

| 场景 | 推荐 | 理由 |
|------|------|------|
| 快速原型 | LangChain（create_agent） | 开箱即用 |
| 复杂工作流/持久化/HITL | LangGraph | 状态图 + checkpointer 可控 |
| Spring 项目 | Spring AI | 生态集成 |
| Java 多 Agent/A2A | LangChain4j | agentic 模块模式丰富 |
| 阿里云生态/通义 | Spring AI Alibaba | Graph + 百炼一体化 |
| 生产级精细控制 | 自研 | 框架抽象泄漏，调试困难 |
| 极致性能 | 自研 | 去掉框架开销 |

## 七、常见面试题

### 基础题
1. LangChain 1.0 为什么废弃 AgentExecutor？create_agent 相比它解决了什么问题？
2. LangChain 旧的 Memory 模块（BufferMemory 等）有什么坑？1.0 之后用什么替代？
3. LangGraph 和 LangChain 的核心区别是什么？什么场景该用 LangGraph？
4. Spring AI 和 LangChain4j 各有什么优缺点？Spring Boot 项目该选哪个？

### 进阶题
5. LangChain 的 LCEL（管道语法 `prompt | model | parser`）底层是怎么实现的？1.0 之后它的定位发生了什么变化？
6. LangGraph 的 checkpointer 机制如何实现状态持久化？如何保证幂等性？换 Postgres checkpointer 为什么图代码零改动？
7. LangChain4j 的 `@Tool` 注解如何工作？langchain4j-agentic 模块提供了哪些多 Agent 编排模式？
8. Spring AI 2.0 把工具调用循环提升为 Advisor 链的一等公民，这带来了什么工程能力？

### 架构题
9. 如果你的项目需要"多 Agent 协作 + 人在回路审批 + 状态持久化"，该选 LangGraph、LangChain4j agentic 还是自研？权衡是什么？
10. Spring AI Alibaba 的通义千问集成相比直接调 DashScope API，多了什么能力？它的 Graph 模块和 LangGraph 有什么异同？什么情况下值得引入框架？
11. A2A（Agent-to-Agent）协议解决什么问题？LangChain4j 和 Spring AI Alibaba 的支持情况如何？
