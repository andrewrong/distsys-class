# 分布式系统的介绍
Copyright 2014, 2016, 2017 Kyle Kingsbury; Jepsen, LLC.

阅读下面的大纲,能基本了解分布式系统中几个关键分布式术语、分布式算法的概述和关于生产环境问题的探讨;

## 是什么让一个东西变成分布式的?

Lamport, 1987:

>  A distributed system is one in which the failure of a computer
>  you didn't even know existed can render your own computer
>  unusable.(分布式系统这么一种系统, 1. 系统节点的错误不会被用户感知 2. 更加不会因此让你的计算机变得不可用)

## 节点和网络

- 我们将分布式系统中的每一个组成部分称之为*节点*
  - 也可以称为: *进程*、*agent*和*actor*

### 节点

- latency的特征
  - 在同一个节点的操作被叫做"fast"
  - 在两个节点的操作被叫做"slow"
- 节点是可靠的
  - 失败的单元是整个节点
  - 你能感知问题的发生
  - 状态是连贯的
  - 状态转换以良好，有序的方式发生
  - 经典的建模是单线程状态机
- 节点本身也作为分布式系统
- 关于process正式的建模方式
  - Communicating Sequential Processes[go里面的csp]
  - Pi-calculus
  - Ambient calculus
  - Actor model[erlang]
- 节点故障的正式建模方式
  - Crash-stop[crash就自动停止]
  - Crash-recover[crash之后会恢复]
  - Crash-amnesia[crash之后会失忆]
  - Byzantine[拜占庭问题]

### 网络作为消息传输的通道

- 分布式节点之间的交互方式是通过网络
  - 人之间交互方式通过语言
  - 粒子通过field进行交互
  - 计算机之间的交互通过tcp、udp、sctp等等
- 我们将这些交互建模为节点之间发送的离散消息
- 发送消息是需要时间的
  - 这也是分布式系统*slow*的一部分
  - 我们称这个为latency
- 消息在网络传输过程中尝尝可能会被丢失
  - 这也是分布式系统不可靠的一部分
- 网络很少是同质的
  - 一些链接相比于其他的链接更加慢、更加容易出错等

### 同步网络

- 节点以锁步的方式执行
- 消息延迟有上限
- 高效且完美的全局时钟
- 证明一个东西比较容易

### 半同步的网络

- 其他的都类似于同步网络，只有时钟不再是绝对，而是近似;

### 异步网络

- 节点之间执行是相互独立,在任何时候，step time可能是处于[0,1]之间
- 消息延迟无上限
- 没有全局时钟
- ip网络被定义是异步网络
  - 但是在现实情况中很多变态的情况是不会发生的
  - 大部分网络恢复可能是几秒也可能是几周，但是不会是永远不恢复
    - Conversely, human timescales are on the orders of seconds to weeks
    - So we can't pretend the problems don't exist

## 当网络出现错误的时候

- 异步网络允许下面的错误:
  - 重复
  - 延迟
  - 丢包
  - 乱序
- 丢包和延迟是难以区分的; 因为作为客户端其实没有收到包的情况下，它无法区分是丢包还是延迟;
- 拜占庭网络中允许消息被任意修改
    - 包括重写消息内容
    - 他们大部分不会发生在现实网络中，但也只是大部分，不表示没有;

## 低层的协议

### TCP

- TCP *works*. Use it.
  - 不完美，但是基于它你能走的更加快;
- In practice, TCP prevents duplicates and reorders in the context of a single
  TCP conn
- 在现实环境中，tcp协议能保证一个连接中重复、乱序的问题;
  - 但是你可能打开了多余一个的连接
  - 如果没有其他原因，TCP conns最终会失败
  - 当这些发生的时候，你将会有两个选择:
    - 1. 抛弃这个包
    - 2. 基于tcp协议，在应用层来保证数据的准确性

### UDP

- 与tcp类似，但是不是以流的方式传播;
- 许多人使用UDP是为了速度;
- 在TCP FSM开销过高的情况下，UDP非常有用
  - 内存压力
  - 太多的短连接和socket重用
- Especially useful where best-effort delivery maps well to the system goals
- 特别适用于那些只需要尽力去完成系统目标的系统; 
  - 语音和视频聊天
  - 游戏
  - 有更加高层的协议来管理这些问题；基于UDP去实现流控、去重、重试、排序等功能,在应用层做;

## 时钟

- 当一个系统被分裂成多个部门,我们仍然有一些需求需要保证事务的顺序性;  
- 时钟能保证事务的顺序: 先是这个，然后是那个;

### 物理时钟

- In theory, the operating system clock gives you a partial order on system events
- 理论上，操作系统时钟为您提供系统事件的部分顺序
  - 警告: NTP服务没有你想象中那么好
  - 警告: 节点之间没有很好的同步
  - 警告: 硬件也是会飘的
  - 警告: By centuries
    - NTP might not care
    - http://rachelbythebay.com/w/2017/09/27/2153/
  - 警告: NTP仍然会跳
    - https://www.eecis.udel.edu/~mills/ntp/html/clock.html
  - 警告: Posix时间戳并不是单调递增
    - Cloudflare 2017: Leap second at midnight UTC meant time flowed backwards
    - At the time, Go didn't offer access to CLOCK_MONOTONIC
    - Computed a negative duration, then fed it to rand.int63n(), which paniced
    - Caused DNS resolutions to fail: 1% of HTTP requests affected for several hours
    - https://blog.cloudflare.com/how-and-why-the-leap-second-affected-cloudflare-dns/
  - 警告: 您想要测量的时间尺度可能无法实现
  - 警告: 线程会休眠
  - 警告: Runtimes can sleep
  - 警告: OS's can sleep
  - 警告: "Hardware" can sleep
- 不要用物理时钟

### Lamport Clocks[??]

- Lamport 1977: "Time, Clocks, and the Ordering of Events in a Distributed System"
  - One clock per process
  - Increments monotonically with each state transition: `t' = t + 1`
  - Included with every message sent
  - `t' = max(t, t_msg + 1)`
- If we have a total ordering of processes, we can impose a total order on
  events
  - But that order could be pretty unintuitive

### Vector Clocks(向量时钟)

- 将Lamport时钟推广到所有进程时钟的向量
- `t_i' = max(t_i, t_msg_i)`
- For every operation, increment that process' clock in the vector
- Provides a partial causal order
  - A < B iff all A_i <= B_i, and at least one A_i < B_i
  - Specifically, given a pair of events, we can determine causal relationships
    - A in causal past of B implies A < B
    - B in causal past of A implies B < A
    - Independent otherwise
- Pragmatically: the past is shared; the present is independent
  - Only "present", independent states need to be preserved
  - Ancestor states can be discarded
  - Lets us garbage-collect the past
