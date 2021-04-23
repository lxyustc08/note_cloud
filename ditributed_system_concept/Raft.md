# Raft

## Raft算法提出目的

**Understandability**，提供一个比Paxos更容易理解的分布式算法。

Raft算法的特点：

+ Strong leader
  + A stronger form of leadership than other consensus algorithms
+ Leader election
  + uses randomized timers to elect leaders
+ Membership changes
  + uses a new joint consensus approach where the majorities of two different configurations overlap during transitions. This allows the cluster to continue operating normally during configuration changes.

## Replicated State Machines

Replicated State Machines是一致性算法需要解决问题的核心所在

Replicated State Machines是将state machines一致性地复制到多个服务器上，这样即使若干个服务器宕机状态机仍可正常操作。

Replicated State Machines被广泛应用于分布式系统中的fault tolerance problems中

Replicated State Machines架构如下图所示：

![Alt Text](distributed_system_pictures/Replicated_state_machines.png)

如图中所示，**Replicated state machines通常使用replicated log实现**，每个服务器节点中存储一个包含一系列命令的log，不同服务器中的log的存储的命令顺序以及命令内容均一致，而状态机是确定的，各服务器中的状态机根据相同的log必然 产生相同的状态。

通过replicated log概念，分布式系统的consensus algorithm算法需要解决的问题即replicated log的一致性。运行在每个服务器上的consensus modules接收客户端的命令并将命令写入其log中，该服务器上的consensus module与其他服务器的consensus module进行通信，确保其他服务器上的replicated log以相同的顺序接收相同的请求。一旦客户端的命令被正确同步到不同机器上后，集群表现为一个具备高可靠性的state machine。

实际应用过程中consensus algorithms通常具有如下特性：

+ 在非拜占庭条件下（包括网络延迟，丢包，重排，分包等）确保正确性；
+ 全功能性，任何服务器集群中的主要部分是可操作的，并且可与集群中的其他服务器进行通信，可与客户端进行通信；
  +  Thus, a typical cluster of ﬁve servers can tolerate the failure of any two servers. Servers are assumed to fail by stopping; they may later recover from state on stable storage and rejoin the cluster.
+ 并不严格使用时间来确保日志的一致性，当然在极端条件下，错误的时钟以及极端的消息延迟将导致可用性问题；
+ 在通常情况下，当集群中的主要部分对单独调用进行响应后，该调用对应的命令立即完成，这样可以保证少数性能较慢的服务器不会影响系统性能。

## What's wrong with Paxos

Paxos是一门异常成功的分布式一致性算法，不过其具有两个重要的缺点：

+ exceptionally difficult to understand.
+ dose not provide a good foundation for building practical implementations.

## Designing for understandability

Raft设计目标有如下几个：

+ provide a complete and practical foundation for system building
+ must be safe under all conditions and available under typical  operating conditions
+ must be efficient for common operations
+ muast be understandability

设计过程中解决问题的方式：

+ problem decomposition, 问题分解
  + raft设计过程中分为4部分： leader election, log replication, safety, membership changes
+ simplify the state machines，降低状态机复杂度
  + reducing the number of states to consider
    + logs are not allower to have holes
    + limits the ways in which logs can become inconsistent with each other
    + introduce nondeterminism to improve understandability

## The Raft consensus algorithm

Raft consensus algorithm是一个用于管理replicated log的算法

![Raft consensus condensed form](distributed_system_pictures/Raft_consensus.png "raft算法主要元素")

![Raft consensus key properties](distributed_system_pictures/Raft_key_properties.png "raft算法的关键要素")

Raft consensus Algorithm基本流程如下所示.

```mermaid
graph TD
    Start([start]) ---> state{leader is running}
    state ---> |No| electing[[electing leader]]
    electing[[electing leader]] ---> responbility[[give leader complete responbility of replicas log managing]]
    state ---> |Yes| accept[accepts logs from clients]
    responbility ---> accept
    accept ---> replicas[replicas logs to other server]
    replicas ---> notification[notification servers log is ready for state machine]
    notification ---> state
    notification ---> e([end])
```
