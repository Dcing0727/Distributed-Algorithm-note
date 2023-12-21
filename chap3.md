# 可靠的广播原语

## BestEffortBroadcast, instance beb.

### 属性

1. **请求（Request）**：`< beb, Broadcast | m >`
   - 这个请求指示“beb”实例将消息“m”广播给所有进程。
   - 广播意味着消息将被发送给系统中的每一个进程。

2. **指示（Indication）**：`< beb, Deliver | p, m >`
   - 这个指示表示消息“m”被广播者“p”发送，并且已经被接收。
   - 当一个进程接收到一条广播消息时，它将触发这个指示。

3. **属性（Properties）**
   - **BEB1. 有效性（Validity）**：如果一个正确的进程广播了消息“m”，那么每个正确的进程最终都会交付“m”。这意味着消息的传递是可靠的，至少对于正确的（即没有发生故障的）进程来说是如此。
   - **BEB2. 无重复（No duplication）**：任何消息都不会被一个进程交付多于一次。这避免了重复处理相同的消息。
   - **BEB3. 无创造（No creation）**：如果一个进程交付了由发送者“s”广播的消息“m”，那么“m”之前必须是由“s”广播的。这保证了广播的消息是由声明的发送者实际发送的，没有被错误创造或篡改。

### 实现

```c
Implements: BestEffortBroadcast, instance beb.
Uses: PerfectPointToPointLinks, instance pl.

upon event < beb, Broadcast | m > do
	for all q ∈ Π do
		trigger < pl, Send | q, m >;

upon event < pl, Deliver | p, m > do
	trigger < beb, Deliver | p, m >;
```

## ReliableBroadcast, instance rb

### 属性

“ReliableBroadcast”（可靠广播，简称rb），扩展了“BestEffortBroadcast”（尽力而为广播）的基本特性，增加了更强的一致性保证。

1. **请求（Request）**：`< rb, Broadcast | m >`
   - 这个请求指示“rb”实例将消息“m”广播给所有进程。

2. **指示（Indication）**：`< rb, Deliver | p, m >`
   - 这个指示表示消息“m”被广播者“p”发送，并且已经被接收。

3. **属性（Properties）**
   - **RB1–RB3**：与“BestEffortBroadcast”（beb）的属性BEB1–BEB3相同。
   - **RB4. 协议（Agreement）**：如果某个正确的进程交付了消息“m”，那么“m”最终会被所有正确的进程交付。这意味着一旦一个正确的进程确认收到消息，其他所有正确的进程也将最终收到该消息。

### 实现

Implements: ReliableBroadcast, instance rb.

Uses: BestEffortBroadcast, instance beb; PerfectFailureDetector, instance P.

```c
# 当可靠广播初始化时触发的事件
upon event < rb, Init > do
    correct := Π;  # 初始化correct集合，包含所有进程
    delivered := ∅;  # 初始化delivered集合，用于跟踪已交付的消息
    for all p in Π do 
        from[p] := ∅;  # 初始化from字典，记录每个进程p接收到的消息

# 当有广播消息请求时触发的事件
upon event < rb, Broadcast | m > do # 要广播消息m
    trigger < beb, Broadcast | [DATA, self, m] >;  # 通过尽力而为广播发送消息

# 当尽力而为广播传递消息时触发的事件
upon event < beb, Deliver | p, [DATA, s, m] > do # 当一条进程都收到了p广播的来自s的消息m
    if m not in delivered then
        delivered := delivered ∪ {m};  # 将消息添加到delivered集合，避免重复交付
        trigger < rb, Deliver | s, m >;  # 触发可靠广播的交付事件
        from[p] := from[p] ∪ {s, m};  # 记录进程p接收到的来自s的消息m
        if p not in correct then
            trigger < beb, Broadcast | [DATA, s, m] >;  # 如果进程p不在correct集合中，系统将再次通过尽力而为广播发送消息m。这是因为p的状态不确定可能影响了消息m的可靠传递。

# 当完美故障检测器检测到进程崩溃时触发的事件
upon event < P, Crash | p > do
    correct := correct \ {p};  # 从correct集合中移除崩溃的进程
    for all {s, m} in from[p] do
        trigger < beb, Broadcast | [DATA, s, m] >;  # 重新广播该崩溃进程接收过的所有消息

```