- O(processes) in space
  - Requires coordination for GC
  - Or sacrifice correctness and prune old vclock entries
- Variants
  - Dotted Version Vectors - for client/server systems, orders *more* events
  - Interval Tree Clocks - for when processes come and go

### GPS & Atomic Clocks[google]

- Much better than NTP
  - Globally distributed total orders on the scale of milliseconds
  - Promote an asynchronous network to a semi-synchronous one
  - Unlocks more efficient algorithms
- Only people with this right now are Google
  - Spanner: globally distributed strongly consistent transactions
  - And they're not sharing
- More expensive than you'd like
  - Several hundred per GPS receiver
  - Atomic clocks for local corroboration: $$$$?
  - Need multiple types of GPS: vendors can get it wrong
  - I don't know who's doing it yet, but I'd bet datacenters in the
    future will offer dedicated HW interfaces for bounded-accuracy time.


## 总结

我们已经覆盖了分布式系统的主要功能点，节点之间通过网络来交换消息；节点之间和网络之间都可能因为各种方式失败; 类似于TCP和UDP这样的协议给我们一种比较原始的通道能用来处理通信，并且我们使用时钟来保证事务的顺序性; 接下里我们将讨论更加高级的分布式系统的属性;


## 可用性

- 可用性基本上是成功的尝试操作的一小部分。

### Total availability[全可用性]

- 幼稚的认为: 每一个操作都能成功
- 在一致性的语义中, 每一个在非错误节点上的操作都能成功
    - 关于失败的节点，您无能为力


### Sticky availability[粘性可用性]

- 针对非故障节点的每个操作都会成功
  - 这限制了客户端尝尝会连接到相同的节点上;

### High availability[高可用性]

- 比非分布式系统更加好的可用性

### Majority available[大部分可用性]

- 与集群中大多数节点操作是成功的
- 如果与集群中小部分节点操作会可能是失败的

### Quantifying availability[量化可用性]

- We talk a lot about "uptime"
  - Are systems up if nobody uses them?
  - Is it worse to be down during peak hours?
  - Can measure "fraction of requests satisfied during a time window"
  - Then plot that fraction over windows at different times
  - Timescale affects reported uptime
- Apdex
  - Not all successes are equal
  - Classify operations into "OK", "meh", and "awful"
  - Apdex = P(OK) + P(meh)/2
  - Again, can report on a yearly basis
    - "We achieved 99.999 apdex for the year"
  - And on finer timescales!
    - "Apdex for the user service just dropped to 0.5; page ops!"
- Ideally: integral of happiness delivered by your service?


## Consistency->[一致性](http://www.cs.cornell.edu/courses/cs734/2000FA/cached%20papers/SessionGuaranteesPDIS_1.html#HEADING7)

- A consistency model is the set of "safe" histories of events in the system
- 一致性模式是系统中事件的安全记录的集合;

### Monotonic Reads[单调读]

- 一旦已经读到过一个value，那么后面的所有其他的读都必须能读到这个value或者更新的value

### Monotonic Writes[单调写]

- 一旦写成功之后，后续一系列的写操作都是基于这次写而发生的;

### Read Your Writes[读你写的]

- 一旦你写入value，后面一系列的读操作都能读到这个value或者更新的value;

### Writes Follow Reads

- Once I read a value, any subsequent write will take place after that read
- 一旦我读取了一个值，任何后续写入都将在读取之后进行

### Serializability

- 所有操作(事务)都必须原子执行

### Causal consistency[因果一致性]

- Suppose operations can be linked by a DAG of causal relationships
  - A write that follows a read, for instance, is causally related
    - Assuming the process didn't just throw away the read data
  - Operations not linked in that DAG are *concurrent*
- Constraint: before a process can execute an operation, all its precursors
  must have executed on that node
- Concurrent ops can be freely reordered

### Sequential consistency[顺序一致性]

- Like causal consistency, constrains possible orders
- All operations appear to execute atomically
- Every process agrees on the order of operations
  - Operations from a given process always occur in order
  - But nodes can lag behind

### Linearizability[线性一致性]

- All operations appear to execute atomically
- Every process agrees on the order of operations
- Every operation appears to take place *between* its invocation and completion
  times
- Real-time, external constraints let us build very strong systems

### ACID隔离性等级

- ANSI SQL's ACID关于隔离的定义是混乱的
  - 规定中对一些定义是含糊不清并且有二义性
- Adya 1999: 弱一致性行: 分布式事务的广义理论与乐观实现
  - Read Uncommitted(未提交读)
    - Prevents P0: *dirty writes*
      - w1(x) ... w2(x)
      - Can't write over another transaction's data until it commits
    - Can read data while a transaction is still modifying it
    - Can read data that will be rolled back
  - Read Committed(已提交读)
    - Prevents P1: *dirty reads*
      - w1(x) ... r2(x)
      - Can't read a transaction's uncommitted values
  - Repeatable Read(可重复读)
    - Prevents P2: *fuzzy reads*
      - r1(x) ... w2(x)
      - Once a transaction reads a value, it won't change until the transaction
        commits
  - Serializable(可串行化)
    - Prevents P3: *phantoms*
      - Given some predicate P
      - r1(P) ... w2(y in P)
      - Once a transaction reads a set of elements satisfying a query, that
        set won't change until the transaction commits
      - Not just values, but *which values even would have participated*.
  - Cursor Stability
    - Transactions have a set of cursors
      - A cursor refers to an object being accessed by the transaction
    - Read locks are held until cursor is removed, or commit
      - At commit time, cursor is upgraded to a writelock
    - Prevents lost-update
  - Snapshot Isolation(快照独立 MVCC)
    - Transactions always read from a snapshot of committed data, taken before
      the transaction begins
    - Commit can only occur if no other committed transaction with an
      overlapping [start..commit] interval has written to any of the objects
      *we* wrote
      - First-committer-wins

## 权衡

- Ideally, we want total availability and linearizability
- Consistency requires coordination
  - If every order is allowed, we don't need to do any work!
  - If we want to disallow some orders of events, we have to exchange messages
- Coordinating comes (generally) with costs
  - More consistency is slower
  - More consistency is more intuitive
  - More consistency is less available

### 可用性和一致性

- CAP理论: 线性一致性 或者 全可用
- 还有其他的理论
  - Bailis 2014: Highly Available Transactions: Virtues and Limitations
  - Other theorems disallow totally or sticky available...(其他的理论中去掉了全可用性和粘性可用性)
    - Strong serializable
    - Serializable
    - Repeatable Read
    - Cursor Stability
    - Snapshot Isolation
  - You can have *sticky* available...
    - Causal
    - PRAM
    - Read Your Writes
  - You can have *totally* available...
    - Read Uncommitted
    - Read Committed
    - Monotonic Atomic View
    - Writes Follow Reads
    - Monotonic Reads
    - Monotonic Writes


