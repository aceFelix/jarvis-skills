# 多模态与扩展生态

## 一、语音交互

### STT（语音识别）

#### 主流引擎对比
| 引擎 | 协议 | VAD | 实时性 | 适用场景 |
|------|------|-----|--------|---------|
| Paraformer | WebSocket 流式 | 客户端 RMS | 实时 | 通用场景 |
| QwenASR | WebSocket 流式 | 服务端 | 实时 | 中英混合高质量 |
| FunASR Flash | HTTP 文件上传 | 客户端 RMS | 非实时 | 批量转写 |
| Whisper | HTTP/流式 | 客户端 | 准实时 | 多语言 |
| DashScope Realtime | WebSocket 全双工 | 服务端 | 实时 | 全双工对话 |

#### VAD（语音活动检测）
- **客户端 VAD**：本地 RMS 能量检测，超阈值认为在说话，低于阈值静音 duration 后认为说完
- **服务端 VAD**：音频流持续推送到服务端，服务端自动检测说话开始/结束
- **权衡**：客户端 VAD 省带宽但不够精准；服务端 VAD 精准但需要持续推流

### TTS（语音合成）

#### 主流引擎
| 引擎 | 特点 | 音色 |
|------|------|------|
| CosyVoice | DashScope，流式合成 | longanlang_v3 等 |
| ElevenLabs | 高质量，多语言 | 自然度高 |
| Azure TTS | 微软云，多语言 | 神经语音 |
| MLX 本地 | 本地推理，离线可用 | 自定义 |

#### 流式 TTS
- LLM 每输出一个完整句子 → 立即送 TTS 合成 → 边来边播
- 替代"等 LLM 全文 → TTS 整段播放"的阻塞模式
- 降低首句延迟

### 全双工实时对话

#### 架构
```
麦克风 ──→ WebSocket 持续推送 PCM 音频 ──→ 实时语音服务端
                                           │
                                           ├── 服务端 VAD 检测说话开始/结束
                                           ├── 实时语音转录（文本显示）
                                           └── AI 语音流式返回 ──→ 扬声器播放
```

#### 关键特性
- **说话即打断**：用户说话时自动中断 AI 播报（服务端 VAD 检测）
- **同时听说**：全双工，不需要"说完→等回复→播报完→再听"
- **低延迟**：端到端响应时延控制在亚秒级

#### qwen-audio-3.0-realtime-flash
- DashScope 实时双工语音大模型
- 输入模态：文本 + 语音
- 输出模态：文本 + 语音
- 支持 function calling
- WebSocket 连接，session.update 配置 voice/turn_detection

### 唤醒词与待机机制
- **待机状态**：AI 不主动说话，等用户唤醒
- **唤醒词检测**：关键词匹配（如"贾维斯"）
- **待机意图识别**：LLM 检测 `<standby/>` 标记，用户说"退下/不聊了"时进入待机
- **防误触**：检测标记后验证正文长度，超过阈值不进入待机（防 LLM 误加标记）

## 二、视觉感知

### 截图工具
- 全屏截图 / 窗口截图 / 区域截图
- 回传给视觉模型（base64 编码，最长边 ≤1280）

### 摄像头与实时视觉
- 摄像头采集实时视频流
- MediaPipe 手势识别（21 个手部关键点）
- MediaPipe 人脸检测（468 个面部关键点）

### 屏幕变化监控
- 定时截图对比（像素差异检测）
- 检测到显著变化时触发告警

### 图片回传机制
- 工具输出 JSON 中提取图片路径
- base64 编码后作为 ImageContent 回传
- 视觉模型可直接"看到"渲染结果

## 三、工具生态

### MCP（Model Context Protocol）

#### 协议概述
- Anthropic 推出的开放通信协议
- 标准化 LLM 与外部工具的连接方式
- 支持 stdio 和 SSE 两种传输方式

#### 核心概念
- **Server**：提供工具能力的服务端
- **Client**：Agent 作为客户端连接 Server
- **Tool**：Server 暴露的具体能力
- **Resource**：Server 提供的资源（如文件、数据库）

