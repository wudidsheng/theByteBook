# 5.3.4 Multi Paxos

:::tip 额外知识
lamport 提到的 Multi Paxos 是一种思想，所以 Multi Paxos 算法实际上是个统称，Multi Paxos 算法是指基于 Multi Paxos 思想，通过多个 Basic Paxos 实例实现的一系列值的共识算法。
:::

lamport 的论文中对于 Multi Paxos 的描述称之为**Implementing a State Machine**，我们从理论转向现实问题，浅浅解释下分布式中的 State Machine。

在分布式环境中，如果我们要让一个服务具有容错能力，那么最常用最直接的办法就是让一个服务的多个副本同时运行在不同的节点上。但是，当一个服务的多个副本都在运行的时候，我们如何保证它们的状态都是同步的呢，或者说，如果让客户端看起来无论请求发送到哪一个服务副本，最后都能得到相同的结果？实现这种同步方法就是所谓的**状态机复制**（State Machine Replication）。

:::tip 额外知识

状态机（State Machine）来源于数学领域，全称是有限状态自动机，自动两个字也是包含重要含义的。给定一个状态机，同时给定它的当前状态以及输入，那么输出状态时可以明确的运算出来的。
:::

分布式系统为了实现多副本状态机（Replicated state machine），常常需要一个**多副本日志**（Replicated log）系统，如果日志的内容和顺序都相同，多个进程从同一状态开始，并且以相同的顺序获得相同的输入，那么这些进程将会生成相同的输出，并且结束在相同的状态。

<div  align="center">
	<img src="../assets/Replicated-state-machine.webp" width = "500"  align=center />
	<p></p>
</div>

多副本日志问题是如何保证日志数据在每台机器上都一样？不过，笔者先不回到这个问题，我们再继续返回到 Multi Paxos 。

既然 Paxos Basic 可以确定一个值，**想确定多个值（日志）那就运行多个 Paxos Basic ，然后将这些值列成一个序列（ Replicated log），在这个过程中并解决效率问题** -- 这就是 Multi Paxos 。

Replicated log 类似一个数组，因此我们需要知道当次请求是在写日志的第几位，Multi-Paxos 做的第一个调整就是要添加一个日志的 index 参数到 Prepare 和 Accept 阶段，表示这轮 Paxos 正在决策哪一条日志记录。

我们知道三台机可以容忍一台故障，为了更具体的分析，我们假设此时是 S~3~ 宕机。同时。当 S~1~ 收到客户端的请求命令 jmp（提案）时：

- 找到第一个没有 chosen 的日志记录：图示中是第 3 条 cmp 命令。
- 这时候 S~1~ 会尝试让 jmp 作为第 3 条的 chosen 值，运行 Paxos。
- 因为 S~1~ 的决策者已经接受了 cmp，所以在 Prepare 阶段会返回 cmp，接着用 cmp 作为提案值跑完这轮 Paxos，S~2~ 也将接受 cmp，S~1~ 的 cmp 变为 chosen 状态。继续找下一个没有 chosen 的位置 —— 也就是第 4 位。
- S~2~ 的第 4 个位置已经接受了 sub，所以在 Prepare 阶段会返回 sub，S~1~ 第 4 位会 chosen sub，继续接着往下找。
- 第 5 位 S~1~ 和 S~2~ 都为空，不会返回 acceptedValue，所以第 5 个位置就确定为 jmp 命令的位置，运行 Paxos，并返回请求。

<div  align="center">
	<img src="../assets/multi_paxos.png" width = "650"  align=center />
	<p></p>
</div>

通过上面的流程可以看出，每个提案在最优情况下需要 2 个 RTT。当多个节点同时进行提议的时候，对于 index 的争抢会比较严重，会造成 Split Votes。为了解决 Split Votes，节点需要进行随机超时回退，这样写入延迟就会增加。针对这个问题，一般通过如下方案进行解决：

- 选择一个 Leader，任意时刻只有一个 Proposer，这样可以避免冲突，降低延迟
- 消除大部分 Prepare，只需要对整个 Log 进行一次 Prepare，后面大部分 Log 可以通过一次 Accept 被 chosen

## 1. 选主

有很多办法可以进行选举，Lamport 提出了一种简单的方式：让 server_id 最大的节点成为 Leader（在上篇说到提案编号由自增 id 和 server_id 组成，就是这个 server_id）。

- 既然每台服务器都有一个 server_id，我们就直接让 server_id 最大的服务器成为 Leader，这意味着每台服务器需要知道其它服务器的 server_id，
- 为此，每个节点每隔 Tms 向其它服务器发送心跳，
- 如果一个节点在 2Tms 时间内没有收到比自己 server_id 更大的心跳，那它自己就转为 Leader，意味：
	- 该节点处理客户端请求
	- 该节点同时担任 Proposer 和 Acceptor。
- 如果一个节点收到比自己 server_id 更大的服务器的心跳，那么它就不能成为 Leader，意味着：
	- 该节点拒绝掉客户端请求，或者将请求重定向到 Leader
	- 该节点只能担任 Acceptor

值得注意的是，这是非常简单的策略，这种方式系统中同时有两个 Leader 的概率是较小的。即使是系统中有两个 Leader，Paxos 也是能正常工作的，只是冲突的概率就大了很多，效率也会降低。

## 2. 减少 Prepare 请求

在讨论如何减少 Prepare 请求之前，先讨论下 Prepare 阶段的作用，需要 Prepare 有两个原因：

- 屏蔽老的提案：但 Basic Paxos 只作用在日志的一条记录
- 检查可能已经被 chosen 的 value 来代替原本的提案值：多个 Proposer 并发进行提案的时候，新的 Proposal 要确保提案的值相同

我们依然是需要 Prepare 的。我们要做的是减少大部分 Prepare 请求，首先要搞定这两个功能。

对于 1，我们不再让提案编号只屏蔽一个 index 位置，而是让它变成全局的，即屏蔽整个日志。一旦 Prepare 成功，整个日志都会阻塞（值得注意的是，Accept 阶段还是只能写在对应的 index 位置上）。

对于2，需要拓展 Prepare 请求的返回信息，和之前一样，Prepare 还是会返回最大提案编号的 acceptedValue，除此之外，Acceptor 还会向后查看日志记录，如果要写的这个位置之后都是空的记录，没有接受过任何值，那么 Acceptor 就额外返回一个标志位 noMoreAccepted。

后续，如果 Leader 接收到超过半数的 Acceptor 回复了 noMoreAccepted，那 Leader 就不需要发送 Prepare 请求了，直接发送 Accept 请求即可。这样只需要一轮 RPC。