### 收获和产量

- Fox & Brewer, 1999: Harvest, Yield, and Scalable Tolerant Systems
  - Yield: probability of completing a request
  - Harvest: fraction of data reflected in the response
  - Examples
    - Node faults in a search engine can cause some results to go missing
    - Updates may be reflected on some nodes but not others
      - Consider an AP system split by a partition
      - You can write data that some people can't read
    - Streaming video degrades to preserve low latency
  - This is not an excuse to violate your safety invariants
    - Just helps you quantify how much you can *exceed* safety invariants
    - e.g. "99% of the time, you can read 90% of your prior writes"
  - Strongly dependent on workload, HW, topology, etc
  - Can tune harvest vs yield on a per-request basis
   - "As much as possible in 10ms, please"
   - "I need everything, and I understand you might not be able to answer"

### 混合系统

- So, you've got a spectrum of choices!
  - Chances are different parts of your infrastructure have different needs
  - Pick the weakest model that meets your constraints
    - But consider probabilistic bounds; visibility lag might be prohibitive
    - See Probabilistically Bounded Staleness in Dynamo Quorums
- Not all data is equal
  - Big data is usually less important
  - Small data is usually critical
  - Linearizable user ops, causally consistent social feeds

### 回顾

可用性用来量化操作成功的概率;一致性模型是管理可能发生的操作以及何时发生的规则; 强一致性模型通常需要有性能和可用性上的代价，下面我们将用不同的方法来构建弱一致性系统到强一致性系统


## 尽可能的避免使用共识算法

### CALM conjecture(CALM 猜想)

- Consistency As Logical Monotonicity
  - If you can prove a system is logically monotonic, it is coordination free
  - What the heck is "coordination"
  - For that matter, what's "monotonic"?
- Monotonicity, informally, is retraction-free
  - Deductions from partial information are never invalidated by new information
  - Both relational algebra and Datalog without negation are monotone
- Ameloot, et al, 2011: Relational transducers for declarative networking
  - Theorem which shows coordination-free networks of processes unaware of the
    network extent can compute only monotone queries in Datalog
    - This is not an easy read
  - "Coordination-free" doesn't mean no communication
    - Algo succeeds even in face of arbitrary horizontal partitions
- In very loose practical terms
  - Try to phrase your problem such that you only *add* new facts to the system
  - When you compute a new fact based on what's currently known, can you ensure
    that fact will never be retracted?
  - Consider special "sealing facts" that mark a block of facts as complete
  - These "grow-only" algorithms are usually easier to implement
  - Likely tradeoff: incomplete reads
- Bloom language
  - Unordered programming with flow analysis
  - Can tell you where coordination *would* be required


### Gossip

- Message broadcast system
- Useful for cluster management, service discovery, CDNs, etc
- Very weak consistency
- Very high availability
- Global broadcast
  - Send a message to every other node
  - O(nodes)
- Mesh networks(网格网络)
  - Epidemic models
  - Relay to your neighbors
  - Propagation times on the order of max-free-path
- Spanning trees
  - Instead of a mesh, use a tree
  - Hop up to a connector node which relays to other connector nodes
  - Reduces superfluous messages
  - Reduces latency
  - Plumtree (Leit ̃ao, Pereira, & Rodrigues, 2007: Epidemic Broadcast Trees)

### CRDTs

- Order-free datatypes that converge
  - Counters, sets, maps, etc
- Tolerate dupes, delays, and reorders
- Unlike sequentially consistent systems, no "single source of truth"
- But unlike naive eventually consistent systems, never *lose* information
  - Unless you explicitly make them lose information
- Works well in highly-available systems
  - Web/mobile clients
  - Dynamo
  - Gossip
- INRIA: Shapiro, Preguiça, Baquero, Zawirski, 2011: "A comprehensive study of
  Convergent and Commutative Replicated Data Types"
  - Composed of a data type X and a merge function m, which is:
    - Associative: m(x1, m(x2, x3)) = m(m(x1, x2), x3)
    - Commutative: m(x1, x2) = m(x2, x1)
    - Idempotent:  m(x1, x1) = m(x1)
- Easy to build. Easy to reason about. Gets rid of all kinds of headaches.
  - Did communication fail? Just retry! It'll converge!
  - Did messages arrive out of order? It's fine!
  - How do I synchronize two replicas? Just merge!
- Downsides
  - Some algorithms *need* order and can't be expressed with CRDTs
  - Reads may be arbitrarily stale
  - Higher space costs

### HATs

- Bailis, Davidson, Fekete, et al, 2013: "Highly Available Transactions,
  Virtues and Limitations"
  - Guaranteed responses from any replica
  - Low latency (1-3 orders of magnitude faster than serializable protocols!)
  - Read Committed
  - Monotonic Atomic View
  - Excellent for commutative/monotonic systems
  - Foreign key constraints for multi-item updates
  - Limited uniqueness constraints
  - Can ensure convergence given arbitrary finite delay ("eventual consistency")
  - Good candidates for geographically distributed systems
  - Probably best in concert with stronger transactional systems
  - See also: COPS, Swift, Eiger, Calvin, etc


## 我们需要共识算法

- The consensus problem:
  - Three process types
    - Proposers: propose values
    - Acceptors: choose a value
    - Learners: read the chosen value
  - Classes of acceptors
    - N acceptors total
    - F acceptors allowed to fail
    - M malicious acceptors
  - Three invariants:
    - Nontriviality: Only values proposed can be learned
    - Safety: At most one value can be learned
    - Liveness: If a proposer p, a learner l, and a set of N-F acceptors are
      non-faulty and can communicate with each other, and if p proposes a
      value, l will eventually learn a value.

- Whole classes of systems are *equivalent* to the consensus problem
  - So any proofs we have here apply to those systems too
  - Lock services
  - Ordered logs
  - Replicated state machines

- FLP tells us consensus is impossible in asynchronous networks
  - Kill a process at the right time and you can break *any* consensus algo
  - True but not as bad as you might think
  - Realistically, networks work *often enough* to reach consensus
  - Moreover, FLP assumes deterministic processes
    - Real computers *aren't* deterministic
    - Ben-Or 1983: "Another Advantage of free choice"
      - Nondeterministic algorithms *can* achieve consensus

- Lamport 2002: tight bounds for asynchronous consensus
  - With at least two proposers, or one malicious proposer, N > 2F + M
    - "Need a majority"
  - With at least 2 proposers, or one malicious proposer, it takes at least 2
    message delays to learn a proposal.

