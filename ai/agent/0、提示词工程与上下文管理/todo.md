1. 动态模板引擎 (Dynamic Template Engine)
   大厂里的 Prompt 不是写死的字符串，而是高度动态的模板，这就好比我们后端渲染用的 Thymeleaf 或 FreeMarker，只是现在渲染出来的是发给 LLM 的指令。

变量注入 (Variable Injection): 运行时动态将 {{user_input}}、{{retrieved_context}} (RAG 捞出来的数据)、{{current_time}}、{{user_profile}} 注入模板。

工具与签名渲染: 将后端定义好的 Java 接口通过反射转换成 JSON Schema，再动态拼接进 Prompt 的 <tools> 标签里，告诉大模型“你现在有哪些武器”。

历史记录滑动 (History Sliding): 动态决定在 Prompt 里塞入多少轮的对话历史，防止 Token 超出上限。

2. 结构化边界约束 (Structured Boundary Constraint)
   让大模型天马行空地聊天很容易，但让它乖乖按规矩输出却极难。Agent 要调用后端接口，输出格式错一个括号都会导致反序列化失败。

格式化输出指令 (Formatting Instructions): 使用 XML 标签（如 <thought>, <action>, <json_output>）来严格规范大模型的输出结构。XML 标签在工业界极其好用，因为它能明确划定数据的边界。

负向约束 (Negative Prompting): “不要做什么”比“要做什么”更难教。需要通过清晰的惩罚性描述（如“严禁输出任何 Markdown 格式以外的内容，否则系统将崩溃”）来压制模型的发散。

Few-Shot 示例构造: 在 Prompt 中塞入几个完美的 Input -> Output 样例（Shot）。这比写一万字规则都管用，这是约束大模型输出格式的最强杀器。

3. 认知范式注入 (Cognitive Paradigm Injection)
   除了之前的 CoT，提示词本身可以重塑模型的思考路径。

角色与系统设定 (System Persona): 赋予极其具体、带有专业知识背景的人设。例如：“你是一个具有 10 年经验的高级 Java 架构师，精通 JVM 调优与并发编程，你的回答必须直接切中技术底层，拒绝车轱辘话。”

元提示词 (Meta-Prompting): 用一段提示词，去指导大模型“如何写出更好的提示词”，或者让大模型在回答前，先输出它自己认为应该遵循的规则。

4. PromptOps：提示词工程化运维
   这是大厂基建的核心。Prompt 是一种玄学，改一个词可能导致准确率掉 10%，必须用后端的思维去管理它。

版本控制 (Versioning): 将 Prompt 像代码一样存入 Git 或专用的 Prompt Hub，任何修改都要经过 Review。

A/B 测试与灰度发布: 上线新版本的 Prompt 时，先切 10% 的真实流量给新 Prompt，对比业务转化率、API 调用成功率和耗时，跑赢了再全量发布。

Prompt 路由 (Prompt Routing): 针对不同复杂度的任务，挂载不同的 Prompt 模板。简单的闲聊用短模板，复杂的代码分析用携带长下文的重型模板。