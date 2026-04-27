# function calling 深入

用户提问了一句话  帮我查一下浦东新区昨天下午的高并发派单记录。 

请求体PayLoad详解。

## 第一次交互

Springboot应用收到前端的请求 则后端就知道需要去调用大模型API。不仅要传用户的问题给llm 还需要传tools参数，也就是把需要用到的接口（查询派单接口）的说明书 JSON Schema一起发过去 让llm会用这个接口。

### java发给LLM的 Request Payload 请求报文

```json
{
  "model": "gpt-4o",
  "messages": [
    {
      "role": "user",
      "content": "帮我查一下浦东新区昨天下午的高并发派单记录"
    }
  ],
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "query_dispatch_records",
        "description": "查询指定区域和时间段的环卫车辆派单记录",
        "parameters": {
          "type": "object",
          "properties": {
            "region": { "type": "string", "description": "行政区，如：浦东新区" },
            "time_range": { "type": "string", "description": "时间范围，如：昨天下午" }
          },
          "required": ["region", "time_range"]
        }
      }
    }
  ]
}
```

简单概括就是  调用模型名  消息（包括用户和问题） 还有工具（可能不止一个）的 tools schema主要包括函数名 描述 参数。 llm主要靠描述去理解这个工具如何使用。 以及llm此时是接受所有工具 因为他需要从这些工具中选择出要用什么

### llm第一次响应 Response Payload

llm收到java的请求 知道了用户要查数据。就判断需要用传来的 tools里的哪个tools。然后返回java一个响应报文

```json
{
  "choices": [
    {
      "message": {
        "role": "assistant",
        "content": null, // 注意！这里是空的，它没说话！
        "tool_calls": [
          {
            "id": "call_abc123", // 极其关键！这是这次调用的全局唯一流水号
            "type": "function",
            "function": {
              "name": "query_dispatch_records",
              "arguments": "{\"region\":\"浦东新区\",\"time_range\":\"昨天下午\"}" // 完美的 JSON 字符串
            }
          }
        ]
      },
      "finish_reason": "tool_calls" // 状态码：我还没干完，卡在调工具这步了，等你把数据拿给我
    }
  ]
}
```

主要包括 一个tool_calls 代表函数调用，里面有这次调用的唯一id号 还有调用函数的名字和参数信息 并且返回一个状态码finish_reason 为tool_calls 代表 llm不能直接调tools 还得等待java后端调用返回结果给llm 此时content是空的 llm没有输出文本

## 中间过程 

java收到llm的信息 发现状态码是tool_calls 然后就去解析传过来需要调用的tools schema 的json字符串 直接反序列化为一个java DTO对象 作为一个参数 传入接口 得到接口输出数据

## 第二次交互

java输出数据不能直接给用户 还需要经过llm润色

### java再次发给llm Request PayLoad

```json
{
  "model": "gpt-4o",
  "messages": [
    // 1. 保留用户的原话
    { "role": "user", "content": "帮我查一下浦东新区昨天下午的高并发派单记录" },
    
    // 2. 原样塞回大模型刚才下达的指令（含流水号）
    { "role": "assistant", "content": null, "tool_calls": [...] }, 
    
    // 3. 核心！新增一条角色为 tool 的消息，把咱们 Java 查到的结果塞进去
    {
      "role": "tool",
      "tool_call_id": "call_abc123", // 拿着流水号复命，大模型才知道这串数据是给哪个任务的
      "content": "{\"total\": 1, \"records\": [{\"plate\": \"沪A123\", \"status\": \"已完成\"}]}"
    }
  ]
}
```



主要是新增了一条 角色为tool的消息 代表java接口的返回数据 里面对应了函数调用的唯一id 代表是这次函数调用返回的结果



### LLM响应java的信息 Reponse payload

```json
{
  "choices": [
    {
      "message": {
        "role": "assistant",
        "content": "帮您查到了！浦东新区昨天下午的高并发派单记录中，共有一条记录，车牌号为沪A123，目前状态为已完成。"
      },
      "finish_reason": "stop" // 彻底搞定
    }
  ]
}
```

就是把java接口传来的数据润色一下 返回springboot.



## 详细思考

如果不需要函数调用 其实llm就不需要返回需要的tools 就第一次交互直接结束了  直接返回结果。 也就是函数调用多了一次java自身调用接口查询数据 以及第二次交互 给llm数据 llm润色返回。



## 优点

存文本react 大模型是在写很长的文字 后端需要用正则表达式 提取 如果生成错误 就提取不出来 报错了。

在函数调用中 大模型是在填表 输出的格式 是llm在训练的时候就学会的 强制约束的纯净json java后端可以直接反序列化得到 非常稳定。 ----》也就是把自然语言解析 降维成 结构化 RPC通信

并且 纯文本时代  所有东西都在prompt中 容易引发幻觉。 函数调用在http协议层把messages 消息和 tools 能力定义分开 大模型处理起来更清晰。



## 并行调用

如果 需要调用一个工具两次 可以开启两个线程 调用同一个接口 分别 传入两个对象 。然后返回的时候 塞两条信息给llm就行 每个tool对应一个id 也就是有两次函数调用唯一ID。

把java高并发处理和llm推理能力结合。