- This is a pragmatically achievable bound
  - In stable clusters, you can get away with only a single round-trip to a
    majority of nodes.
  - More during cluster transitions.


### Paxos

- Paxos is the Gold Standard of consensus algorithms
  - Lamport 1989 - The Part Time Parliament
    - Written as a description of an imaginary Greek democracy
  - Lamport 2001 - Paxos Made Simple
    - "The Paxos algorithm for implementing a fault-tolerant distributed system
      has been regarded as difficult to understand, perhaps because the
      original presentation was Greek to many readers [5]. In fact, it is among the
      simplest and most obvious of distributed algorithms... The last section
      explains the complete Paxos algorithm, which is obtained by the straightforward
      application of consensus to the state machine approach for building a
      distributed system—an approach that should be well-known, since it is the
      subject of what is probably the most often-cited article on the theory of
      distributed systems [4]."
  - Google 2007 - Paxos Made Live
    - Notes from productionizing Chubby, Google's lock service
  - Van Renesse 2011 - Paxos Made Moderately Complex
    - Turns out you gotta optimize
    - Also pseudocode would help
    - A page of pseudocode -> several thousand lines of C++
- Provides consensus on independent proposals
- Typically deployed in majority quorums, 5 or 7 nodes
- Several optimizations (优化)
  - Multi-Paxos
  - Fast Paxos
  - Generalized Paxos
  - It's not always clear which of these optimizations to use, and which
    can be safely combined
  - Each implementation uses a slightly different flavor
  - Paxos is really more of a *family* of algorithms than a well-described
    single entity
- Used in a variety of production systems
  - Chubby
  - Cassandra
  - Riak
  - FoundationDB
  - WANdisco SVN servers
