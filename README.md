# jarvis-skills

J.A.R.V.I.S 自定义 **skill 市场** — 管理和存放新创建的 Agent 技能包。

与 `jarvis-harness-market`（桌面软件操控 harness）互补：本市场存放**知识型/流程型技能包**（SKILL.md + references 参考文档），不含可执行 Python 包。

## 目录结构

```
jarvis-skills/
├── registry.json              # 市场登记表
├── README.md
└── skills/
    └── ai-agent-interviewer/  # AI Agent 面试官
        ├── SKILL.md           # 角色设定 + 面试流程
        └── references/        # 七大知识模块参考文档
            ├── agent-runtime.md           # Agent 运行时、上下文管理、记忆系统
            ├── engineering-security.md    # 工程实现、多厂商抽象、安全权限
            ├── multimodal-ecosystem.md    # 语音/视觉/工具生态/多Agent协作
            ├── ai-frameworks.md           # LangChain/LangGraph/LangChain4j/SpringAI 对比
            ├── rag-knowledge-base.md      # RAG 全链路
            └── interview-questions.md     # 分级面试题库（100+ 题）
```

## 登记 registry

每个新 skill 需在 `registry.json` 的 `skills` 数组中登记：

```json
{
  "id": "skill-id",
  "display_name": "显示名",
  "description": "一句话功能描述",
  "requires": "依赖（无则留空）",
  "skill_md": "skills/<id>/SKILL.md",
  "install_cmd": "安装到 ~/.jarvis/skills/ 的命令"
}
```

## 新增 skill 流程

1. 在 `skills/` 下新建目录，命名与 SKILL.md frontmatter 的 `name` 一致
2. 编写 `SKILL.md`（frontmatter 必须含 `name` 和 `description`）
3. 参考资料（分级题库、领域知识等）放 `references/` 子目录，SKILL.md 中用相对路径引用
4. 在 `registry.json` 登记条目
5. 更新本 README 的目录结构示例

## 许可证

本项目采用 Apache License 2.0 许可证，详见 [LICENSE](./LICENSE) 文件。
