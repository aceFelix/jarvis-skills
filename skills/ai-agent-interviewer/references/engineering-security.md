# 工程实现与安全权限

## 一、多厂商 LLM 抽象

### Provider 适配层

#### 常见 Provider
| Provider | 协议 | 特点 |
|----------|------|------|
| OpenAI | OpenAI API | 行业标准，兼容最多 |
| Anthropic | Messages API | system 单独传、无 system role in messages |
| DashScope | OpenAI 兼容 + 原生 | Qwen 系列，部分模型走原生 |
| ZAI | OpenAI 兼容 | 智谱 GLM 系列 |

#### api_format 规范化
不同 Provider 的请求格式差异：
- **OpenAI**：`messages: [{role, content}]`，content 可以是 string 或 array
- **Anthropic**：`system` 单独传，`messages` 里无 system role；content 必须是 array
- **DashScope 原生**：`input: {messages}`，部分模型参数名不同

工程实现：
```python
# 统一抽象
class LLMProvider:
    async def chat(self, messages, model, **kwargs) -> ChatResponse:
        # 1. 规范化 messages（处理 system role 差异）
        # 2. 转换为 Provider 特定格式
        # 3. 发送请求
        # 4. 规范化响应
```

### 流式 SSE 解析差异
- **OpenAI**：`data: {"choices": [{"delta": {"content": "..."}}]}`
- **Anthropic**：`event: content_block_delta\ndata: {"delta": {"text": "..."}}`
- **DashScope**：`data: {"output": {"choices": [{"message": {"content": "..."}}]}}`

### 模型热切换
- 运行时切换 Provider 和模型（`/model` 命令）
- API Key 隔离：语音模块固定用 DashScope Key，不随 LLM 切换变化
- 成本优化：贵模型规划，便宜模型执行

### 前缀缓存与成本优化
- **前缀缓存**：相同 system prompt 复用，减少重复计算
- **延迟加载**：大文件按需读取，不一次性注入上下文
- **模型分级**：简单任务用 flash 模型，复杂任务用 pro 模型

## 二、文件与代码智能

### 文件操作

#### mtime 缓存
- 读取文件时记录 mtime（修改时间）
- 编辑前检查 mtime 是否变化，变了说明外部修改了，拒绝覆盖
- 避免覆盖用户在编辑器中的未保存修改

#### 冲突检测
- old_string 必须在文件中唯一
- 不唯一时要求加更多上下文
- 精确匹配包括空白字符

### LSP 集成

#### 九种操作
| 操作 | 用途 |
|------|------|
| documentSymbol | 了解文件结构 |
| definition | 跳转定义 |
| references | 查引用 |
| typeDefinition | 类型信息 |
| hover | 悬停信息 |
| callHierarchy | 调用层次 |
| completion | 代码补全 |
| rename | 重命名 |
| codeAction | 快速修复 |

#### 工程使用
- 写代码前先用 `documentSymbol` 了解结构
- 修改前用 `findReferences` 看影响范围
- 不确定类型时用 `hover`

### Shell 执行

#### Git Bash 适配
- Windows 上用 `bash -c` 而非 `cmd /c`
- 自动检测 Git Bash 路径（PATH / 常见安装路径）
- 路径格式：Bash 用 `/e/...`，文件工具用 `E:\...`

#### 命令分类
- **readonly**：ls/cat/grep 等，自动放行
- **dangerous**：rm/drop 等，需要确认
- **unknown**：默认询问

## 三、安全权限

### 五层 fail-closed 管线

```
工具调用请求
    ↓
① 工具特判（白名单/黑名单）
    ↓ 拒绝
② 路径守护（危险目录检测）
    ↓ 拒绝
③ 模式覆写（PLAN 模式只读）
    ↓ 拒绝
④ 规则匹配（permissions.yaml）
    ↓ 无规则
⑤ 默认 ASK（询问用户）
```

**fail-closed 原则**：任何未明确允许的操作都默认拒绝或询问，绝不默认放行。

### 四种权限模式

| 模式 | 行为 | 适用场景 |
|------|------|---------|
| DEFAULT | 只读自动放行，写操作询问 | 日常开发 |
| PLAN | 只读白名单，禁止所有写操作 | 规划阶段 |
| ACCEPT_EDITS | 文件编辑自动放行，其他写操作仍询问 | 批量改代码 |
| YOLO | 所有操作自动放行 | 信任环境/演示 |

### 路径守护

#### 危险目录拒绝
- `~/.ssh/`、`~/.aws/`、`/etc/`、`C:\Windows\System32`
- 拒绝读写这些目录下的文件

#### 符号链接逃逸检测
- 攻击者可能创建符号链接指向危险目录
- 读取文件前解析真实路径（`os.path.realpath`），检查是否在安全范围内

### 命令注入防御

#### 白名单前缀
```python
_ALLOWED_COMMAND_PREFIXES = (
    "python", "node", "npm", "npx", "cli-anything-", "jarvis-harness-"
)
```
- 命令必须以白名单前缀开头
- 不走 shell（`create_subprocess_exec` 而非 `create_subprocess_shell`）
- 参数用列表传递，不用字符串拼接

#### 超时强制终止
- 默认 120 秒超时
- 超时后 `proc.kill()` 终止进程
- 避免恶意命令卡死 Agent

### 权限规则配置（permissions.yaml）

```yaml
rules:
  - pattern: "git *"
    action: ALLOW
  - pattern: "npm install *"
    action: ALLOW
  - pattern: "rm -rf *"
    action: DENY
  - pattern: "curl *"
    action: ASK
```

合并优先级：DENY > ASK > ALLOW

## 四、常见面试题

### 基础题
1. 为什么 Agent 的权限管线要 fail-closed 而不是 fail-open？
2. ACCEPT_EDITS 模式下，哪些操作仍然需要询问？为什么？
3. 多厂商 LLM 抽象中，OpenAI 和 Anthropic 的 messages 格式有什么关键差异？

### 进阶题
4. 设计一个命令注入防御方案：用户要执行 `npm install`，如何确保安全？
5. 符号链接逃逸攻击是什么？如何防御？
6. 文件 mtime 缓存机制如何防止覆盖用户未保存的修改？

### 架构题
7. 如果让你设计一个 Agent 的权限系统，规则引擎应该支持哪些匹配模式？如何保证规则合并的可预测性？
8. 多厂商 LLM 抽象中，如何处理流式输出的差异？设计一个统一的流式解析接口。