- New research: Paxos quorums need not be majority: can optimize for fast phase-2 quorums [Howard, Malkhi, and Spiegelman](https://arxiv.org/abs/1608.06696).
  - We're not sure how to USE this yet
  - Durability still requires distribution

### ZAB

- ZAB is the Zookeeper Atomic Broadcast protocol
- Junqueira, Reed, and Serafini 2011 - Zab: High-performance broadcast for
  primary-backup systems
- Differs from Paxos
- Provides sequential consistency (linearizable writes, lagging ordered reads)
  - Useful because ZK clients typically want fast local reads
  - But there's also a SYNC command that guarantees real-time visibility
  - (SYNC + op) allows linearizable reads as well
- Again, majority quorum, 5 or 7 nodes

### Humming Consensus

- Metadata store for managing distributed system reconfiguration
- Looks a little like CORFU's replicated log
- See also: chain replication

### Viewstamped Replication

- Presented as a replication protocol, but also a consensus algorithm
- Transaction processing plus a view change algorithm
- Majority-known values are guaranteed to survive into the future
- I'm not aware of any production systems, but I'm sure they're out there
- Along with Paxos, inspired Raft in some ways

### Raft

- Ongaro & Ousterhout 2014 - In Search of an Understandable Consensus Algorithm
- Lamport says it's easy, but we still have trouble grokking Paxos
  - What if there were a consensus algorithm we could actually understand?
- Paxos approaches independent decisions when what we *want* is state machines
  - Maintains a replicated *log* of state machine transitions instead
- Also builds in cluster membership transitions, which is *key* for real systems
- Very new, but we have a Coq proof of the core algorithm
- Can be used to write arbitrary sequential or linearizable state machines
  - RethinkDB
  - etcd
  - Consul

## Review

只添加事实而不是收回事实的系统需要较少的协调建立。我们可以使用gossip系统向其他进程广播消息，CRDT用于合并来自同行的更新，以及用于弱一致性的HAT交易。可串行性和线性化需要*共识*算法，我们可以通过Paxos，ZAB，VR或Raft; 下面我们将讨论关于分布式系统的扩展性


## 延迟(latency)的特点

- Latency is *never* zero
- 延迟永远不会是0
    - 虽然带宽在增长，但是它也违背不了物理的极限(光速)
    - 对延迟的预算会影响你的系统设计
        - 你可能需要关心系统需要进行多少次网络调用;
- 不同的系统对于"慢"有着不同的定义
    - 不同的目标
    - 不同的算法
  
### 多核系统

- 多核(特别是NUMA)架构和分布式系统非常的类似
  - 节点不会突然失败，但是节点之间的消息传递会很慢
  - 通过总线来提供通过网络
  - 通过在软件和硬件上建立一组复杂的协议来保证内存看上去清晰
  - 非临时存储指令(比如 MOVNTI)
- 他们通过抽象来隐藏分布式
  - MFENCE/SFENCE/LFENCE
    - Introduce a serialization point against load/store instructions
    - Characteristic latencies: ~100 cycles / ~30 ns
      - Really depends on HW, caches, instructions, etc
  - CMPXCHG Compare-and-Swap (sequentially consistent modification of memory)
  - LOCK
    - Lock the full memory subsystem across cores!
- 但是这些抽象也带来了损失
  - Hardware lock elision may help but is nascent
  - Blog: Mechanical Sympathy
  - Avoid coordination between cores wherever possible
  - Context switches (process or thread!) can be expensive
  - Processor pinning can really improve things
  - When writing multithreaded programs, try to divide your work into
    independent chunks
    - Try to align memory barriers to work unit boundaries
    - Allows the processor to cheat as much as possible within a work unit
  - See Danica Porobic, 2016: [High Performance Transaction Processing on Non-Uniform Hardware Topologies](https://infoscience.epfl.ch/record/219117/files/EPFL_TH7023.pdf)

### 本地网络

- You'll often deploy replicated systems across something like an ethernet LAN
- Message latencies can be as low as 100 micros
  - But across any sizable network (EC2), expect low millis
  - Sometimes, packets could be delayed by *five minutes*
  - Plan for this
- Network is within an order of mag compared to uncached disk seeks
  - Or faster, in EC2
    - EC2 disk latencies can routinely hit 20ms
      - 200ms?
        - *20,000* ms???
          - Because EBS is actually other computers
          - LMAO if you think anything in EC2 is real
            - Wait, *real disks do this too*?
              - What even are IO schedulers?
- But network is waaaay slower than memory/computation
  - If your aim is *throughput*, work units should probably take longer than a
    millisecond
  - But there are other reasons to distribute
    - Sharding resources
    - Isolating failures

### Geographic replication

- You deploy worldwide for two reasons
  - End-user latency
    - Humans can detect ~10ms lag, will tolerate ~100ms
      - SF--Denver: 50ms
      - SF--Tokyo: 100 ms
      - SF--Madrid: 200 ms
    - Only way to beat the speed of light: move the service closer
  - Disaster recovery
    - Datacenter power is good but not perfect
    - Hurricanes are a thing
    - Entire Amazon regions can and will fail
      - Yes, regions, not AZs
- Minimum of 1 round-trip for consensus
  - Maybe as bad as 4 rounds
    - Maybe 4 rounds all the time if you have a bad Paxos impl (e.g. Cassandra)
  - So if you do Paxos between datacenters, be ready for that cost!
  - Because the minimum latencies are higher than users will tolerate
    - Cache cache cache
    - Queue writes and relay asynchronously
    - Consider reduced consistency guarantees in exchange for lower latency
    - CRDTs can always give you safe local writes
    - Causal consistency and HATs can be good calls here
- What about strongly consistent stuff?
  - Chances are a geographically distributed service has natural planes of
    cleavage
    - EU users live on EU servers; US users live on US servers
    - Use consensus to migrate users between datacenters
  - Pin/proxy updates to home datacenter
    - Which is hopefully the closest datacenter!
    - But maybe not! I believe Facebook still pushes all writes through 1 DC!
  - Where sequential consistency is OK, cache reads locally!
    - You probably leverage caching in a single DC already

### 回顾

我们讨论了三种分布式系统的扩展方式: 通过同步网络连接的多核系统，通过局域网连接的分布式系统和通过专线、光纤连接的数据中心；多核系统中cpu交互是最终重要的性能中心，所以要知道如何去最小化cpu之间的协调; 在局域网中，系统的延迟已经足够的小; 在跨中心的备份系统中，高延迟推动了最终的一致性和数据中心固定解决方案。


## 通用的分布式系统

### 外部内存缓存

- Redis, memcached, ...
- Data fits in memory, complex data structures
- Useful when your language's built-in data structures are slow/awful
- Excellent as a cache
- Or as a quick-and-dirty scratchpad for shared state between platforms
- Not particularly safe

### KV stores(持久化的分布式kv存储)

- Riak, Couch, Mongo, Cassandra, RethinkDB, HDFS, ...
- Often 1,2,3 dimensions of keys
- O(1) access, sometimes O(range) range scans by ID
- No strong relationships between values
- Objects may be opaque or structured
- Large data sets
- Often linear scalability
- Often no transactions
- Range of consistency models--often optional linearizable/sequential ops.

### SQL databases(关系型数据库)

- Postgres, MySQL, Percona XtraDB, Oracle, MSSQL, VoltDB, CockroachDB, ...
- Defined by relational algebra: restrictions of products of records, etc
- Moderate sized data sets(中等大小的数据集)
- Almost always include multi-record transactions
- Relations and transactions require coordination, which reduces scalability
- Many systems are primary-secondary failover
- Access cost varies depending on indexes
- Typically strong consistency (SI, serializable, strict serializable)

### Search

- Elasticsearch, SolrCloud, ...
- Documents referenced by indices(存储的文件是为了索引)
- Moderate-to-large data sets
- Usually O(1) document access, log-ish search
- Good scalability
- Typically weak consistency

### Coordination services(协调服务)

- Zookeeper, etcd, Consul, ...
- Typically strong (sequential or linearizable) consistency
- Small data sets(小量的数据集)
- Useful as a coordination primitive for stateless services(为一些无状态服务做协调语义)

### Streaming systems(流式系统)

- Storm, Spark...
- Usually custom-designed, or toolkits to build your own.
- Typically small in-memory data volume
- Low latencies
- High throughput
- Weak consistency

### Distributed queues(分布式队列)

- Kafka, Kestrel, Rabbit, IronMQ, ActiveMQ, HornetQ, Beanstalk, SQS, Celery, ...
- Journals work to disk on multiple nodes for redundancy
- Useful when you need to acknowledge work now, and actually do it later
- Send data reliably between stateless services
- The *only* one I know that won't lose data in a partition is Kafka
  - Maybe SQS?
- Queues do not improve end-to-end latency
  - Always faster to do the work immediately
- Queues do not improve mean throughput
  - Mean throughput limited by consumers
- Queues do not provide total event ordering when consumers are concurrent
  - Your consumers are almost definitely concurrent
- Likewise, queues don't guarantee event order with async consumers
  - Because consumer side effects could take place out of order
  - So, don't rely on order
- Queues can offer at-most-once or at-least-once delivery
  - Anyone claiming otherwise is trying to sell you something
  - Recovering exactly-once delivery requires careful control of side effects
  - Make your queued operations idempotent
- Queues do improve burst throughput
  - Smooth out load spikes
- Distributed queues also improve fault tolerance (if they don't lose data)
  - If you don't need the fault-tolerance or large buffering, just use TCP
  - Lots of people use a queue with six disk writes and fifteen network hops
    where a single socket write() could have sufficed
- Queues can get you out of a bind when you've chosen a poor runtime

### 回顾

我们将具有一定结构的数据存储在外置内存系统中，他们是分布式系统之间的纽带; kv-store和关系型数据库通常是用来保存记录的;kvstore被用于存储一些独立的key并且不能与关系型数据库很好的配套; 但是对比关系型数据库它们有着更加好的扩展性和部分容错; 而sql存储主要是提供丰富的查询和强一致的事务保证;分布式搜索和协调服务完善了我们构建应用程序的基本工具包；流式处理被应用于持续的、低延迟的数据集处理，它更加倾向于是一种框架而不是数据库; 分布式队列更加注重的消息而非消息转化;


## A Pattern Language

- 对于构建分布式系统的一般建议
  - Hard-won experience
  - Repeating what other experts tell me
    - Over beers
  - Hearsay
  - Oversimplifications
  - Cargo-culting
  - Stuff I just made up
  - YMMV

### Don't distribute(不要轻易分布式)

- Rule 1: don't distribute where you don't have to()
  - Local systems have reliable primitives. Locks. Threads. Queues. Txns.
    - When you move to a distributed system, you have to build from ground up.
  - Is this thing small enough to fit on one node?
    - "I have a big data problem"
      - Softlayer will rent you a box with 3TB of ram for $5000/mo.
      - Supermicro will sell a 6TB box for ~$115,000 total.
  - Modern computers are FAST.
    - Production JVM HTTP services I've known have pushed 50K requests/sec
      - Parsing JSON events, journaling to disk, pushing to S3
    - Protocol buffers over TCP: 10 million events/sec
      - 10-100 event batches/message, in-memory processing
  - Can this service tolerate a single node's guarantees?
  - Could we just stand up another one if it breaks?
  - Could manual intervention take the place of the distributed algorithm?

### Use an existing distributed system(使用已经存在的分布式系统)

- 假如你不得不用分布式，那是否可以将工作迁移到其他软件之上
  - 分布式数据库和日志怎么样能满足你的要求不?
  - 我们能支付亚马逊上的服务来满足我们的要求吗？
  - 我们关心的和为此愿意付出价值的是什么？
  - 你为了使用或者操作分布式系统需要学习的代价是多少?

### Never fail(从来也不会错误)

- Buy really expensive hardware
- 购买真正的昂贵的硬件
- Make changes to software and hardware in a controlled fashion
  - Dry-run deployments against staging environments
- Possible to build very reliable networks and machines
  - At the cost of moving slower, buying more expensive HW, finding talent
  - HW/network failure still *happens*, but sufficiently rare => low priority

### Accept failure(接受错误)

- 分布式系统的特点不仅仅只有延迟，还有经常性的部分错误;
- Can we accept this failure and move on with our lives?
  - What's our SLA anyway?
  - Can we recover by hand?
  - Can we pay someone to fix it?
  - Could insurance cover the damage?
  - Could we just call the customer and apologize?
- Sounds silly, but may be much cheaper
  - We can never prevent 100% of system failures
  - Consciously choosing to recover *above* the level of the system
  - This is how financial companies and retailers do it!

### Backups(备份)

- Backups are essentially sequential consistency, BUT you lose a window of ops.
  - When done correctly
    - Some backup programs don't snapshot state, which leads to FS or DB
      corruption
    - Broken fkey relationships, missing files, etc...
  - Allow you to recover in a matter of minutes to days
  - But more than fault recovery, they allow you to step back in time
    - Useful for recovering from logical faults
      - Distributed DB did its job correctly, but you told it to delete key
        data

### Redundancy(冗余)

- OK, so failure is less of an option
- Want to *reduce the probability of failure*
- Have the same state and same computation take place on several nodes
  - I'm not a huge believer in active-spare
    - Spare might have cold caches, broken disks, old versions, etc
    - Spares tend to fail when becoming active
    - Active-active wherever possible
      - Predictability over efficiency
  - Also not a huge fan of only having 2 copies
    - Node failure probabilities just too high
    - OK for not-important data
    - I generally want three copies of data
      - For important stuff, 4 or 5
      - For Paxos and other majority-quorum systems, odd numbers: 3, 5, 7
        common
  - Common DR strategy: Paxos across 5 nodes; 3 or 4 in primary DC
    - Ops can complete as soon as the local nodes ack; low latencies
    - Resilient to single-node failure (though latencies will spike)
    - But you still have a sequentially consistent backup in the other DC
      - So in the event you lose an entire DC, all's not lost
    - See Camille Fournier's talks on ZK deployment
- Redundancy improves availability so long as failures are uncorrelated
  - Failures are not uncorrelated
    - Disks from the same batch failing at the same time
    - Same-rack nodes failing when the top-of-rack switch blows
    - Same-DC nodes failing when the UPS blows
    - See entire EC2 AZ failures
    - Running the same bad computation on every node will break every node
      - Expensive queries
      - Riak list-keys
      - Cassandra doomstones
    - Cascading failures
      - Thundering-herd
      - TCP incast

### Sharding(分片)

- The problem is too big
- Break the problem into parts small enough to fit on a node
  - Not too small: small parts => high overhead
  - Not too big: need to rebalance work units gradually from node to node
  - Somewhere around 10-100 work units/node is ideal, IMO
- Ideal: work units of equal size
  - Beware hotspots
  - Beware changing workloads with time
- Know your bounds in advance
  - How big can a single part get before overwhelming a node?
  - How do we enforce that limit *before* it sinks a node in prod?
    - Then sinks all the other nodes, one by one, as the system rebalances
- Allocating shards to nodes
  - Often built in to DB
  - Good candidate for ZK, Etcd, and so on
  - See Boundary's Ordasity

### Independent domains

- Sharding is a specific case of a more general pattern: avoiding coordination
  - Keep as much independent as possible
    - Improves fault tolerance
    - Improves performance
    - Reduces complexity
  - Sharding for scalability
  - Avoiding coordination via CRDTs
  - Flake IDs: generate globally unique identifiers locally
  - Partial availability: users can still use some parts of the system
  - Processing a queue: more consumers reduces the impact of expensive events

### ID structure

- Things in our world have to have unique identifiers
  - At scale, ID structure can make or break you
  - Consider your access patterns
    - Scans
    - Sorts
    - Shards
  - Sequential IDs require coordination: can you avoid them?
    - Flake IDs: *mostly* time-ordered identifiers, zero-coordination
      - See http://yellerapp.com/posts/2015-02-09-flake-ids.html
  - For *shardability*, can your ID map directly to a shard?
  - SaaS app: object ID can also encode customer ID
  - Twitter: tweet ID can encode user ID

### Immutable values(不可变得value)

- Data that never changes is trivial to store
  - Never requires coordination
  - Cheap replication and recovery
  - Minimal repacking on disk
- Useful for Cassandra, Riak, any LSM-tree DB.
  - Or for logs like Kafka!
- Easy to reason about: either present or it's not
  - Eliminates all kinds of transactional headaches
  - Extremely cachable
- Extremely high availability and durability, tunable write latency
  - Low read latencies: can respond from closest replica
  - Especially valuable for geographic distribution
- Requires garbage collection!
  - But there are good ways to do this

### Mutable identities(可变的value)

- Pointers to immutable values
- Pointers are small! Only metadata!
  - Can fit huge numbers of pointers on a small DB
  - Good candidate for consensus services or relational DBs
- And typically, not many pointers in the system
  - Your entire DB could be represented by a single pointer
  - Datomic only has ~5 identities
- Strongly consistent operations over identities can be *backed* by immutable
  HA storage
  - Take advantage of AP storage latencies and scale
  - Take advantage of strong consistency over small datasets provided by
    consensus systems
  - Write availability limited by identity store
    - But, reads eminently cachable if you only need sequential consistency
    - Can be even cheaper if you only need serializability
  - See Rich Hickey's talks on Datomic architecture
  - See Pat Helland's 2013 RICON West keynote on Salesforce's storage

### 总结

- Systems which are order-independent are easier to construct and reason about
- Also helps us avoid coordination
- CRDTs are confluent, which means we can apply updates without waiting
- Immutable values are trivially confluent: once present, fixed
- Streaming systems can leverage confluence as well:
  - Buffer events, and compute+flush when you know you've seen everything
  - Emit partial results so you can take action now, e.g. for monitoring
  - When full data is available, merge with + or max
  - Bank ledgers are (mostly) confluent: txn order doesn't affect balance
    - But when you need to enforce a minimum balance, no longer confluent
    - Combine with a sealing event (e.g. the day's end) to recover confluence
- See Aiken, Widom, & Hellerstein 1992, "Behavior of Database Production Rules"

### Backpressure(过载之后要怎么办)

- Services which talk to each other are usually connected by *queues*
- Service and queue capacity is finite
- 当下游服务无法处理负载时，如何处理它
  1. 资源无限被消耗然后挂掉
  2. 丢弃过量的请求
  3. 拒绝请求并且告诉客户端失败
  4. 客户端能知道后端压力比较大,并且主动慢下来
- 2-4 allow the system to catch up and recover
  - But backpressure reduces the volume of work that has to be retried
- Backpressure defers choice to producers: compositional
  - Clients of load-shedding systems are locked into load-shedding
    - They have no way to tell that the system is hosed
  - Clients of backpressure systems can apply backpressure to *their clients*
    - Or shed load, if they choose
  - If you're making an asynchronous system, *always* include backpressure
    - Your users will thank you later
- Fundamentally: *bounding resources*
  - Request timeouts (bounded time)
  - Exponential backoffs (bounded use)
  - Bounded queues
  - Bounded concurrency
- See Zach Tellman, "Everything Will Flow"

### Services for domain models(域模型服务-云平台的多租户系统)

- The problem is composed of interacting logical pieces
- Pieces have distinct code, performance, storage needs
  - Monolithic applications are essentially *multitenant* systems
    - Multitenancy is tough
    - But its often okay to run multiple logical "services" in the same process
- Divide your system into logical services for discrete parts of the domain
  model
  - OO approach: each *noun* is a service
    - User service
    - Video service
    - Index service
  - Functional approach: each *verb* is a service
    - Auth service
    - Search service
    - Dispatch/routing service
  - Most big systems I know of use a hybrid
    - Services for nouns is a good way to enforce *datatype invariants*
    - Services for verbs is a good way to enforce *transformation invariants*
    - So have a basic User service, which is used *by* an Auth service
  - Where you draw the line... well that's tricky
    - Services come with overhead: have as few as possible
    - Consider work units
    - Separate services which need to scale independently
    - Colocate services with tight dependencies and tight latency budgets
    - Colocate services which use complementary resources (e.g. disk and CPU)
      - By hand: Run memcache on rendering nodes
      - Newer shops: Google Borg, Mesos, Kubernetes
  - Services should encapsulate and abstract
    - Try to build trees instead of webs
    - Avoid having outsiders manipulate a service's data store directly
  - Coordination between services requires special protocols
    - Have to re-invent transactions
    - Go commutative where possible
    - Sagas
      - Was written for a single-node world: we have to be clever in distributed contexts
      - Transactions must be idempotent, OR commute with rollbacks
    - [Typhon/Cerberus](http://www.cs.ucsb.edu/~vaibhavarora/Typhon-Ieee-Cloud-2017.pdf)
      - Protocol for causal consistency over multiple data stores

### Structure Follows Social Spaces

- Production software is a fundamentally social artifact
- Natural alignment: a team or person owns a specific service
  - Jo Freeman, "The Tyranny of Structurelessness"
    - Responsibility and power should be explicit
    - Rotate people through roles to prevent fiefdoms
      - Promotes information sharing
    - But don't rotate too often
      - Ramp-up costs in software are very high
- As the team grows, its mission and thinking will formalize
  - So too will services and their boundaries
  - Gradually accruing body of assumptions about service relation to the world
  - Punctuated by rewrites to respond to changing external pressures
  - Tushman & Romanelli, 1985: Organizational Evolution
- Services can be libraries
  - Initially, *all* your services should be libraries
  - Perfectly OK to depend on a user library in multiple services
  - Libraries with well-defined boundaries are easy to extract into
    services later
- Social structure governs the library/service boundary
  - With few users of a library, or tightly-coordinated users, changes are easy
  - But across many teams, users have varying priorities and must be convinced
  - Why should users do work to upgrade to a new library version?
  - Services *force* coordination through a defined API deprecation lifecycle
    - You can also enforce this with libraries through code review & tooling
- Services enable centralized control
  - Your performance improvements affect everyone instantly
  - Gradually shift to a new on-disk format or backing database
  - Instrument use of the service in one place
  - Harder to do these things with libraries
- Services have costs
  - The failure-complexity and latency overhead of a network call
  - Tangled food web of service dependencies
  - Hard to statically analyze codepaths
  - You thought library API versioning was hard
  - Additional instrumentation/deployment
- Services can use good client libraries
  - That library might be "Open a socket" or an HTTP client
    - Leverage HTTP headers!
      - Accept headers for versioning
      - Lots of support for caching and proxying
    - Haproxy is an excellent router for both HTTP and TCP services
  - Eventually, library might include mock IO
    - Service team is responsible for testing that the service provides an API
    - When the API is known to be stable, every client can *assume* it works
    - Removes the need for network calls in test suites
    - Dramatic reduction in test runtime and dev environment complexity

### 回顾

如果可能的话，用单节点来替换分布式系统; 需要接受一些失败是不可避免的, 不同SLA都是具有代价的; 为了处理突如其来的错误，我们使用容灾的方式；为了提升可靠性，我们介绍了备份的方案；为了解决扩展性这个问题，我们用分片来分离数据; 不可能改变的数据更加容易被存储和缓存，也可以被可变标示引用,允许我们建立更加强一致性的保证; 随着软件的增长，不同组成部分都需要做到独立扩展，所以我们将服务分离开来，类似于微服务的感觉;



## 生产环境遇到的问题

- More than design considerations
- Proofs are important, but real systems do IO

### 分布式系统在你的环境中能被支持

- 了解生产中的分布式系统需要具有多种角色的人员的密切合作
  - 开发
  - 测试
  - 运维
- 同理心很重要
  - 开发需要关心产品
  - 运维需要关心实现
  - 好的交流能更快的对话

### 测试每一个点

- Type systems are great for preventing logical errors
  - Which reduces your testing burden
- However, they are *not* great at predicting or controlling runtime
  performance
- So, you need a solid test suite
  - Ideally, you want a *slider* for rigorousness
  - Quick example-based tests that run in a few seconds
  - More thorough property-based tests that can run overnight
  - Be able to simulate an entire cluster in-process
  - Control concurrent interleavings with simulated networks
  - Automated hardware faults
- Testing distributed systems is much, much harder than testing local ones
  - Huge swath of failure modes you've never even heard of
  - Combinatorial state spaces
  - Bugs can manifest only for small/large/intermediate time/space/concurrency

### "It's Slow"

- Jeff Hodges: The worst bug you'll ever hear is "it's slow"
  - Happens all the time, really difficult to localize
  - Because the system is distributed, have to profile multiple nodes
    - Not many profilers are built for this
    - Sigelman et al, 2010: Dapper, a Large-Scale Distributed Systems Tracing
      Infrastructure
    - Zipkin
    - Big tooling investment
  - Profilers are good at finding CPU problems
    - But high latency is often a sign of IO, not CPU
    - Disk latency
    - Network latency
    - GC latency
    - Queue latency
  - Try to localize problem using application-level metrics
    - Then dig in to process and OS performance
  - Latency variance between nodes doing the same work is an important signal
    - 1/3 nodes slow: likely node HW, re-route
    - 3/3 nodes slow: likely a logical fault: look at shard size, workload, queries
  - Tail latencies are magnified by fanout workloads
    - [Jeff Dean, 2013: The Tail at Scale](https://research.google.com/pubs/pub40801.html)
    - Consider speculative parallelism

### 监控每一个点

- Slowness (and outright errors) in prod stem from the interactions *between*
  systems
  - Why? Because your thorough test suite probably verified that the single
    system was mostly correct
  - So we need a way to understand what the system is doing in prod
    - In relation to its dependencies
    - Which can, in turn, drive new tests
  - In a way, good monitoring is like continuous testing
   - But not a replacement: these are distinct domains
   - Both provide assurance that your changes are OK
  - Want high-frequency monitoring
    - Production behaviors can take place on 1ms scales
      - TCP incast
      - ~1ms resolution *ideally*
    - Ops response time, in the limit, scales linearly with observation latency
      - ~1 second end to end latency
    - Ideally, millisecond latencies, maybe ms resolution too
      - Usually cost-prohibitive; back off to 1s or 10s
      - Sometimes you can tolerate 60s
  - And for capacity planning, hourly/daily seasonality is more useful
  - Instrumentation should be tightly coupled to the app
    - Measure only what matters
      - Responding to requests is important
      - Node CPU doesn't matter as much
    - Key metrics for most systems
      - Apdex: successful response WITHIN latency SLA
      - Latency profiles: 0, 0.5, 0.95, 0.99, 1
        - Percentiles, not means
        - BTW you can't take the mean of percentiles either
      - Overall throughput
      - Queue statistics
      - Subjective experience of other systems latency/throughput
        - The DB might think it's healthy, but clients could see it as slow
        - Combinatorial explosion--best to use this when drilling into a failure
    - You probably have to write this instrumentation yourself
      - Invest in a metrics library
  - Out-of-the-box monitoring usually doesn't measure what really matters: your
    app's behavior
    - But it can be really useful in tracking down causes of problems
    - Host metrics like CPU, disk, etc
    - Where your app does something common (e.g. rails apps) tools like New
      Relic work well
  - Superpower: distributed tracing infra (Zipkin, Dapper, etc)
    - Significant time investment
    - [Mystery Machine](https://www.usenix.org/system/files/conference/osdi14/osdi14-paper-chow.pdf)
      - Automatic inference of causal relationships between services from trace data
      - Identification of critical paths
      - Performance modeling new algorithms before implementation

### 记录

- Logging is less useful at scale
  - Problems may not be localized to one node
    - As requests touch more services, must trace through many logfiles
    - Invest in log collection infrastructure
      - ELK, Splunk, etc
  - Unstructured information is harder to aggregate
    - Log structured events

### 真实压测

- Load tests are only useful insofar as the simulated load matches the actual
  load
- Consider dumping production traffic
  - Awesome: kill a process with SIGUSR1, it dumps five minutes of request load
  - Awesome: tcpdump/tcpreplay harnesses for requests
  - Awesome: shadowing live prod traffic to your staging/QA nodes
- See Envoy from Lyft

### 版本管理

- Protocol versioning is, as far as I know, a wide-open problem
  - Do include a version tag with all messages
  - Do include compatibility logic
  - Inform clients when their request can't be honored
    - And instrument this so you know which systems have to be upgraded

### Rollouts(感觉是灰度发布的)

- Rollouts are often how you fix problems
- Spend the time to get automated, reliable deploys
  - Amplifies everything else you do
  - Have nodes smoothly cycle through to prevent traffic interruption
    - This implies you'll have multiple versions of your software running at
      once
      - Versioning rears its ugly head
  - Inform load balancer that they're going out of rotation
  - Coordinate to prevent cascading failures
- Roll out only to a fraction of load or fraction of users
  - Gradually ramp up number of users on the new software
  - Either revert or roll forward when you see errors
  - Consider shadowing traffic in prod and comparing old/new versions
    - Good way to determine if new code is faster & correct

### Feature flags(修改记录)

- We want incremental rollouts of a changeset after a deploy
  - Introduce features one by one to watch their impact on metrics
  - Gradually shift load from one database to another
  - Disable features when rollout goes wrong
- We want to obtain partial availability when some services are degraded
  - Disable expensive features to speed recovery during a failure
- Use a highly available coordination service to decide which codepaths to
  enable, or how often to take them
  - This service should have minimal dependencies
    - Don't use the primary DB
- When things go wrong, you can *tune* the system's behavior
  - When coordination service is down, fail *safe*!

### Chaos engineering(破坏者)

- Breaking things in production
  - Forces engineers to handle failure appropriately *now*, not in response to
    an incident later
  - Identifies unexpected dependencies in the critical path
    - "When the new stats service goes down, it takes the API with it. Are you
      *sure* that's necessary?"
  - Requires good instrumentation and alerting, so you can measure impact of
    events
  - Limited blast radius
    - Don't nuke an entire datacenter every five minutes
      - But *do* try it once a quarter
    - Don't break *too* many nodes in a replication group
    - Break only a small fraction of requests/users at a time

### Oh no, queues

- 每个队列都是一个可怕的地方，可怕的错误
  - 没有一个节点具有无限内存，所以你的队列必须是有界的;
  - 设置多大的队列呢？没人知道
  - 周期性的监控你的队列长度
- 队列可以平滑负载
  - 通过延迟为代价来增加吞吐量;
  - 假如你的负载高于你的容量，那么队列不能为你做什么
    - Shed load or apply backpressure when queues become full
    - Instrument this
      - When load-shedding occurs, alarm bells should ring
      - Backpressure is visible as upstream latency
  - Instrument queue depths
    - High depths is a clue that you need to add node capacity
      - End to end queue latency should be smaller than fluctuation timescales
    - Raising the queue size can be tempting, but is a vicious cycle
  - All of this is HARD. I don't have good answers for you
    - Ask Jeff Hodges why it's hard: see his RICON West 2013 talk
    - See Zach Tellman - Everything Will Flow


## 回顾

运行分布式系统需要开发者、qa和运维工程师一起配合; 静态分析、测试框架都能更加保证程序的正确性;而且需要对生产环境的系统设置报警和监控; 维护分布式系统的团队还需要一些工具：线上流量复制、版本管理、增量发布和修改记录； 最后如果需要队列，那么必须对它进行特别的监控;


