模块五：工程落地与防御架构 (Engineering & RAGOps)
这是区分“Demo 玩家”和“大厂正规军”的分水岭。

1. 可观测性与链路追踪 (Observability):

由于 Agent 的调用栈非常深（LLM -> Tool -> LLM），如何在系统中埋点，追踪每一次调用的耗时、Token 消耗和具体的 Prompt 变更。

2. 异步执行与回调机制：

Agent 思考和执行可能长达数分钟，后端接口决不能同步阻塞。如何设计 WebSocket 或 Webhook 机制向前端推送思考过程流（Streaming）。

3. 权限与熔断机制：

Human-in-the-loop (人类在环): 当 Agent 准备执行高危操作（如转账、删库）时，如何暂停流转，等待人类确认。

Rate Limiting: 防止 Agent 陷入死循环（无限调用 API 失败），设置最大思考步数（Max Iterations）和熔断策略。