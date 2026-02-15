---
title: AI Agent 通信进化史：从 Prompt 到 MCP
date: 2026-02-16 18:00:00
tags:
  - AI Agent
  - MCP
  - Function Calling
  - Prompt Engineering
  - 历史演进
categories:
  - AI 技术史
  - 工程架构
---

> 本文是[《深度解析 Model Context Protocol (MCP)》](../深度解析-Model-Context-Protocol-MCP-AI-时代的通用接口标准)的姊妹篇——从「为什么」视角回顾 Agent、Prompt、Function Calling 到 MCP 的必然之路。

## TL;DR
- **历史主线**：Prompt → AutoGPT → Function Calling → MCP，本质是「让模型可靠地调用外部世界」的层层封装。  
- **格式混战**：SystemPrompt vs FunctionCalling 并存，各家 JSON Schema 不兼容 → 需要统一协议。  
- **MCP 定位**：Agent 与 Tool 之间的「USB-C 插座」，不关心模型是谁，只负责把工具标准化插上去。  

<!-- more -->

## 1. 史前时代：Prompt 里「夹带私货」
- **做法**：把函数签名、示例输出一股脑写进 user prompt，让模型“悟”出该调啥。  
- **痛点**：
  - 长度爆炸，上下文占用高；  
  - 概率生成，格式经常跑偏；  
  - 换模型 = 重写 prompt，维护地狱。  

## 2. 第一次觉醒：AutoGPT 与 AI Agent 概念
- **核心思想**：模型不再只回答，而是成为**中间调度员**——  
  1. 接收用户目标；  
  2. 自主拆解任务链；  
  3. 调用外部工具（搜索、代码执行、文件读写）；  
  4. 把结果汇总再返回。  
- **AutoGPT 流程**（下图灰色部分即「AgentTool」雏形）：  
  - 用户 → Agent → 工具 → Agent → 用户  
- **遗留问题**：
  - 工具描述仍然自由文本，格式不统一；  
  - 模型幻觉 → 调错参数、乱调接口；  
  - 无类型校验，运行时爆炸。  

## 3. 规范之战：Function Calling 登场
- **解法**：把「工具说明书」结构化——JSON Schema 描述函数名、参数类型、必填项。  
- **各家实现**（2023 起）：  
  | 厂商 | 入口字段 | 协议细节 |
  |----|---------|---------|
  | OpenAI | `functions[]` / `tool_choice` | 1.x 版 Schema |
  | Google | `tools.function_declarations` | 差异字段命名 |
  | Anthropic | `tools` | 支持 `<tool_call>` XML 标记 |
  - **共同点**：都走**SystemPrompt 注入** → 模型生成符合 Schema 的调用 → 客户端代理执行。  
- **并存现实**：  
  - 老模型/开源模型不支持 FC，只能继续拼 SystemPrompt；  
  - 同一套工具要维护两份描述 → 急需“与模型无关”的抽象层。  

## 4. 通用插座：MCP 接管工具侧
- **设计哲学**：  
  - 把「工具」做成标准服务，**与模型解耦**；  
  - 任何 Agent（不论用 GPT、Claude 还是 Llama）只要会讲 MCP，就能即插即用；  
  - 统一资源、提示词、函数三合一抽象，详见[姊妹篇](../深度解析-Model-Context-Protocol-MCP-AI-时代的通用接口标准)。  

- **角色划分**：  
  - **MCP Server**：暴露工具/资源/提示词；  
  - **MCP Client**：随 Agent 启动，负责协议握手、调用、结果回传；  
  - **Agent**：专注决策与编排，不再关心工具格式。  

## 5. 端到端通信全景（Mermaid 图）
```mermaid
sequenceDiagram
    participant U as 用户
    participant A as AI Agent<br/>(MCP Client)
    participant M as AI 模型
    participant S as MCP Server<br/>(Tool 服务)

    U->>A: 提问：「帮我总结最新 AI 论文」
    A->>A: 把问题封装为 userPrompt
    A->>S: MCP 协议：列出可用工具
    S-->>A: 返回工具列表+JSON Schema
    A->>A: 将工具描述注入 SystemPrompt 或 FC
    A->>M: userPrompt + SystemPrompt/FC
    M->>M: 理解任务，生成工具调用
    M-->>A: 调用 search_arxiv(keyword="AI 2026")
    A->>S: 实际执行工具调用
    S-->>A: 返回论文列表
    A->>M: 把结果回传给模型
    M->>M: 生成自然语言总结
    M-->>A: 最终答案
    A-->>U: 返回精炼总结
```

## 6. 小结与展望
- **Prompt 时代**靠「说」；**Function Calling**靠「规范说」；**MCP**靠「把工具做成标准件」。  
- 对开发者：写一次 Tool → 任何 Agent 可用；对模型方：专注推理，无需兼容 N 种格式。  
- 下一步：多 Agent 之间用 MCP 互相发现、交易、组合，形成「工具即服务」市场——模型只负责决策，工具生态交给协议。  

---
> 想亲手跑通一个 MCP Server？移步[《深度解析 Model Context Protocol (MCP)：AI 时代的通用接口标准》](../深度解析-Model-Context-Protocol-MCP-AI-时代的通用接口标准)，内含最小可运行示例与最佳实践清单。 