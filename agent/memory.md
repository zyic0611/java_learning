# memory

大模型 是一个无状态的函数 y=f(x) 记忆就是把y当成x重新输入



## stm

短期记忆是llm的上下文窗口 context window。 一般存在redis或者应用内存里 java的list<message>

### 问题：

1. token限制和成本 上下文窗口有限 聊天记录越长 单次请求越贵 延迟latency越高
2. 大海捞针效应。短期记忆太长 大模型会走神 忽略中间关键信息 只关注开头结尾

### 解决：

1. Sliding window 滑动窗口。只保留最近N轮对话
2. token buffer token截断  用token数量计算
3. summary memory 摘要记忆。对话长到限定后 后台触发一个异步任务 把前面的对话总结成一个summary 把summary放在下一次请求开头 替换掉原始对话。



## ltm

本质上是 rag 检索增强生成系统的一个子集。 依托于向量数据库 。

### 运作机制

1. 写入 ： 用户说完一句话后 或者agent得出一个重要的结论 掉哟个embedding模型 转成vector 存入vector db 并且打上time 用户id等元素句 metadat。
2. 读取 用户提问的时候 不着急调用大模型 把question也 vector化 去db里做 knn or 余弦相似度检索 找出最相关的几条历史记忆



## 架构搭配

springboot中 搭建支持百万用户的agent记忆系统。

1. 接口层 用intercptor or aop 在每次post青睐 提取sessionId or userId
2. stm层 使用redis的list or zset 存储最近20轮对话 设置ttl自动过期 避免oom
3. ltm 使用异步消息队列 每次对话结束 发消息给向量化 异步的把对话嵌入并写入向量数据库 不阻塞主业务
4. 组装层  在调用llm客户端前 先打开两个线程。一个从主redis里查近期对话 一个并发去向量数据库查相关的历史记忆。 汇总后使用特定的prompt template 拼接