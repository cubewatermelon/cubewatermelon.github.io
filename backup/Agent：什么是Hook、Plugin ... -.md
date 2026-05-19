* **Agent**：以大模型为核心，能够理解任务、调用工具、持续执行并反馈结果的智能执行体。

* **Tool**：Agent 可直接调用的具体函数、API 或外部系统能力。

* **Skill**：指导 Agent 完成某类任务的方法说明、经验规则或操作 SOP。

* **Plugin**：将多个 Tool、Skill、Hook 和配置打包在一起的能力扩展模块。

* **Hook**：插入 Agent 执行流程中的自定义逻辑点，用于校验、拦截、增强或记录。

* **Planner**：负责将复杂任务拆解为多个可执行步骤的规划模块。

* **Tool Router**：负责判断当前步骤应调用哪个工具的决策模块。

* **Tool Executor**：负责真正执行工具/API 调用并返回结果的执行模块。

* **Memory**：保存长期知识、历史经验和用户偏好的记忆模块。

* **State**：记录当前任务进度、中间结果和上下文状态的数据。

* **ContextBuilder**：将系统规则、历史、技能、工具和用户输入组织成模型上下文的模块。

* **Observation**：工具执行后的返回结果，供 Agent 继续推理和决策。

* **Reflection**：Agent 对当前结果进行自检、纠错和补充推理的过程。

* **Guardrails**：限制 Agent 行为边界、防止高风险或越权操作的安全机制。

* **Workflow**：预定义的任务执行流程，适合稳定、可控的场景。

* **MessageBus**：负责用户、Agent 和外部系统之间消息传递的通信通道。

* **Function Calling**：让大模型以结构化参数形式调用工具/API 的机制。

* **ReAct**：一种“推理（Reasoning）+ 行动（Action）”交替循环的 Agent 执行模式。

* **Scratchpad**：Agent 在推理过程中保存中间思考和临时结果的工作区。

* **Session**：一次连续任务或对话的上下文集合，用于维持连续性。

* **MCP（Model Context Protocol）**：一种标准化协议，用于让 Agent 统一接入外部工具、数据源和服务。
