# Lab2
在实现lab2之前，我认为应该进行以下准备工作，希望能帮助看到这篇笔记的你：

- 通过lab1的小试牛刀，熟悉了Go的基本语法和编码风格，熟悉Go的并发相关原语和状态（协程、锁及死锁的排查、race、channel、waitgroup等等）、Go中基本数据结构的使用（如map、切片、深/浅拷贝等）以及**Debug日志的打印、留档和排查技巧（极其重要，这是824实验的核心难点）**
- 熟读并理解了raft原文第7章及其之前的部分（不包括第6章）
- 通读官方lab2给的所有实验intro和Hint 
- 过一遍[Student Guide](https://thesquareplanet.com/blog/students-guide-to-raft/)、[Guidance](https://pdos.csail.mit.edu/6.824/labs/guidance.html)和[Structure Advice](https://pdos.csail.mit.edu/6.824/labs/raft-structure.txt)，并且在实验过程中你一定会反复不停的回头看它们来排查某些问题
- 并行测试脚本的使用能大幅减少你的测试时间，并且能更快发现潜在的bug
- **保有对论文任何算法描述部分的足够敬畏和思考（尤其是figure2）**。在lab2的阶段，对于初次接触分布式的同学，任何lab未提及的“改造”、“简化”、“优化”都是不必要且可能致命的。**请先保证自己代码的正确性（至少每个测试在lab的框架内稳定pass 1k+），才能在此基础上考虑以上尝试。**
- 如果你更加注重工业界的落地实现和实际生产环境的可用，那么824并不适合你，建议转向ETCD源码阅读和PingCap发起的Talent Plan项目（实现TinyKv，6.824的plus版）
- **不要照搬他人公开的实现，也不要公开自己的完整代码库**（这也是824的课程规定）。很多公开出来的实现并不是正确的，大部分甚至都无法稳定pass 100次、500次。可以参考他人的实现思路、伪代码和讨论，但他们并不一定是bug free或最佳实现（**当然也包括我**），你应该作出你自己的思考、判断和实践。
没有自己经历一场饱含血与泪的日志Debug排查和捶胸顿足后的顿悟是无法理解这门课以及Raft的真正内涵的。
## Intro
该实验的目标是实现基础的Raft协议层，也即小论文中的figure2 以及figure13为代表的snapshot机制。一些骨干代码官方已经给出，在其基础上需要我们设计数据结构并填充自己的方法实现。具体来说，lab2由四个小实验构成：

- 2A是实现基本的选举
- 2B是实现日志同步
- 2C是实现日志及状态的持久化
- 2D是实现snapshot机制

![JMTE9HC$DZPDLKZJ0{A@%D5.png](https://cdn.nlark.com/yuque/0/2022/png/29672299/1662892720734-be5f1a72-193a-43ac-b1bb-5cc7e29a29e2.png#averageHue=%23f9f6f3&clientId=u0538ae75-a277-4&from=paste&height=1191&id=u0931effe&name=JMTE9HC%24DZPDLKZJ0%7BA%40%25D5.png&originHeight=1191&originWidth=991&originalType=binary&ratio=1&rotation=0&showTitle=false&size=523302&status=done&style=none&taskId=u04fdde0c-f300-48a7-8eae-653a86549f8&title=&width=991)
**figure 2 **
![9Q(9N3{ZLMYBWE1SA4`4{%6.png](https://cdn.nlark.com/yuque/0/2022/png/29672299/1662893076069-0f1f6f4d-f8a2-4a89-8656-aaccb3ca36dd.png#averageHue=%23f9f6f3&clientId=u0538ae75-a277-4&from=paste&height=623&id=u7e65d7af&name=9Q%289N3%7BZLMYBWE1SA4%604%7B%256.png&originHeight=630&originWidth=469&originalType=binary&ratio=1&rotation=0&showTitle=false&size=118409&status=done&style=none&taskId=u32ed80b4-fb26-409a-abb5-5e4043bed02&title=&width=464)
         ** figure 13**
## Raft peer结构及其初始化
```go
type Raft struct {
    mu        sync.RWMutex          // Lock to protect shared access to this peer's state
    peers     []*labrpc.ClientEnd   // RPC end points of all peers
    persister *Persister           // Object to hold this peer's persisted state
    me        int                  // this peer's index into peers[]
    dead      int32                // set by Kill()

    // Persistent state on all servers:
    currentTerm int
    votedFor    int
    log         Log

    // Volatile state on all servers:
    applych      chan ApplyMsg
    applyCond    *sync.Cond
    role         int
    latestHBtime time.Time
    commitIndex  int
    //lastApplied  int
    applyQueue	 []ApplyMsg

    // Volatile state on candidate, Reinitialized after election timeout:
    // receivedVotes int

    // Volatile state on leaders, Reinitialized after election
    nextIndex  []int
    matchIndex []int
    replicatorCond []*sync.Cond
}


func Make(peers []*labrpc.ClientEnd, me int,
	persister *Persister, applyCh chan ApplyMsg) *Raft {
	rf := &Raft{}
	rf.peers = peers
	rf.persister = persister
	rf.me = me

	rf.currentTerm = 0
	rf.votedFor = -1
	rf.applych = applyCh
	rf.applyCond = sync.NewCond(&rf.mu)
	rf.role = FOLLOWER
	rf.resetTimeOut()
	rf.log = make([]Entry, 1)
	rf.log[0] = Entry{0, 0, nil} //索引0存放snapshot，此后的日志从1开始存储

	rf.nextIndex = make([]int, len(rf.peers))
	rf.matchIndex = make([]int, len(rf.peers))
	rf.replicatorCond = make([]*sync.Cond, len(rf.peers))
	rf.initialLeaderState(true) //内部无锁
	
	rf.applyQueue = make([]ApplyMsg, 0)
	// initialize from state persisted before a crash
	rf.readPersist(persister.ReadRaftState()) // 第一次集群启动时无事发生，之后crash都从这里恢复
	rf.commitIndex = rf.log[0].Index
	
	// start ticker goroutine to start elections
	go rf.ticker()
	go rf.applier_cond(applyCh)

	return rf
}
```
## 选举
### 超时选举
这里我采用的是sleep+检测时间戳的方式实现超时检测，这也是[Structure Advice](https://pdos.csail.mit.edu/6.824/labs/raft-structure.txt)推荐的最直观和简单有效的方法。Hint中也提到，如果使用Go的time.Ticker和time.Timer可能会遇到一些难以排查的BUG和问题。并且sleep的实现思想是任何语言都通用的，不涉及语言层面的特性和语法糖。
在发送投票时，注意不能持有锁，任何可能耗时、等待和阻塞的操作都应该在锁外进行。

- 节点的状态可能在任意时刻发生修改，因此如果在发起投票的全程持有锁，会阻碍其他并发协程对节点状态的及时更新。
- 比较好的方式是持有锁后更改节点状态和保存此时的投票信息，再立即释放锁，之后再用保存的信息并行发送RPC调用。即使在发送和等待投票结果时节点状态改变了，Raft的选举机制和安全性也可以使这些带有过期信息的RPC得到拒绝/过滤，从而正确处理。
```go
// The ticker go routine starts a new election if this peer hasn't received
// heartsbeats recently.
func (rf *Raft) ticker() {
	for rf.killed() == false {
		
        timeout := generateTimeout(OPEN_PREVOTE)
		time.Sleep(time.Duration(timeout) * time.Millisecond)
		
		if rf.cherkIsTimeOut(timeout) {
			rf.raiseVote(OPEN_PREVOTE)
		}
	}
}


// 修改状态且并行发起投票协程
func (rf *Raft) raiseVote(preVote bool) {
	rf.mu.Lock()

	// 复制此时自己的请求投票信息，后续用于传递给并发的handleRequestVote()，确保在发出去的投票请求信息是此时此状态下的term
	// 因为可能在并发发送请求投票的时候，自己会变成其他更高term的candidate的follower，并更新了自己的term
	RVargs := RequestVoteArgs{
		rf.currentTerm + 1, //Whether it is vote or prevote
		rf.me,
		rf.log.last().Term,
		rf.log.last().Index,
		preVote,
	}

	if preVote {
		rf.stepToPreCandidate() // only set state to PRECANDIDATE
	} else {
		rf.stepToCandidate() // set rf.currentTerm ++ and voteFor = me
		rf.persist()
	}
	rf.resetTimeOut()

	rf.mu.Unlock()

	receivedVotes := 1 // vote for itself
	for server := 0; server < len(rf.peers); server++ {
		if server != rf.me {
			DPrintf("[raft%d-ticker]:send vote request to peer%d", RVargs.CandidateId, server)
			go rf.handleRequestVote(&receivedVotes, server, RVargs, preVote)
		}
	}
}
```
### 投票及其处理
请求投票函数RequestVote的实现遵照论文的figure2

- 需要注意一旦请求投票方的任期比自己大，都要立马转为follower
- 选举超时器的重置需要严格保证**在且只在**以下三种情况内进行：
   1. 自己检测到超时，发起了一轮新的选举
   2. 自己向Candidate投出了一票
   3. 收到了来自当前任期**合法leader**的RPC调用，包括AppendEntries 和 RequestVote RPC（也包括2D中的InstallSnapshot RPC）

投票机处理函数handleRequestVote负责向节点发送投票请求，并等待处理：

- 注意由于RPC调用是同步的，协程会阻塞直到得到回复或RPC调用超时，因此在这部分不能持有锁
- 如果得到了RequestVote RPC回复，须要判断自己的状态是否改变才能进行进一步的处理。一种有效的方法是：先判断自己的当前状态（Term、身份），是否与发送RequestVote RPC时的参数相同。
- 当引用传递的投票计数达到大多数，即可转为leader。后续仍旧进行的投票计数，需要先检测自己的状态是否还是candidate，否则可能在转为leader后重复转为leader，重置状态导致了后退。
```go
func (rf *Raft) RequestVote(args *RequestVoteArgs, reply *RequestVoteReply) {
	// Your code here (2A, 2B).
	rf.mu.Lock()
	defer rf.mu.Unlock()

	vote := false
	
	if args.Term >= rf.currentTerm {
		if args.Term > rf.currentTerm { //term比自己大，转为follower，但是否投票还要判断log是否是比自己新
			rf.stepToFollower(args.Term)
			rf.persist()
		}
		if (rf.votedFor == -1 || rf.votedFor == args.CandidateId) &&
			rf.isUptoDate(args.LastLogTerm, args.LastLogIndex) {
			vote = true
			if !args.PreVote {
				rf.votedFor = args.CandidateId
				rf.persist()
			}
			rf.resetTimeOut()
		}
	}

	reply.VoteGranted = vote
	if vote {
		DPrintf("[raft%d-RequestVote]: vote to peer %d", rf.me, args.CandidateId)
	}
	reply.Term = rf.currentTerm
}

//
// 向单个peer发送投票请求并处理投票回复
//
func (rf *Raft) handleRequestVote(receivedVotes *int, server int, RVargs RequestVoteArgs, preVote bool) {

	RVreply := RequestVoteReply{}
	ok := rf.sendRequestVote(server, &RVargs, &RVreply) // 一直阻塞直到得到回复

	if !ok {
		return
	}

	rf.mu.Lock()
	defer rf.mu.Unlock()
	// 收到投票回复的时候，自己的任期还没变（说明目前仍然是发出投票时的状态，且任期内没有超时），则处理；
	// 否则不处理，因为收到该回复时自己的任期已经变化了（要么变成follwer，要么已经当选leader，要么转为CANDIDATE，要么自增任期重新发起了下一轮选举）

	// 检查rf.currentTerm是否还是自己当时发送请求时的那个term，并且：
	// 如果是正式投票，需要判断收到该回复时自己还是不是当时的candidate，避免已经成为leader后又再次steptoleader，重复重置nextindex导致倒退，增加RPC调用
	// 如果是preVote，需要判断收到该回复时自己还是不是当时的precandidate，避免成为candidate后又再次steptocandidate

	switch preVote {
	case true:
		if rf.currentTerm+1 == RVargs.Term && rf.role == PRECANDIDATE { //
			if RVreply.VoteGranted {
				*receivedVotes++
				DPrintf("[raft%d-handleRequestVote]: recived prevote from peer%d, now sum votes %d/%d ", rf.me, server, receivedVotes, len(rf.peers)/2)
				if *receivedVotes > len(rf.peers)/2 {
					BPrintf("[raft%d-handleRequestVote]: raft%d become precandidate", rf.me, rf.me)
					go rf.raiseVote(false) // 里面有锁，所以要go，或者先unlock再调用
				}
			} else if RVreply.Term > rf.currentTerm {
				rf.stepToFollower(RVreply.Term)
				rf.persist()
				rf.resetTimeOut()
			}
		}
	case false:
		if rf.currentTerm == RVargs.Term && rf.role == CANDIDATE { //
			if RVreply.VoteGranted {
				*receivedVotes++
				DPrintf("[raft%d-handleRequestVote]: recived vote from peer%d, now sum votes %d/%d ", rf.me, server, receivedVotes, len(rf.peers)/2)
				if *receivedVotes > len(rf.peers)/2 {
					rf.stepToLeader()
					if RAFT_APPEND_NOP {
						go rf.appendEmptyEntry() // 里面调用start()有锁，所以要go，或者先unlock再调用
					}
					BPrintf("[raft%d-handleRequestVote]: raft%d become leader", rf.me, rf.me)
				}
			} else if RVreply.Term > rf.currentTerm {
				rf.stepToFollower(RVreply.Term)
				rf.persist()
				rf.resetTimeOut()
			}

		}
	}
}
```
## 日志结构
封装了一下日志结构，并为其操作定义了专门的索引方法。

- 如果你使用的是切片等结构来实现日志存储，由于后续lab 2D需要实现snapshot来丢弃掉部分旧的日志，也即日志结构中第N项日志的真实Index可能并不是N，因此**不要使用下标索引来直接定位日志项**。
- 建议在lab 2D之前从下标1开始存放日志，预留切片/数组的第0项，用于2D中存放snapshot信息结构。
```go
type Entry struct {
	Term     int
	Index    int
	Commmand interface{}
}

type Log []Entry

func (log Log) at(index int) Entry {
	return log[index-log[0].Index]
}

func (log Log) last() Entry {
	return log[len(log)-1]
}

// return a deepcopy [a, b) of log by entry index，return empty slice if a>=b
func (log Log) cut(a int, b int) []Entry {
	if a >= b {
		return Log{}
	}
	entries := make(Log, b-a)
	copy(entries, log[a-log[0].Index:b-log[0].Index])
	return entries
}
```
## 日志同步

- start函数要立即返回，不能执行任何耗时操作，如果需要执行某些耗时、阻塞的操作，建议使用协程或者cond通知的方式
- 复制模型-为每个Follower分配一个单独的Replicator协程，负责向该节点进行日志同步
- 日志匹配不能直接从prelogindex开始截断follower的日志然后拼接上发来的entries；因为收到的rpc可能是过期的。 
   - 因此当prelogindex和prelogterm都匹配：需要再依次往下检测，直到遍历完entries，或者只在log中覆盖rpc中包含的那几个entries的位置，并且不能丢弃后面的log。如果直接截断，该rpc又是过期的，会导致follower截断掉最新的几个log，导致回退。
   - 基于上面的考虑，我的实现是，如果preindex的位置匹配，仍旧遍历完RPC发来的entries，从PreLogIndex+1开始，挨个比对自己的log和entries中的所有entry，看看有不有conflict，如果没有冲突则不做任何操作。
- 快速回退：这里采用的是[Student Guide](https://thesquareplanet.com/blog/students-guide-to-raft/)提到的方法，实际上查找本地日志中某个任期为N的日志，可视作查找算法进行优化，例如二分查找。
- 日志提交：采用生产者消费者模型，每当CommitIndex更新，则将待提交日志送入缓存区，并异步唤醒apply协程消费缓存中的所有日志向上层提交。
## 异步日志提交

- 等待唤醒+缓存区消费+异步提交
- 注意在向channel apply日志时，不能持有锁，否则会导致[Student Guide](https://thesquareplanet.com/blog/students-guide-to-raft/)中提到的four-way deadlock
```go
func (rf *Raft) applier_cond(applyCh chan ApplyMsg) {
	for rf.killed() == false {
		rf.mu.Lock()
		for len(rf.applyQueue) == 0 {
			rf.applyCond.Wait()
		}
        
		copyqueue := make([]ApplyMsg, 0)
		copyqueue = append(copyqueue, rf.applyQueue...)
		rf.applyQueue = make([]ApplyMsg, 0)
		
		rf.mu.Unlock()
		for _, msg := range(copyqueue){
			applyCh <- msg
			if msg.CommandValid {
				BPrintf("[raft%d-applier]: apply log[%d] to server", rf.me, msg.CommandIndex)
			} else if msg.SnapshotValid {
				BPrintf("[raft%d-applier]: apply snap[%d] to server", rf.me, msg.SnapshotIndex)
			}
 		//time.Sleep(time.Duration(10) * time.Millisecond)
		}
	}
}
```
## 持久化
该部分对应2C，代码量并不大，只需要注意以下几点：

- decode的顺序要按照encode的顺序
- 持久化代码的准确插入：当任期、投票记录和日志任一发生更改都要及时持久化
   - 在后面的实验中你可能会频繁回过头来改动、添加Raft的代码，注意任何地方一旦修改了上述三个状态，都要进行持久化
   - 比较方便的做法是，在所有的可能修改上述状态的函数开头添加`defer 持久化代码`，但这和函数级粒度的锁一样，显然会导致损失性能。
- 2C真正难的地方在于它的测试用例比2B更为严苛，模拟了更多的极端情况，它有可能测试出很多你在2B中的潜在的bug，特别是关于论文中figure 8的一些测试。
```go
func (rf *Raft) persist() {
	rf.persister.SaveRaftState(rf.persistState())
}

func (rf *Raft) persistState() []byte {
	w := new(bytes.Buffer)
	e := labgob.NewEncoder(w)
	e.Encode(rf.currentTerm)
	e.Encode(rf.votedFor)
	e.Encode(rf.log)
	return w.Bytes()
}

//
// restore previously persisted state.注意要按encode顺序decode
//
func (rf *Raft) readPersist(data []byte) {
	if data == nil || len(data) < 1 { // bootstrap without any state?
		return
	}
	r := bytes.NewBuffer(data)
	d := labgob.NewDecoder(r)
	var currentTerm int
	var votedFor int
	var log []Entry
	if d.Decode(&currentTerm) != nil ||
		d.Decode(&votedFor) != nil ||
		d.Decode(&log) != nil {
		// error...
		panic("fail to decode state")
	} else {
		rf.currentTerm = currentTerm
		rf.votedFor = votedFor
		rf.log = log
		//BPrintf("节点%d重载：term:%d ;votefor:%d ;log长度：%d", rf.me, currentTerm, votedFor, len(log))
	}
}
```
## 快照及其恢复
这里首先需要说明的是，22的lab中官方已经不建议实现CondInstallSnapshot，关于该函数的历史缘由可以参考。
实现snapshot机制主要有三块构成：

- （Leader）Snapshot，Leader接受上层应用层的快照调用，裁剪日志并存储快照状态
- （Leader）handelInstallSnapshot，Leader向落后的follower发送快照并等待回复、处理
- （Follower）InstallSnapshot，用于安装Leader发送过来的快照
```go
//
// A service wants to switch to snapshot.  Only do so if Raft hasn't
// have more recent info since it communicate the snapshot on applyCh.
//
func (rf *Raft) CondInstallSnapshot(lastIncludedTerm int, lastIncludedIndex int, snapshot []byte) bool {

	// Your code here (2D).
    // 这是个废弃的接口，历史lab版本遗留，简单返回true即可

	return true
}

// the service says it has created a snapshot that has
// all info up to and including index. this means the
// service no longer needs the log through (and including)
// that index. Raft should now trim its log as much as possible.
func (rf *Raft) Snapshot(index int, snapshot []byte) {
	if rf.killed() {
		return
	}

	rf.mu.Lock()
	defer rf.mu.Unlock()

	// 如果下标大于自身的提交，说明没被提交不能安装快照，如果自身快照点大于index说明不需要安装
	if index <= rf.log[0].Index || index > rf.commitIndex {
		return
	}

	DPrintf("节点%d: 执行Snapshot[%d]", rf.me, index)
	snapshotinfo := Log{
		Entry{
			Index:    index,
			Term:     rf.log.at(index).Term,
			Commmand: nil,
		},
	}
	rf.log = rf.log.cut(index+1, rf.log.last().Index+1) //深拷贝，左闭右开
	rf.log = append(snapshotinfo, rf.log...)
	BPrintf("[raft%d-Snapshot]:raft take snapshot at index%d", rf.me, index)
	// user persister to persist persistdata and SnapShot
	rf.persister.SaveStateAndSnapshot(rf.persistState(), snapshot)

}

// Installsnapshot to follower
func (rf *Raft) InstallSnapshot(args *InstallSnapshotArgs, reply *InstallSnapshotReply) {
	rf.mu.Lock()
	defer rf.mu.Unlock()

	if args.Term >= rf.currentTerm { // 过期RPC不处理
		rf.resetTimeOut()

		if args.Term > rf.currentTerm { //term比自己大，转为follower
			rf.stepToFollower(args.Term)
			rf.persist()
		}
		DPrintf("follower%d收到快照[%d,%d]，此时log长度：%d", rf.me, args.LastIncludedIndex, args.LastIncludedTerm, len(rf.log))

		if args.LastIncludedIndex > rf.commitIndex { //旧快照不处理request.LastIncludedIndex <= rf.commitIndex
			snapshotinfo := Log{Entry{
				Index:    args.LastIncludedIndex,
				Term:     args.LastIncludedTerm,
				Commmand: nil,
			}}
			rf.log = append(snapshotinfo, rf.log.cut(args.LastIncludedIndex+1, rf.log.last().Index+1)...)
			//持久化状态和快照
			rf.persister.SaveStateAndSnapshot(rf.persistState(), args.Data)
			rf.resetTimeOut()
			//将快照发给本节点的上层应用，让应用决定是否安装快照

			snapshotMsg := ApplyMsg{
				CommandValid:  false,
				SnapshotValid: true,
				Snapshot:      args.Data,
				SnapshotIndex: args.LastIncludedIndex,
				SnapshotTerm:  args.LastIncludedTerm,
			}

			rf.applyQueue = append(rf.applyQueue, snapshotMsg)

			//快照是比自己新的，更新自己状态，无条件服从leader
			//rf.lastApplied = args.LastIncludedIndex
			rf.commitIndex = args.LastIncludedIndex
			BPrintf("[raft%d-InstallSnapshot]:raft change snapindex=%d ", rf.me, rf.log[0].Index)
			BPrintf("[raft%d-InstallSnapshot]:raft change commitIndex=%d ", rf.me, rf.commitIndex)
			rf.applyCond.Signal()
		}
	}
	reply.Term = rf.currentTerm

}

// send , wait for reply and handle
func (rf *Raft) handelInstallSnapshot(server int, ISargs InstallSnapshotArgs) {
	ISreply := InstallSnapshotReply{}
	ok := rf.sendInstallSnapshot(server, &ISargs, &ISreply) // 一直阻塞直到得到回复

	if !ok {
		return
	}

	rf.mu.Lock()
	defer rf.mu.Unlock()

	// 收到投票回复的时候，自己的任期还没变（说明目前仍然是发出snapshot时的状态），则处理；
	// 否则不处理，因为收到该回复时自己的任期已经变化了
	if rf.currentTerm == ISargs.Term {
		if ISreply.Term < rf.currentTerm {
			return
		}

		if ISreply.Term > rf.currentTerm {
			rf.stepToFollower(ISreply.Term)
			rf.persist()
			return
		}

		rf.matchIndex[server] = max(rf.matchIndex[server], ISargs.LastIncludedIndex)
		rf.nextIndex[server] = max(rf.nextIndex[server], ISargs.LastIncludedIndex+1)
		BPrintf("[raft%d-handelInstallSnapshot]:installsnap sucess to peer%d, update[nextindex:%d]", rf.me, server, rf.nextIndex[server])
		rf.cherkCommitIndex()

	}

}

```






