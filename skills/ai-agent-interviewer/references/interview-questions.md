# AI Agent 面试题库

## 初级（0-1 年）

### Agent 基础

1. 什么是 Function Calling？它解决了什么问题？
2. 什么是 Tool Calling？它和 Function Calling 有什么区别？
3. ReAct 模式的基本流程是什么？Thought-Action-Observation 循环怎么工作？
4. LLM 的上下文窗口是什么意思？超出会怎样？
5. 为什么 Agent 需要工具？不能直接靠模型自身回答吗？
6. Prompt 工程中，System Prompt 和 User Prompt 有什么区别？
7. Few-shot 和 Zero-shot 各适用于什么场景？
8. 什么是 token？为什么 Agent 要关注 token 消耗？

### RAG 基础

9. RAG 是什么？它解决了 LLM 的什么问题？
10. RAG 的基本流程是什么？（加载→分片→Embedding→存储→检索→生成）
11. 什么是 Chunking？为什么需要分片？
12. Embedding 是什么？它的作用是什么？
13. 向量数据库和传统数据库有什么区别？
14. 什么是 Top-K 检索？

### 框架基础

15. LangChain 的核心模块有哪些？
16. LangChain 的 Memory 模块有什么作用？
17. SpringAI 和 LangChain4j 各是什么定位？
18. 什么是 LCEL（LangChain 表达式语言）？

## 中级（1-3 年）

### Agent 运行时

19. 流式工具调用解析中，`input_json_delta` 是什么？为什么要累积拼接？
20. 输出截断自动续写的原理是什么？如何检测截断？
21. 工具结果截断落盘策略是什么？为什么要截断？
22. Plan-Execute 模式和 ReAct 模式各适用于什么场景？
23. 推理模型（deepseek-v4-pro 开启思考模式）的 reasoning_content 通道如何处理？为什么不进 TTS？
24. 设计一个 Agent 的工具注册中心，需要考虑哪些问题？

### 上下文管理

25. 上下文压缩的水位触发机制怎么设计？三档水位各做什么？
26. 结构化摘要提示词的九段结构是什么？为什么要分段？
27. 压缩后文件回灌策略是什么？为什么要限制回灌数量？
28. 会话崩溃恢复的原子性如何保证？
29. 长期记忆和工作记忆的区别是什么？如何同时支持？
30. MEMORY.md 模式和向量检索记忆各有什么优缺点？

### 工具生态

31. MCP 协议解决了什么问题？stdio 和 SSE 传输各有什么优缺点？
32. CLI-Anything harness 模式的核心思想是什么？
33. harness 的 SKILL.md 里 positional 参数是什么意思？什么时候用？
34. 命令白名单前缀机制如何防止命令注入？
35. 工具执行的并发安全分组是什么？为什么要分组？

### 安全权限

36. fail-closed 管线的五层分别是什么？
37. 四种权限模式（DEFAULT/PLAN/ACCEPT_EDITS/YOLO）各适用于什么场景？
38. 符号链接逃逸攻击是什么？如何防御？
39. 为什么用 `create_subprocess_exec` 而不是 `create_subprocess_shell`？
40. 权限规则合并优先级 DENY > ASK > ALLOW 的理由是什么？

### RAG 进阶

41. 递归分片器的九级语义优先级降级是什么？为什么需要重叠机制？
42. Chunk 选 500 字符的权衡是什么？
43. HyDE 的原理是什么？它解决了什么问题？
44. 为什么需要 Rerank？选 DashScope Rerank 而非 BGE 的理由是什么？
45. 检索准确率 90% 这个数字如何诚实报告？
46. 混合检索（稠密 + 稀疏）的加权融合怎么做？
47. 上下文组装时为什么要做窗口扩展？

### 框架进阶

48. LangChain 的 BufferMemory 在长对话中会出什么问题？有哪些替代方案？
49. LangGraph 的状态图和 LangChain 的线性 Chain 有什么本质区别？
50. LangGraph 的 checkpointer 机制如何实现状态持久化？
51. LangChain4j 的 `@Tool` 注解和 Python LangChain 的 `@tool` 有什么区别？
52. SpringAI 的 ChatClient 和 LangChain4j 的 AiServices 各有什么优缺点？
53. 什么时候该用框架，什么时候该自研？给出判断标准。

### 多模态

54. 客户端 VAD 和服务端 VAD 各有什么优缺点？
55. 全双工语音对话中，"说话即打断"是如何实现的？
56. qwen-audio-3.0-realtime-flash 的 session.update 配置有哪些关键参数？
57. 流式 TTS 的"句子级流式播放"是怎么工作的？
58. 图片回传给视觉模型的 base64 编码有什么限制？最长边为什么要 ≤1280？

## 高级（3-5 年）

### 架构设计

59. 从零设计一个 Agent 运行时，核心模块有哪些？数据流是怎样的？
60. 如何设计一个支持多厂商 LLM 的抽象层？api_format 规范化怎么做？
61. 设计一个 Agent 的上下文压缩策略：128K 窗口，50 轮对话，每轮 2000 token。
62. 如何衡量一个 Agent 系统的"智商"？除了准确率还有哪些指标？
63. Agent 系统的"首 Token 延迟 <2s"如何测量？是端到端还是仅 LLM？
64. 多 Agent 协作中，任务依赖链如何保证正确性？Agent 执行失败如何处理？

