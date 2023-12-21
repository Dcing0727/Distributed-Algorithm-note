我们研究分布式系统中使用的算法与集中式系统的顺序算法相反

什么是分布式系统？

一组进程/节点寻求实现一些共同的目标通过相互沟通

进程/节点：核心、处理器、物理/虚拟机、集群



在流程协同工作中，虽然各个环节都在为通用共同目标而努力，但流程本身可能无法提供对整个系统状态的完整把握，需要额外的手段来计算全局系统状态。



进程没有系统的全局视图

故障 异步

特别是故障+异步

经常影响可解决性！

并发

这些是我们的主要“敌人”



如果进程崩溃

-忽略发送/接收消息（例如，缓冲区已满）

-进程是恶意的,恶意/任意/拜占庭式失败

我们如何处理通信异步？

-网络故障（例如交换机故障）可能导致消息丢失或延迟

-不同的处理速度->异步



检测过程故障（借助时间假设）是不可靠的！



任何规范都可以用

-安全

-活力



安全

-声明不应发生任何坏事的属性

活力

-表示应该发生好事的属性



例如：

安全：不说谎 活泼：说话

安全与健康相结合：（总是）说实话

有用的分布式服务应该同时提供生存性和安全性



给出解决方案根据规范证明正确性

“正确性可能是理论上的，但错误会产生实际影响”

分析复杂性：时间复杂性（影响延迟）+消息复杂性（影响吞吐量）

编码：我们将使用伪代码，没有特定的编程语言



定理。如果底层通信信道容易出现任意消息丢失，则没有算法可以解决两个进程之间的共同决策问题。



A1和A2执行消息交换的连续阶段

矛盾证明

假设算法A使用最少的通信相位

当A终止时（即{yes，yes}），存在发送消息的最后阶段，例如由A1向A2发送的消息m

A1执行的最后一个语句不能取决于A2是否接收到m（因为m可能丢失）

类似地，A2执行的最后一个语句不能依赖于m

m是无用的，可以移除

存在通信相位较少的A