#### 工程实现
- 配置文件：`~/.jarvis/mcp.json` 或项目级 `.mcp.json`
- 启动时自动发现并连接配置的 Server
- Server 进程通过 stdio 通信，JSON-RPC 协议

### CLI-Anything harness 模式

#### 核心思想
把任何桌面软件包装成 CLI 命令行工具，Agent 通过 CLI 调用软件 API。

#### 架构
```
SKILL.md (YAML frontmatter)      ← harness 能力描述
    ↓ loader 解析
Harness 对象 (id/name/command/args)
    ↓ registry 注册
CliAnythingTool (cli_anything__<id>)
    ↓ LLM 调用
runner.py 执行 (白名单校验)
    ↓
stdout JSON → 图片提取 → ImageContent 回传
```

#### SKILL.md 格式
```yaml
---
name: Blender
id: blender
description: 通过 CLI 控制 Blender 3D 建模软件
when_to_use: 用户需要创建/修改 3D 模型、渲染场景时
command: python
args:
  - name: operation
    type: string
    enum: [create_mesh, render, export, info]
    required: true
    positional: true      # 位置参数，不加 -- 前缀
    description: 操作类型
  - name: output_path
    type: string
    required: false
    description: 导出路径
examples:
  - "用 Blender 创建一个立方体"
---
```

#### 安全设计
- 默认 ASK 权限（外部命令不可信）
- 命令白名单前缀（python/node/cli-anything-/jarvis-harness-）
- 不走 shell（`create_subprocess_exec`）
- 超时强制终止（120 秒）
- 路径自动转换（Git Bash `/e/...` → Windows `E:\...`）

### 工具执行安全
- **并发安全分组**：同组工具串行执行，不同组并行
- **文件访问追踪**：记录工具访问了哪些文件
- **工具结果截断**：超长结果存文件，只回灌摘要

## 四、扩展机制

### Skills 技能包
- `SKILL.md` 格式：YAML frontmatter + Markdown 指令
- 触发词匹配或语义匹配
- 能力注入 system prompt
- 用户级（`~/.jarvis/skills/`）和项目级（`.jarvis/skills/`）

### Plugin 插件系统
- 插件市场（搜索/安装/卸载）
- 版本管理
- 配置文件：`~/.jarvis/plugins.json`

### Hook 钩子系统
- 事件驱动：工具调用前/后、会话开始/结束
- 拦截器模式：可修改或阻止工具执行
- 配置文件：`~/.jarvis/hooks.yaml`

## 五、多 Agent 协作

### 子代理（Subagent）
- 独立 QueryLoop（完整 Agent 循环）
- 工具子集（只暴露部分工具）
- 隔离权限（独立权限模式）
- 父 Agent 等待子 Agent 结果

### 团队协作（Team）
- 共享 TaskList（任务列表）
- 任务依赖链（A 完成后 B 才能开始）
- 文件邮箱通信（异步消息传递）
- Plan 模式（只读白名单规划，审批后执行）

### 消息通信
- Mailbox 邮箱：Agent 间异步消息
- 共享上下文：通过 TaskList 共享状态
- 广播 vs 点对点：广播成本高（N 个 Agent = N 次消息）

## 六、常见面试题

### 基础题
1. 客户端 VAD 和服务端 VAD 各有什么优缺点？
2. MCP 协议解决了什么问题？为什么不直接用 Function Calling？
3. CLI-Anything harness 模式相比 GUI 自动化有什么优势？

### 进阶题
4. 全双工语音对话中，"说话即打断"是如何实现的？服务端 VAD 的 silence_duration_ms 参数怎么调？
5. 设计一个 harness 的安全执行方案：如何防止命令注入？如何处理路径格式差异？
6. 子代理和团队协作各适用于什么场景？它们的通信机制有什么区别？

### 架构题
7. 如果让你设计一个支持 73 个桌面软件的 harness 生态，市场管理、版本更新、安全校验分别怎么做？
8. 多 Agent 协作中，如何保证任务依赖链的正确性？如何处理 Agent 执行失败的情况？