### 工程实战

65. 73 个桌面软件的 harness 生态，市场管理、版本更新、安全校验分别怎么做？
66. harness 路径格式差异（Git Bash `/e/...` vs Windows `E:\...`）如何自动转换？
67. Windows 控制台 GBK 编码导致 JSON 含 emoji 时 UnicodeEncodeError，如何修复？
68. LLM 切换到 DeepSeek 后语音模块 401 错误，根因是什么？如何隔离 API Key？
69. 设计一个 harness 的 SKILL.md 参数 schema，支持位置参数和命名参数。

### RAG 架构

70. 设计一个企业级 RAG 系统：10 万文档、1000 并发、P99 <500ms。
71. 检索结果质量很差，从哪些环节排查？给出诊断流程。
72. GraphRAG 和传统 RAG 的本质区别？什么场景有明显优势？
73. Multi-modal RAG 中图片如何做 Embedding？表格如何处理？
74. Self-RAG 的自主决策机制如何设计？如何评估检索结果质量？
75. RAG 系统的知识更新策略？热更新还是批量重建？

### 框架架构

76. LangChain 的 LCEL 底层是怎么实现的？有什么性能开销？
77. LangGraph 的条件分支和循环执行在工程上有什么坑？
78. SpringAI Alibaba 相比直接调 DashScope API，多了什么能力？
79. 如果项目需要"多 Agent + 人在回路 + 状态持久化"，选 LangGraph 还是自研？
80. LangChain 的抽象泄漏在哪些场景会暴露？如何应对？

## 架构师级（5 年+）

### 系统设计

81. 设计一个支持多模态的智能客服 Agent，日活 10 万，首 Token <2s。
82. 设计一个 Agent 系统的权限规则引擎，支持哪些匹配模式？如何保证可预测性？
83. 设计一个 Agent 的记忆系统，同时支持工作记忆、短期记忆、长期记忆、程序性记忆。
84. Agent 系统的"主动感知"如何设计？从被动响应到主动预测的架构演进路径？
85. 设计一个跨平台 Agent 系统（Windows/macOS/Linux），如何处理平台差异？

### 深度追问

86. 乐观锁解决超卖：WHERE 条件怎么写？行锁持有多久？5000 并发推演。
87. 为什么不用 version 做乐观锁？三种失败原因分析。
88. 订单状态流转的 6 个状态 + 支付状态如何设计？定时任务兜底机制？
89. Redis + Spring Cache 双缓存的延迟双删怎么实现？缓存穿透如何防御？
90. Prompt 工程的六层 System Prompt 各是什么？为什么这么分层？
91. AI 智能客服的 Tool Schema 设计：返回值为什么用自然语言而非 JSON？
92. AI 情感分析 + 转人工：关键词快速通道 + LLM 语义兜底 + 主动探测三层递进怎么设计？
93. 自研递归分片器的 9 级语义优先级降级 + 50 字符重叠机制的工程实现。
94. Rerank 选 DashScope 而非 BGE 的三个理由（技术栈一致性、骨干网延迟、运维成本）。
95. 检索准确率 90% 基于 15 个测试问题——如何诚实报告？局限性是什么？

### 前沿与趋势

96. MCP 协议的标准化对 Agent 生态意味着什么？会统一工具调用层吗？
97. 实时双工语音大模型（qwen-audio-3.0-realtime-flash）对 Agent 交互模式的影响？
98. Agent 的"记忆压缩"和"记忆遗忘"如何平衡？什么时候该忘记？
99. 多 Agent 系统的"涌现行为"如何预测和控制？
100. 从"工具调用"到"自主决策"：Agent 的自主性边界在哪里？如何设定红线？

## 压力追问题（深度考核）

这些题不是独立题目，而是针对候选人回答的追问模板：

### 针对"用 RAG"
- 检索准确率怎么测？chunk 选多大？为什么不用 BGE？
- 如果检索结果全是噪声，你怎么判断是分片问题还是 Embedding 问题？
- 你的 RAG 系统知识更新频率是多少？热更新还是批量重建？

### 针对"用 LangChain"
- LangChain 的 Memory 模块有什么坑？什么时候不该用 LangChain？
- LCEL 的 `|` 管道符底层怎么实现？流式输出怎么处理？
- 生产环境遇到过 LangChain 的什么坑？怎么解决的？

### 针对"用乐观锁"
- WHERE 条件怎么写？`stock >= #{count}` 的行锁持有多久？
- 5000 并发抢库存，乐观锁会不会有很多重试？重试几次合适？
- 为什么不用 version 做乐观锁？三种失败怎么区分？

### 针对"用 WebSocket"
- WebSocket 和 SSE 各适用于什么场景？为什么选其中一个？
- WebSocket 的心跳机制怎么设计？断线重连策略？
- 全双工 WebSocket 的并发控制怎么做？如何防止消息乱序？

### 针对"自研框架"
- 为什么不直接用 LangChain？自研的好处和代价各是什么？
- 自研框架的抽象层设计：Provider 适配、工具注册、权限管线各怎么分层？
- 自研框架的测试策略？如何保证 190 个文件的代码质量？
