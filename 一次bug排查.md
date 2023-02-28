# 问题
在实现lab3后，并发测试有概率测试2000次中，1~3次fail在TestSnapshotRecoverManyClients3B这个测试，失败原因是apply out of order。
该测试做的是在一定时间内，几个客户端不断并发地发送命令给服务器，并且服务器可能会重启，还开启了快照。不要同于其他测试，这个测试开启了20个客户端，在时间限制（5s）内会不断并发【超级超级大量】发送RPC，开销远远大于其他基于静态用例的测试。
# BUG排查
之前的实现是start()和broadcast耦合在一起，即上层server每往raft push一条命令，raft 的start()就立即向所有follower发送异步broadcast()广播进行同步。而在broadcast()函数里，会判断每个follower的nextindex[i]来决定是发送installsnap/entries/heartbeat。
这样会导致两个问题，一个是大量RPC的性能开销问题，因为server可能频繁收到大量命令并下推h给raft层，每一次下推都要广播一次，每次广播同步给follower的日志里，有很大部分和上几次发送的日志(follower没来得及回复)有重复。
第二个问题是引入快照机制后会导致不一致的安全问题，也即follower会概率丢失某些log的问题。核心原因就是失去condinstallsnapshot的实现后，没有设计额外措施来保证snapshot在raft层和server层的安装是原子性的。下面讨论这个问题是怎么产生的。
follower安装leader发来的installsnapshot，涉及到两层安装，第一层是在raft层裁剪丢弃自己的log，设置commitindex和lastapplied与发来的snapshot保持一致；第二层是将snapshot数据发给server层，server层读取并覆盖自己的状态机。
在我的实现中，遵从了22的lab建议，没有实现condinstallsnapshot()，而是让其一直返回true，这个函数在以往的lab中是用于协调保证对于某个follower，它的raft层和server层安装leader发来的snapshot这一过程之间是原子性的。关于condinstallsnapshot发作用，可以参考[MIT6.824 spring21 Lab2D总结记录 - sun-lingyu - 博客园 (cnblogs.com)](https://www.cnblogs.com/sun-lingyu/p/14591757.html)
而installsnapshot的实现是如果判断snapshot[index]为可以安装，由于applcyh是无缓冲的，为了不阻塞在applych和避免死锁，执行流程是：【1】lock->在raft层安装->unlock【2】->将snapshot[index] push到applych。
另一边，raft向server层提交日志也是通过applych，这由一个单独的协程applier()负责，流程是【3】lock->检查commitindex和lastapplied->复制要发送的log[lastapplied +1]->unlock->将log[lastapplied +1] push到applych【4】->lock->再次检查commitindex和lastapplied->......
server层在某些test中（例如TestSnapshotRecoverManyClients3B）会在短时间内连续收到多个client的并发请求并立即push给下层raft的start()，每次start()都**立即**将对应的请求命令追加到自己的log中，然后调用异步广播并立即返回。也即leader的raft log会在极快时间内增长，增长的速度大大超过了同步的速度；另一边，server层检测到log 过大便又会调用snapshot通知raft层进行裁剪，因此有概率leader会在极短时间里向落后的follower异步发送两次或多次installsnapshot（因为每start一次就会广播一次，再加上心跳还会广播一次）
# 一个例子
下面是一个例子：
leader向follower1 发送完installsnapshot at index 20，由于自身log极快增长，server调用snapshot在commitindex处进行裁剪后，不得不马上又向follower1发出installsnapshot at index 30。follower1原先commitindex = 15，lastapplied= 12，applier()协程原计划将按照13，14的...顺序进行apply。
正如前面说的，follower1将会收到installsnapshot at index 20，在【1】处拿到锁lock，在raft层裁剪自己的log，并更新lastapplied=commitindex=20，但是在执行到【2】处进行unlock之前，installsnapshot at index 30的请求也早就到了，并阻塞在【1】处；另一边，applier也阻塞在【3】处。
那么当installsnapshot at index 20在【2】处解锁，在向applych 发送snapshot[20]的同时，有两种可能发生：

-  applier()在【3】处先抢到了锁，从index =21开始apply log[21]，解锁，installsnapshot at index 30再拿到锁，更新lastapplied=commitindex=30，向applych发送snapshot[30]，applier()再次拿到锁，从index=31开始apply log[31] 
-  installsnapshot at index 30马上在【1】处抢到了锁，并再次裁剪自己的log，并更新lastapplied=commitindex=30，unlock，有下面两个并发动作同时发生： 
   -  installsnapshot向applych发送snapshot[30]， 
   -  **同时**applier()拿到锁，向applych开始apply log[31] 

对于第一种情况，从applych这个channel的接受端来看，顺序流是：
> log[13], log[14], snap[20], log[21], snap[30]，log[31].... ；
> server端依次读取并应用，没有任何问题。

问题在于第二种情况，如果apply log[31]先占到applych，那么server从applych读取的顺序就变成了：
> log[13], log[14], snap[20], log[31], snap[30]，log[32]....

这样，就出现了一致性的问题，server在应用完snap[30]后，log[31]的执行效果就丢失了。
问题的根源就在于，raft层和server层安装snapshot之间没有保障原子性，插入了其他操作：

- raft层安装snap[20]和server层安装snap[20]之间，不是原子性的，在server层安装snap[20]之前，raft层又安装了snap[30]，并更改了lastapplied和commitindex
- raft层安装snap[30]和server层安装snap[30]之间，也不是原子性的，server层安装snap[30]之前，server层插入了log[31]
# 解决方法

1.  对于问题一，需要减少同步和installsnapshot的频率，也即减少广播次数，将start()和broadcast解耦，start()中每次收到新log并不立即广播。leader为每个follower分配一个单独的replicate协程，该协程负责向该follower批量同步log，也即每个follower的复制改为同步模式而不是异步：**在上一批日志同步得到follower回复之前，不同步下一批日志**（也即实现了粗batch，没有实现pipeline）。该协程在没有新log可以同步的时候wait，start()每次收到新log就唤醒所有follower的replicate协程。如果replicate协程正在等待上一批同步的结果，此期间start不断累积的log并不会触发频繁的同步发送，而是等待replicate协程完成上一批次的同步，进行下一次同步时，把此期间累积的log再全部批量同步。 
   -  心跳还是需要另起协程每100ms立即发送一次，不能等待，否则影响leader统治和选举 
   -  对于问题二，治标不治本，还是有概率出现raft层和server层安装snap之间插入其他apply，只不过大大降低了可能性。 
   - 缺点：对于单个follower的复制过程，是由单个replicator同步串行发送的，需要实现pipeline异步加速。
2.  对于问题二，每个server维护一个队列，每当自己的commitindex**发生更新(需要判断自己的commitindex是变大了而不是回退/没变)**，或是installsnapshot中接受更新的snap，都先将要发送的applymsg先发送到缓存队列中（由于生成applymsg和将applymsg送到队列中是全程在锁内的，因此不会插入其他动作，保证了log和index下标的有序性），再唤醒专门的applier协程读取缓存队列，批量顺序apply给server，并清空此次的缓存队列 
   -  也即转化为生产者消费者模型，当commitindex增加，从[old commitindex +1，new commitindex]之间的log都视为待apply，送往缓存队列 
   - 由于合并了普通log和snapshot的apply，后台只有一个apply协程进行异步提交，因此可以每次apply前检查snapshot发生的index后是否有未提交的log，直接删除。
   - raft将变得不需要维护lastappied这个变量，取而代之需要维护一个缓存队列applyqueue，里面存放待apply的applymsg 
   - crash后初始化，rf.commitindex需要恢复至快照的lastincludeindex ，rf.commitindex和缓存队列applyqueue都不需要持久化，每次重启初始化为空即可，因为应用层状态机需要重放上次snapshoot点之后的log来回复crash时的状态。
