# MapReduce

OSDI'04: 6th Symposium on Operating Systems Design and Implementation

[go back to home](/)

[Paper link](https://pdos.csail.mit.edu/6.824/papers/mapreduce.pdf)

---

# 什么是 MapReduce

在MapReduce原文中是这么说的 

> MapReduce is a programming model and an associated implementation for processing and generating large data sets. [1]

这是一个较为简单的处理大型问题的分布式编程模型。其产生的原因是因为 google 想要让普通的程序员也能够通过一套简单的编程模型来编写分布式应用程序，以此来处理 google 内部的大数据。然后著名的 Jeff Dean 和 Sanjay Ghemawat 出马搞定了这个问题([在知乎上关于Jeff Dean的轶事:-)](https://www.zhihu.com/question/22081653))。MapReduce 在 google 内部运行很长的时间并且处理了很多业务，但是目前也败下阵来，具体原因参见 MapReduce 缺陷章节(MR早在2004年就出来了，其实是古早的技术了，其没有在性能上做文章，主要在可扩展性上)。

## MapReduce 的设计框架和基本流程

MR实际上是一个非常简单的框架，思路也非常的直白。MapReduce 背后的核心思想是将数据集映射到一个 `<key, value>` pair的集合中，然后使用相同的键对所有pair进行整合。 整体概念很简单，但是如果你设想下下面的情况，MR实际上非常有用: 基本上所有的数据都能被映射到一个 `<key, value>` 对中，并且key和value可以是任意类型。

在论文中MR使用的样例是在超大的文件里做单词统计。这个框架还可以用来做：

1. 分布式的排序
2. 分布式的搜索
3. Web链接图遍历
4. ...

---

<div align="center"> 
<img src="./imgs/MapReduceArch.png" width = "80%"/>
<br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">Fig 1. MapReduce Arch(ref [1])</div>
</div>

MR 中的Map和Reduce是从函数式编程语言(LISP)中借鉴过来的一个概念。程序员只需要编写Map和Reduce两个函数就能够对某个任务并发处理。在 Fig 1 中，MR主要的流程分为6个部分

1. 用户程序的MR首先将输入文件分成 M 块，每块通常为 16 MB到 64 MB(可由用户通过参数控制)。 然后，它会在一组机器上启动该程序的多个副本。

2. 这些副本中有一个控制程序(Master)。其余的程序(Worker)由控制程序(Master)来分配。有 M 个 Map 任务和 R 个 Reduce 任务要分配。Master 挑选空闲的 Worker 并为每个机器分配一个 Map 任务或 Reduce 任务。

3. 分配了 map 任务的 worker 读取相应的内容(拆分后的文件块)(设想下，如果这里的文件读取需要从数据仓库中拿出，那么整个框架的计算能力极大的被网络所限制了)。它从输入数据中解析出`<key, value>`对，并将每一对传递给用户定义的 Map 函数。 Map 函数产生的中间`<key, value>`对被缓存在内存中。

4. 缓存在机器中的中间`<key, value>`对周期性地被写入本地磁盘，由分区函数划分为 R 个区域。这些缓冲对在本地磁盘上的位置被传回 Master，Master 负责将这些位置转发给 Reduce Worker。

5. 当 Master 通知 Reduce Worker 有关这些位置时，它会使用RPC从 Map Worker 的本地磁盘读取缓冲数据。当 Reduce Worker 读取所有中间数据时，它会按中间键对其进行排序，以便将所有出现的相同键组合在一起。排序是因为通常许多不同的键映射到同一个 Reduce 任务。如果中间数据量太大而无法放入内存，则使用外部排序。

6. Reduce Worker 迭代排序的中间数据，对于遇到的每个唯一中间键，它将键和相应的中间值集传递给用户的 Reduce 函数。Reduce 函数的输出到此 Reduce 分区的最终输出文件。

7. 当所有的 Map 任务和 Reduce 任务都完成后，Master唤醒用户程序。此时用户程序中的 MR 调用返回给用户代码

因为网络上的数据交换是非常缓慢的，作者在分发数据的时候尽量保证数据就在这台计算服务器上，而不需要传输数据。在原文中是这样说的：

> We conserve network bandwidth by taking advantage of the fact that the input data (managed by GFS) is stored on the local disks of the
machines that make up our cluster.


# MapReduce 的使用范例

论文中，以词频统计为例子，如论文中的伪代码：

```
map(String key, String value):
    // key: document name
    // value: document contents
    for each word w in value:
        EmitIntermediate(w, "1");
```

```
reduce(String key, Iterator values):
    // key: a word
    // values: a list of counts
    int result = 0;
    for each v in values:
        result += ParseInt(v);
        Emit(AsString(result));
```

# MR Fault Tolerance

Fault Tolerance 在分布式程序中是非常重要的。MR 主要的贡献点其实就在 scalability 和 fault tolerance。

## Worker Failure

Master 会定期的 ping Worker 服务器，如果不通，那么这个 Worker 被设置成 idle，分配给这台机器的任务会被分发给其他的机器。

## Master Failure

Master 因为只有少量的机器，所以发生错误的可能性非常的小。可以通过存储 Master 的状态为 checkpoints 来进行错误时回退和恢复。

# MR 缺陷

MR 的缺陷实际上在 Google I/O 上已经说明了，如下:

> Today at Google I/O, we are demonstrating Google Cloud Dataflow for the first time. Cloud Dataflow is a fully managed service for creating data pipelines that ingest, transform and analyze data in both batch and streaming modes. Cloud Dataflow is a successor to MapReduce, and is based on our internal technologies like Flume and MillWheel.[2]

1. MR是一个基于 batch mode 的框架。
2. 用 MR 写复杂的分析 pipeline 太麻烦。

在很多情况下，一次 MR 执行是没法把事情做完的，需要很多个 MR 任务互相组合，这就是 MR pipeline. 在数据流复杂的分析任务中，设计好的 pipeline 达到最高运行效率很困难。

MR 最大的贡献我认为就是 scalability 和 fault tolerance。在 MillWheel 里，结果可以随着数据流的输入实时的显示出来，而不是到数据流结束时才输出一个结果，而且这样中间结果不需要保存下来。

# MR Lab in MIT6.824

[Check the lab-mr page here](https://pdos.csail.mit.edu/6.824/labs/lab-mr.html)

在本实验中，我们将构建一个 MapReduce 系统。 我们将实现一个调用应用程序 Map 和 Reduce 函数并处理读写文件的程序，以及一个将任务分发给 Worker 并处理失败的 Worker 的进程(Master)。在 MR 实验中，使用 Golang 来编码实现。

我的实现在 WSL Ubuntu20.04 上 go版本 1.18。

这次的要求貌似比以前更难了，这次需要 Worker 主动向 Master 请求任务，当任务超时后，需要 Master 来重新分配这个任务。

总体思路其实也非常的简单，worker 和 coordinator 的交互都围绕着 RPC 展开，故先从 RPC 的定义开始。因为深受事件轮询编程模式的'荼毒'，我把 worker 和 coordinator 之间的交互过程以事件的形式来进行处理。首先定义事件的类型。

```go
// src/mr/rpc.go

type TaskTp = int
const (
	TpRequireTask = iota
	TpMapTaskDone
	TpReduceTaskDone
	TpSendMapTask
	TpSendReduceTask
	TpTaskAllDone
	TpWait
)
```

再来考虑 RPC 需要在 worker 和 coordinator 中传递什么信息？不论何种信息，首先都需要一个时间戳来记录 require 和 reply 对。其次每个 worker 都需要知道一共有多少个 Map 和 Reduce 任务。每个 require 需要跟上当前 worker 的任务编号。当然，因为是事件驱动的角度编写的，所以每条 require 和 reply 都需要有一个事件类型记录。

```go
// src/mr/rpc.go

type RequireMsg struct {
	Stamp   int64
	MsgFlag TaskTp
	TaskID  int
}

type ReplyMsg struct {
	Stamp            int64
	MsgFlag          TaskTp
	TaskID           int
	NumReduceWorkers int
	NumMapWorkers    int
	Content          string
}
```

---

对于 coordinator，其作用像是一个状态机，需要维护所有的 worker 的状态，并能做到回退(Fault Tolerance)。Coordinator 的定义如下

```go
// src/mr/coordinator.go

type Coordinator struct {
	nMapWorkers       int
	nReduceWorkers    int
	nMapWorking       int
	nReduceWorking    int
	MapWorkerState    []int64
	ReduceWorkerState []int64
	FilesContent      []string
	MapWorkerPool     chan int
	ReduceWorkerPool  chan int
	IsDone            bool
	GlobalLock        *sync.Cond // to make sure the rpc visit is atomic.
}
```

coordinator 需要处理 RPC 的请求，对于不同的事件进行反应。因为需要超时处理，所以在这里，每一个 worker 都会有一个协程来进行相应的计时和 id 回收。

```go
// src/mr/coordinator.go

func (c *Coordinator) ProcessEvents(args *RequireMsg, reply *ReplyMsg) error {
	c.GlobalLock.L.Lock()
	tmpBool := c.IsDone
	c.GlobalLock.L.Unlock()
	if tmpBool {
		reply.MsgFlag = TpTaskAllDone
		reply.Stamp = args.Stamp
		return nil
	}
	switch args.MsgFlag {
	case TpMapTaskDone:
		c.GlobalLock.L.Lock()
		if c.MapWorkerState[args.TaskID] == args.Stamp {
			c.MapWorkerState[args.TaskID] = 1
			c.nMapWorking--
		}
		c.GlobalLock.L.Unlock()
	case TpReduceTaskDone:
		c.GlobalLock.L.Lock()
		if c.ReduceWorkerState[args.TaskID] == args.Stamp {
			c.ReduceWorkerState[args.TaskID] = 1
			c.nReduceWorking--
		}
		if c.nReduceWorking == 0 {
			c.IsDone = true
		}
		c.GlobalLock.L.Unlock()
	case TpRequireTask:
		if len(c.MapWorkerPool) > 0 {
			// State 1. Send Map Task to worker.
			reply.Stamp = args.Stamp
			reply.TaskID = <-c.MapWorkerPool
			reply.MsgFlag = TpSendMapTask
			reply.NumMapWorkers = c.nMapWorkers
			reply.NumReduceWorkers = c.nReduceWorkers
			reply.Content = c.FilesContent[reply.TaskID]
			c.GlobalLock.L.Lock()
			c.MapWorkerState[reply.TaskID] = args.Stamp
			c.GlobalLock.L.Unlock()
			go func(id int) {
				time.Sleep(10 * time.Second)
				c.GlobalLock.L.Lock()
				if c.MapWorkerState[id] != 1 {
					// run out of 10 secs. recycle this id to id pool.
					c.MapWorkerPool <- id
				}
				c.GlobalLock.L.Unlock()
			}(reply.TaskID)
			return nil
		} else {
			c.GlobalLock.L.Lock()
			nMapOnWorking := c.nMapWorking
			nReduceOnWorking := c.nReduceWorking
			c.GlobalLock.L.Unlock()
			// State 2. All map worker is on working. wait.
			if nMapOnWorking != 0 {
				reply.Stamp = args.Stamp
				reply.TaskID = -1
				reply.MsgFlag = TpWait
				reply.Content = "Just Wait !!! Map Worker !!!"
				reply.NumMapWorkers = c.nMapWorkers
				reply.NumReduceWorkers = c.nReduceWorkers
			} else {
				// State 3. Map is Done. Send Reduce task to works.
				if len(c.ReduceWorkerPool) > 0 {
					reply.Stamp = args.Stamp
					reply.TaskID = <-c.ReduceWorkerPool
					reply.MsgFlag = TpSendReduceTask
					reply.Content = "Reduce no need to handle this"
					reply.NumMapWorkers = c.nMapWorkers
					reply.NumReduceWorkers = c.nReduceWorkers
					c.GlobalLock.L.Lock()
					c.ReduceWorkerState[reply.TaskID] = args.Stamp
					c.GlobalLock.L.Unlock()
					go func(id int) {
						time.Sleep(10 * time.Second)
						c.GlobalLock.L.Lock()
						if c.ReduceWorkerState[id] != 1 {
							// run out of 10 secs. resolved this id to id pool.
							c.ReduceWorkerPool <- id
						}
						c.GlobalLock.L.Unlock()
					}(reply.TaskID)
				} else {
					// State 4. All reduce worker is on working. wait.
					if nReduceOnWorking != 0 {
						reply.Stamp = args.Stamp
						reply.TaskID = -1
						reply.MsgFlag = TpWait
						reply.Content = "Just Wait !!! Reduce Worker !!!"
						reply.NumMapWorkers = c.nMapWorkers
						reply.NumReduceWorkers = c.nReduceWorkers
					}
				}
			}
		}
	default:
		return nil
	}
	return nil
}
```

同样的，在 worker 中的流程也是不断的循环产生事件，处理事件

```go
// src/mr/worker.go

func Worker(mapf func(string, string) []KeyValue,
	reducef func(string, []string) string) {
	for true {
		ThisStamp := time.Now().Unix()
		Req := RequireMsg{}
		Req.Stamp = ThisStamp
		Req.MsgFlag = TpRequireTask
		Req.TaskID = -1
		Rep := ReplyMsg{}
		ok := call("Coordinator.ProcessEvents", &Req, &Rep)
		if ok && Rep.Stamp == Req.Stamp {
			switch Rep.MsgFlag {
			case TpSendMapTask:
				doMap(mapf, &Rep)
			case TpSendReduceTask:
				doReduce(reducef, &Rep)
			case TpWait:
				time.Sleep(time.Second)
			case TpTaskAllDone:
				return
			default:
				return
			}
		}
	}
	return
}
```

`doMap(mapf, &Rep)` 和 `doReduce(reducef, &Rep)` 就是非常简单的逻辑编写了。Map需要把文件分拆到`N=numReduce`块中，总共产生`N * M(reduce num, map num)`块文件。Reduce worker从自己对应的编号中取出`M`块进行排序和合并。

---

结果顺利通过，没有过多的延迟(因为10s的判断)产生。

<div align="center"> 
<img src="./imgs/test-mr-pass.png" width = "40%"/>
<br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">Fig 2. test mr</div>
</div>

# Reference

[1] "MapReduce: Simplified Data Processing on Large Clusters" googleusercontent.com.

[2] [Google cloud blog about MapReduce's successor](https://cloudplatform.googleblog.com/2014/06/reimagining-developer-productivity-and-data-analytics-in-the-cloud-news-from-google-io.html)
