# ReAct Prompt

1.  system prompt

   是工具的描述 大模型靠阅读描述 来决定是否调用工具。 描述不清晰 llm就会乱调tools

2. Format instruction

​	格式强指令 是react的发动机  规定大模型输出格式

3. User Input

​	用户传入的query

4. Agent Scratchpad 

​	agent草稿本。  实现agent有记忆和循环的关键变量。 每一次调用tools 返回的ans 都会append到草稿本 然后重新把巨大的prompt给大模型。



```Plaintext
尽可能回答以下问题。你可以使用以下工具：
{tools}  <-- 这里会被替换成比如：[Search(用于搜索网页), Calculator(用于计算器)]

请严格按照以下格式输出：

Question: 你必须回答的输入问题
Thought: 你应该总是思考你要做什么
Action: 需要采取的行动，必须是 [{tool_names}] 中的一个
Action Input: 传入动作的参数（通常是 JSON 格式）
Observation: 动作返回的结果
... (这个 Thought/Action/Action Input/Observation 的循环可以重复 N 次)
Thought: 我现在知道最终答案了
Final Answer: 针对原始问题的最终答案

开始！

Question: {input} <-- 这里替换为用户问题，例如："2023年上海环卫项目占比最高的是哪个区？"
Thought:{agent_scratchpad} <-- 这里是留给后端动态追加历史记录的地方
```

一个模版 。

## React 纯文本解析

正则匹配Action: Action Input 有很大缺点：

纯文本的ReAct极其依赖 大模型的 指令遵循能力。

1. json格式化失败 大模型在action input里输出格式错误 就会导致工具调用直接报错
2. 死循环 llm连续输出不存在的tools 后端解析失败 返回错误作为observation传回去 下一次循环 llm无法理解错误 继续输出error 导致token消耗。



### Function Calling

使用函数调用： 调llmAPI的时候。不再把工具描述塞进prompt 。通过api的**tools参数** 用**json schema**的形式传给llm

llm在底层直接返回 结构化的 函数名 和参数json

降低了java后端解析的压力。

从text-based react --> tool-calling agent





### 分析区别

1. API传参通道隔离
   - Text-based 工具描述 格式要求 用户问题 全部揉成一团 作为一个巨大的String 塞进cotext 传入大模型
   - Function Calling : api请求体实现了通道分离
     - messages 放聊天记录
     - tools数组 定义工具库 底层是json schema
     - 大模型在底层架构层面区分 人话 和 函数 
2. 返回结果
   - Text-based 返回的都是一串String 后端必须做文本分析一样 用Regex抠出Action:后的字 
   - Function Calling : 返回结构化的Object   返回一个专门的tool-calss数组 后端可以用反序列化解析
3. 模型的基座能力差异 
   - Text-based 依赖模型的指令遵循能力
   - function calling 现在的基座模型 在sft【监督微调 和预训练阶段 就针对tools 这种json schema格式输入输出进行 海量的针对微调  。 
   - 训练出了一个本能 看到tools参数传入 且用户的意图匹配 就会切换为结构化输出模式 而不是聊天模式
4. 总结：

​	text-based react 进化到 function calling agent 本质上 把工具调用的格式话要求 从prompt工程层 下沉到llm底层的api协议 和模型微调成 解放了后端的解析压力。

