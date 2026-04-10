# Multi Agent

## 3 topologies

1. pipeline

架构逻辑：线性结构 a做好给b b做好给c

业务映射 ：适合有明确sop【标准作业程序的场景  

优点：高度可控 稳定性很强 easy debug

缺点：灵活性低 某个环节出错 全链路崩溃

2. 主管/路由模式 supervisor

架构逻辑：用户把任务发给supervisor agent 不干活 就拆解任务 把子任务给子worker agent ,做完后返回给supervisor 来汇总or决定是否redo

优点： 能处理复杂的模糊任务 扩展性强 加一个性能力 加一个worker

缺点：supervisor的能力决定系统上线 他的路由错了 就都错了

3. 辩论/评审模式  actor-critic

架构逻辑：两个agent相互对抗or协作。生成器generator 批评者reviewer 。生成的东西给reviewer去测试 如果错了 就拿错误日志给generator重新生成 直到reviewer满意。

优点：降低大模型幻觉最有效

缺点：token消耗巨大 没有优秀的停止条件 容易死循环



## 状态管理与状态机

Q:多智能体如何通信 如何知道进行到哪一步？

A: 构建multiagent 不能依靠简单的srting拼接 。 在全局维护一个state状态对象 比如一个包含message sender current_tast_status的java object。

每个agent本质上 都是 graph的一个node  agent执行完毕后 要返回一个更新的state agent之间通过条件边相连。   一个节点运行后 根据状态里的某个标识变量 决定下一步是到a节点 还是回到c节点。

图状态机的设计 让multi agent 有了容错 暂停 甚至人工介入的能力。



## 提升

Q： 多个agent之间是共享一个 context or 各自独立

A： 按需分配。 pipeline 通常传递summary节约token 不是传递全亮对话。  supervisor 模式 主管拥有全局context 具体woker只拥有需要的局部信息

Q：怎么房子multi agent进入 死循环

A： 工程上要有兜底机制

1. 硬性截断：状态机维护最大循环次数 超过5n次抛异常or 转人工
2. 强制仲裁者 两个agent僵持不下 引入第三方配置更高算力模型 的裁判agent进行最终裁决

