Agent整体架构包含如下几个关键部分

1. 会话管理，如何处理（多个）用户的消息
2. AgentLoop：如何拉取消息、按照工作流执行、如何拼接上下文、如何产生记忆

## 以NanoBot为例快速理解

### NanoBot整体框架

```
Chat / WebUI / API
      ↓
MessageBus 消息总线
      ↓
AgentLoop 主循环
      ↓
ContextBuilder 组装提示词
      ↓
LLM Provider 调模型
      ↓
AgentRunner 执行 ReAct 循环
      ↓
ToolRegistry / MCP / Skills 调工具
      ↓
Session / Memory 保存状态
      ↓
回复用户
```



## 1. 入口层：MessageBus

nanobot 不是直接让用户消息进 LLM，而是先经过 `MessageBus`。它把不同渠道的消息统一成 `InboundMessage`，再把 Agent 回复统一成 `OutboundMessage`。这样 Telegram、WebUI、CLI、Feishu 等都可以复用同一个 Agent 核心。源码里 `MessageBus` 明确是“解耦 chat channels 和 agent core”的异步队列。

## 2. 核心层：AgentLoop

`AgentLoop` 是总控。它负责：

```
接收消息 → 恢复会话 → 构建上下文 → 调用 Runner → 保存结果 → 返回消息
```

源码注释直接写了它的职责：接收 bus 消息、构建包含 history / memory / skills 的上下文、调用 LLM、执行工具调用、发送响应。

它还有一个状态机：

```
RESTORE → COMPACT → COMMAND → BUILD → RUN → SAVE → RESPOND → DONE
```

这说明 nanobot 不是“问一句答一句”，而是把一次用户请求拆成可控流程：先恢复状态，再压缩上下文，再判断命令，再构建 prompt，再运行 Agent。

## 3. 上下文层：ContextBuilder

`ContextBuilder` 负责把各种信息塞进 LLM 输入：

```
系统身份
+ AGENTS.md / SOUL.md / USER.md / TOOLS.md
+ 长期记忆 MEMORY.md
+ Active Skills
+ Recent History
+ 当前用户输入
+ 当前时间、渠道、chat_id 等运行时信息
```

源码中 `build_system_prompt()` 会加载 identity、bootstrap files、memory、skills、recent history 和 session summary；`build_messages()` 再把历史对话和当前用户消息组合成 LLM 消息。

这对应通用 Agent 架构里的：**Prompt 构造器 + 记忆注入 + 技能注入**。

## 4. 执行层：AgentRunner

`AgentRunner` 是真正的 ReAct 循环：

```
for iteration in max_iterations:
    调用 LLM
    如果 LLM 要调用工具：
        执行工具
        把工具结果写回 messages
        继续下一轮 LLM
    如果没有工具：
        输出最终答案
```

源码里 `AgentRunner.run()` 会循环调用模型，判断 `response.should_execute_tools`，然后执行 `_execute_tools()`，再把 tool result 作为 `role="tool"` 的消息追加回上下文。

这就是 Agent “智能执行工具”的核心机制：**模型不是直接给最终答案，而是可以先生成 tool call；框架执行工具后，把结果喂回模型，让模型继续判断下一步。**

## 5. 工具层：ToolRegistry + ToolLoader

nanobot 的工具不是写死在主循环里，而是统一注册到 `ToolRegistry`。

`ToolRegistry` 负责：

```
注册工具 register
获取工具定义 get_definitions
校验参数 prepare_call
执行工具 execute
```

源码中 `get_definitions()` 会把工具 schema 提供给模型，`prepare_call()` 会检查工具是否存在、参数是否合法，`execute()` 才真正调用工具。

`ToolLoader` 负责自动发现工具模块，并支持外部插件通过 `entry_points(group="nanobot.tools")` 注册。

这对应工程落地里的：**工具注册中心 + 工具 schema + 参数校验 + 插件化扩展**。

## 6. 记忆层：Session + Memory

nanobot 有两类状态：

**短期会话 Session**：保存当前聊天历史，按 `channel:chat_id` 区分会话。`SessionManager` 把会话存成 JSONL 文件，并支持加载、保存、截断历史。

**长期记忆 Memory**：保存 `MEMORY.md`、`history.jsonl`、`SOUL.md`、`USER.md`。源码里 `MemoryStore` 明确是文件型记忆系统，`Consolidator` 会把过长历史总结归档，避免上下文爆掉。

这就是 Agent 能做长程任务的基础：**不是每次都从零开始，而是能带着历史、摘要和长期记忆继续工作。**

## 7. Skills 层：能力说明，不一定是工具

nanobot 的 `skills` 目录里每个 skill 是一个目录，里面有 `SKILL.md`，包含 YAML 元数据和 Markdown 指令。源码 README 说明内置技能包括 GitHub、weather、summarize、tmux、clawhub、skill-creator、long-goal 等。

所以 skill 更像是：

```
“当你处理这类任务时，应该遵循的操作手册 / SOP / 能力说明”
```

工具是“可执行 API”，skill 是“怎么用这些能力完成任务的说明”。

## 8. MCP 层：外部工具接入

nanobot 支持 MCP。README 里明确说它支持 chat channels、memory、MCP 和部署路径；v0.1.4 release notes 也提到 MCP tool server support。

这意味着 nanobot 可以把外部系统接成工具，例如：

```
本地脚本
数据库
RAG 服务
网管接口
MML 命令服务
故障诊断服务
```

## 一句话总结

nanobot 的 Agent 架构可以总结为：

**MessageBus 负责接入，AgentLoop 负责任务生命周期，ContextBuilder 负责给模型组织上下文，AgentRunner 负责“LLM ↔ Tool”的循环执行，ToolRegistry/MCP 负责扩展外部能力，Session/Memory 负责状态和长期记忆，Skills 负责沉淀任务方法。**



下图比较全面的展示了当一条消息输入到Agent中时，发生了哪些步骤
<img width="1672" height="941" alt="Image" src="https://github.com/user-attachments/assets/9bcbcb96-4e6d-4d16-8241-128b9656f151" />