工作原理是：通过尽力而为广播（beb）传递消息，并使用完美故障检测器（P）来跟踪进程的崩溃。一旦某个消息被一个进程接收，它就被记录下来，并确保所有正确的（没有崩溃的）进程最终都能交付这条消息。如果检测到进程崩溃，系统将重新广播该进程接收到的所有消息，以确保所有消息都能被正确的进程接收和交付。这种机制增加了消息交付的可靠性，确保了即使在面临进程崩溃的情况下，所有正确的进程也能接收到广播的消息。

###  表现

1. **最佳情况**
   - **条件**：初始发送者没有崩溃。
   - **性能**：
     - **通信步骤**：单个通信步骤。
     - **消息数量**：O(N)条消息，其中N是系统中进程的数量。
   - **解释**：在这种情况下，由于初始发送者没有崩溃，其发送的消息会被所有其他进程接收并处理。每个进程只需要接收并交付一次消息，因此总消息数量与进程数成线性关系。

2. **最坏情况**
   - **条件**：进程按顺序依次崩溃。
   
   - **性能**：
     - **通信步骤**：O(N)步，每个进程崩溃时都可能需要一次额外的通信步骤。
     - **消息数量**：O(N²)条消息。
     
   - **解释**：在最坏情况下，每个进程在崩溃前可能已经接收到了消息并需要对其进行广播。由于进程依次崩溃，每个后续的进程都需要重新广播这些消息，导致总消息数量呈二次方增长。
   

### 另一个版本

```c
# 当可靠广播系统初始化时触发的事件
upon event < rb, Init > do
    delivered := ∅;  # 初始化delivered集合，用于跟踪已交付的消息

# 当有广播消息请求时触发的事件
upon event < rb, Broadcast | m > do
    trigger < beb, Broadcast | [DATA, self, m] >;  # 通过尽力而为广播发送包含消息m的广播请求

# 当尽力而为广播传递消息时触发的事件
upon event < beb, Deliver | p, [DATA, s, m] > do
    if m not in delivered then  # 如果消息m尚未被交付
        delivered := delivered ∪ {m};  # 将消息m标记为已交付
        trigger < rb, Deliver | s, m >;  # 触发可靠广播的消息交付事件
        trigger < beb, Broadcast | [DATA, s, m] >;  # 重新通过尽力而为广播发送消息m
```

  表现：

   最佳情况：单个通信步骤和O（N2）消息

   最坏情况：进程按顺序崩溃，O（N）步和O（N2）消息

## UniformReliableBroadcast, instance urb.

### 属性

“UniformReliableBroadcast”（统一可靠广播，简称urb），扩展了常规的“ReliableBroadcast”（可靠广播）机制，增加了更强的一致性保证。

2. **请求（Request）**：`< urb, Broadcast | m >`
   - 这个请求指示“urb”实例将消息“m”广播给所有进程。

3. **指示（Indication）**：`< urb, Deliver | p, m >`
   - 这个指示表示消息“m”被广播者“p”发送，并且已经被接收。

4. **属性（Properties）**
   - **URB1–URB3**：与常规可靠广播（RB1–RB3）的属性相同。
     - **URB4: 统一协议（Uniform agreement）**：如果某个进程（无论是正确的还是有故障的）交付了消息“m”，那么“m”最终会被所有正确的进程交付。这意味着一旦任何进程（无论状态如何）确认收到消息，所有其他正确的进程也将最终收到该消息。

确保即使在面对进程故障的情况下，所有正确的进程也能以一致的方式接收到每条广播的消息。

### 实现

