# 发生网络隔离，孤立节点会发生什么？

![image.png](https://cdn.nlark.com/yuque/0/2022/png/29672299/1659522781913-8f614e30-7676-42ee-b921-4f83b66f11b7.png#averageHue=%23fcfcfc&clientId=u830c0ed7-a584-4&from=paste&id=u367b67a9&name=image.png&originHeight=445&originWidth=720&originalType=url&ratio=1&rotation=0&showTitle=false&size=55532&status=done&style=none&taskId=u9c836158-e655-4737-9316-9de2ad81758&title=)

   Follower_2在electionTimeout倒计时结束之前没收到leader的心跳,会发起选举，并转为Candidate。每次发起选举时，会把Term加1。由于网络隔离，它既不会被选成Leader，也不会收到Leader的消息，而是会一直不断地发起选举。它的Term会不断增大。
   一段时间之后，这个节点的Term会变得非常大。在网络恢复之后，这个节点会把它的Term传播到集群的其他节点，导致其他节点更新自己的term，leader也会因为term没有它的大而变为Follower，集群发生一次中断。然后触发重新选主，但这个旧的Follower_2节点由于其日志不是最新，并不会成为Leader。所以，整个集群被这个网络隔离过的旧节点扰乱，leader自动降级为follower，其他follower也易主，更新自己的term，也即集群发生了短暂性的**抖动**，这段时间内集群不可用，显然是需要避免的。
解决方案：**两阶段提交思想的PreVote机制**。简而言之，当一个节点触发选举超时，想要发起一轮选举时，并不会立即自增Term，而是先发起一轮“预投票”，只有在该轮预投票获得了大多数节点的投票，才会开启正式的投票并转为新term的Candidate。因此，该机制避免了一个被网络分割的节点不断发起选举自增term的情况，当网络恢复，被分割的少数节点重新加入主网络，也不会扰乱集群。

## ETCD 源码实现

[https://masutangu.com/2018/07/08/etcd-raft-note-6/](https://masutangu.com/2018/07/08/etcd-raft-note-6/)

## PreVote流程

   整个过程有几个需要额外描述的关键点。首先，**发起PreVote并不会改变发起者的任何状态（包括Term和投票状态）**。这样做的目的是一旦集群中有人发起了正式的选举流程，PreVote不会阻塞正式选举。其次，其他节点在响应PreVote消息时，不会更新自身的投票状态。原因同上。
Sender侧：

1. 节点检测到timeout，发起PreVote流程
2. 转为PreCandidate**（不给自己投票，不递增自身任期）**
3. **重置timeout计时器 **
4. 发送prevote投票，其中包含的任期为**自身任期+1**
5. 等待prevote投票结果，如果收到超过半数投票:
   1. 转为Candidate（立即给自己投票，递增自身任期）
   2. 重置timeout计时器
   3. 发起正式选举流程
6. 如果没有收到半数投票：
   1. 转为follower（或者停留在PreCandidate状态）
   2. 重新陷入timeout循环

Receiver侧：

1. 收到RequestVote请求
2. 判断请求中携带的Term（例如遇到更大的term，需要立即转为follower, 并**重置timeout**）
3. 判断是否投票，如果确定投票：
   1. 如果是PreVote，**不更新自己的voteFor状态**，重置timeout计时器
   2. 如果是正式投票，更新自己的voteFor状态，持久化状态，重置timeout计时器
   3. 回复RPC
   4. （tips：个人觉得a中是否重置timeout计时器从prevote机制本身来说并不会影响raft的正确性，只会影响其效率和性能; 重置timeout比较保守，意味着follower相信该PreCandidate会马上再次发起正式投票，所以压制自己；而不重置timeout则是较激进的方案，保留自己可能马上也会timeout的权力，从而可以发起prevote，系统中可能同时存在多个PreCandidate。后者的优点是如果PreCandiate发送完prevote请求就发生了crash，follower可以更快地检测到自己timeout并发起新的prevote。缺点也很明显，增加了集群中的prevote RPC的数量，加大了重新选举的概率。另一方面，lab测试的正确性和心跳、timeout时间有关，如果在引入prevote之前，设置的timeout时间比较小，那么在引入prevote之后，需要适当增大timeout的范围，因为prevote带来了双倍的rpc开销和RTT时间，也加大了平票重新选举的几率（特别是后一种方案）。否则在复杂且不可靠的网络条件下，将有概率在tester规定的选举时间内没有选出leader，无法就某条日志达成共识，导致概率失败。特别是TestFigure8Unreliable2C这个测试。）
