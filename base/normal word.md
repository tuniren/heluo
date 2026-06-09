# normal word

## RAG

（Retrieval-Augmented Generation，检索增强生成）是一种将“大模型生成能力”和“外部知识检索能力”结合起来的架构。

“先查资料，再回答问题。”

## Prompt

Prompt（提示词）本质是：给模型的“输入约束 + 任务定义 + 上下文信息”的组合。

从工程角度看，它不是一句“让模型回答问题的话”，而是一段结构化控制信号。

```
[ROLE]
你是XXX专家

[GOAL]
完成XXX任务

[CONTEXT]
提供的数据/文档/日志

[CONSTRAINTS]
输出格式 / 限制

[OUTPUT FORMAT]
结构化定义（JSON / Markdown）
```

## MCP

Model Context Protocol，模型上下文协议）是由 Anthropic 提出的一个开放协议，用于让大模型能够标准化地连接外部工具、数据源和服务。

## LLM