```c
# 当统一可靠广播系统初始化时触发的事件
upon event < urb, Init > do
    delivered := ∅;  # 初始化delivered集合，用于跟踪已交付的消息
    pending := ∅;  # 初始化pending集合，用于跟踪待交付的消息
    correct := Π;  # 初始化correct集合，包含所有进程
    for all m do ack[m] := ∅;  # 为每个消息m初始化一个空集合，用于跟踪确认接收消息的进程

# 当有广播消息请求时触发的事件
upon event < urb, Broadcast | m > do
    pending := pending ∪ {(self, m)};  # 将消息添加到pending集合
    trigger < beb, Broadcast | [DATA, self, m] >;  # 通过尽力而为广播发送消息

# 当尽力而为广播传递消息时触发的事件
upon event < beb, Deliver | p, [DATA, s, m] > do
    ack[m] := ack[m] ∪ {p};  # 记录确认接收消息m的进程p
    if (s, m) not in pending then
        pending := pending ∪ {(s, m)};  # 如果消息不在pending集合中，则添加
        trigger < beb, Broadcast | [DATA, s, m] >;  # 重新广播消息

# 当完美故障检测器检测到进程崩溃时触发的事件
upon event < P, Crash | p > do
    correct := correct \ {p};  # 从correct集合中移除崩溃的进程

# 检查是否可以交付消息的函数
function candeliver(m) returns Boolean is
    return (correct ⊆ ack[m]);  # 如果所有正确的进程都确认接收了消息m，则返回true

# 检查并交付消息的逻辑
upon exists (s, m) ∈ pending such that candeliver(m) ∧ m not in delivered do
    delivered := delivered ∪ {m};  # 将消息标记为已交付
    trigger < urb, Deliver | s, m >;  # 触发统一可靠广播的交付事件

```

### 表现

1. **最佳情况**
   - **通信步骤**：两个通信步骤。
   - **消息数量**：O(N²)条消息，其中N是系统中进程的数量。
   - **解释**：在最佳情况下，每个进程都需要广播一次消息，然后再根据其他进程的响应决定是否需要再次广播。由于每个进程都可能参与广播，所以总消息数量与进程数的平方成正比。

2. **最坏情况**
   - **通信步骤**：N+1步。
   - **消息数量**：O(N²)条消息。
   - **解释**：在最坏的情况下，可能需要多个步骤来确保所有正确的进程都接收到了消息。每个进程至少参与一次广播，加上可能的重复广播，使得总步骤数可能达到N+1。同样，由于每个进程都可能参与广播，总消息数量也是与进程数的平方成正比。

3. **与常规可靠广播的比较**
   - 相比于常规的可靠广播（ReliableBroadcast），统一可靠广播（UniformReliableBroadcast）需要额外的一个步骤来确保即使是崩溃的进程也能被考虑在内，从而保证消息的统一交付。这导致在最坏情况下，统一可靠广播的步骤数比常规可靠广播多一步。

# 因果顺序广播原语

## Logical clocks (Lamport’s clocks)逻辑时钟（兰波特时钟）

1. **每个进程维护自己的事件计数器**：
   - `cnt(p)`：这是进程`p`维护的一个事件计数器。每个进程都有自己的计数器来跟踪它经历的事件数量。

2. **计数器随每个步骤增加**：
   - 每当进程`p`执行一个操作（比如发送消息、处理本地事件等），它就会将`cnt(p)`增加1。这样，计数器反映了该进程经历的事件总数。

3. **发送消息时包含计数器**：
   - 当进程`p`发送消息时，它会在消息中包含当前的计数器值`cnt(p)`。这允许接收消息的进程了解发送者在发送消息时的“时间”状态。

4. **接收消息时更新计数器**：
   - 当进程`q`接收来自进程`p`的消息时，它会执行以下操作：
     - `cnt(q) := max(cnt(q), cnt(p)) + 1`：进程`q`将其计数器更新为`cnt(q)`和`cnt(p)`中的较大值加1。这个操作确保了`q`的计数器不仅反映了它自己的事件，还考虑了从`p`接收到的消息中蕴含的事件顺序。

尽管逻辑时钟是一种有用的工具，它们在表达事件间的因果关系方面存在一些局限性。下面我将解释这个问题以及向量时钟（Vector Clocks）作为解决方案。

1. **逻辑时钟的局限性**
   - 逻辑时钟通过为每个事件分配一个单一的数值来提供事件的顺序，但这种方法不能完全传达事件间的因果关系。
   - 举个例子：如果我们有两个事件a和b，即使逻辑时钟的值`lc(a)`小于`lc(b)`（意味着事件a在事件b之前发生），我们也不能肯定事件a导致了事件b。这是因为逻辑时钟不能区分并发事件和因果关联的事件。

2. **向量时钟的解决方案**

## Vector clocks矢量时钟

每个进程维护一个向量计数器，用于记录系统中所有进程的事件发生情况。

(p1的时钟，p2的时钟，p3的时钟，...)

### CausalOrderReliableBroadcast, instance crb.

