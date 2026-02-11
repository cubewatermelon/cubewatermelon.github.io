## 谷歌A2A协议
在A2A（Agent-to-Agent）协议中，把 Agent 变成了网络上的“标准服务”，核心逻辑围绕着一份像名片一样的 Agent Card 展开，如下

每一个 A2A 兼容的 Agent 都必须拥有一份机器可读的 JSON 格式文档。这份“名片”里记录了它的底牌：
**身份信息**：名称、描述、版本。
**通信端点**（Endpoint）：别的 Agent 该去哪个 URL 找它发请求。
**技能清单（Skills）**：它能干什么（比如“查询光纤损耗”、“生成周报”）。
**能力（Capabilities）**：支持的功能，如是否支持流式传输（Streaming）、是否支持推送通知等。
**安全要求**：需要什么样的鉴权机制（API Key、OAuth 等）。

最简单的“广泛发现”模式下，A2A 推荐遵循 Web 标准，将这份卡片托管在一个固定的“知名路径”下：
https://your-agent-domain.com/.well-known/agent.json
任何 Agent 都能通过这个固定路径读到Agent Card

一个大致流程如下：
当一个 Client Agent（发起任务方）需要帮手时，它会经历以下几个发现阶段：

第一步：获取卡片（Fetching）
Client Agent 会访问目标域名的 /.well-known/agent.json 端点。如果是在企业内网或者特定的生态内，它可能已经预先配置了一系列已知的 Agent 列表。

第二步：能力匹配（Capability Matching）
拿到卡片后，Client Agent 会解析 JSON 中的 skills 字段：
比如Client Agent 手头有一个“光路径优化”的任务，它会遍历手头或目录中的 Agent Card，寻找标记有 skill: "optical-path-optimization" 的 Remote Agent。

第三步：注册中心查询（Registry-based Discovery）
对于大规模环境，还包含**精选注册表（Curated Registries）**机制。Agent 可以将自己的卡片注册到中心化的目录服务中。Client Agent 只需给注册表发一个查询请求（例如：“给我找个擅长处理故障告警且支持 SSE 流式输出的 Agent”），注册表就会返回符合要求的 Agent 列表。

在task中，采用JSON-RPC结构规范化任务指令。
MCP：是 Agent 用来接管工具和数据（如数据库、本地文件）的。
A2A：是 Agent 用来跟另一个 Agent 谈合作的。

## 与光网络的融合