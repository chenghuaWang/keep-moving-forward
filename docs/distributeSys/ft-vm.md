# The Design of a Practical System for Fault-Tolerant Virtual Machines

[go back to home](/)

[Paper link](https://pdos.csail.mit.edu/6.824/papers/vm-ft.pdf)

ACM SIGOPS Operating Systems Review, 2010, 44(4): 30-39.

Last Edit: Jan 19, 2023

---

# Introduction

是的，这节内容还是 FT(fault-tolerance)。现在做 FT 主要有两种手段[2]：

1. State transfer

Primary用来执行所有的任务，Primary发送该机器的所有状态(所有的内存变动，所有的磁盘变动，等)给Replica。State Transfer 虽然听起来非常的简单(实际上做起来也是的，相对于Replicated state machine)，但是需要占用非常大量的网络带宽来实现。

尽管如此(占用大量的网络资源，导致传输缓慢)，State Transfer是对多处理器友好的一种方式，而Replicated state machine则不是。

2. Replicated state machine

Client 向 Primary 发送操作和数据(inputs)。Primary把这些操作和数据发送给Replicas，Replicas和Primary都会执行这些指令，都会受到这些数据(inputs)，所有的指令都以同样的顺序执行，只要Primary和Replicas初始的状态是一致的，那么二者就一直保持着同步(也意味着deterministic)。

在并行化的应用中，State transfer 是首选，毕竟在并行的时候，并行顺序对于Replicated state machine来说是异常难同步的。

主从复制时的挑战[2]：

1. 要复制什么状态
2. primary需要等待backup吗
3. 什么时候需要切换到backup
4. 切换的时候异常情况是否能看到
5. 如何提高创建新backup的速度

---

VM-FT使用的就是Replicated state machine的方法。为了能够捕获数据，并且做出一些软件中断，Primary和Backup都是在VM上运行的，由hypervisor来管理他们。

# Arch

VM-FT 总体的结构较为简单，论文中是一主一从的结构。VM-FT主要依赖于 Deterministic replay(确定性重放)。VM-FT 通过确定性重放来产生相关的日志条目，但不将日志写入磁盘，而是通过 logging channel 发送给backup(这里指备用机)。backup实时重放日志项。

因为一切都是在 logging channel 上做同步的。为了容错，必须在 logging channel 上实现严格的容错协议，有以下要求：

1. Primary直到backup接收并确认了和输出相关的日志的时候，才发送输出给外界。(这样做的目的是，只要backup收到了所有的日志条目，即使primary宕机了，backup仍能够重放到客户端最后看到的状态。)

> Primary和backup的数据都是10。现在客户端发送increase请求，Primary+1并回复给client 11，之后马上宕机了，更糟糕的是Primary发给backup的+1操作也丢包了。这时候backup还是10，并接管了primary的工作，client再次请求+1，又会收到11的回复。这就产生了矛盾。

2. 如果备机在主机故障后接管，备机将以和主机已经向外界发送的输出完全一致的方式继续运行。

为了确保一次只有一个虚拟机成为主机，避免出现brain split，VMware 在共享存储上执行test and set指令("OSTEP"中提到过)。

为了保证一定的性能，VM-FT决定在Primary没收到Ack以前，可以继续往下执行，但是这样会拉大和backup之间的距离。所以文中用一定的方法来限制这个gap：如果backup在需要读取下一个日志条目时遇到空日志条目，则停止执行，直到有新的日志条目可用。因为backup没有与外部通信，因此此暂停不会影响clients。相同的，Primary如果在需要写入日志条目时遇到一个满的buffer(**虚拟机管理程序维护了一个大的日志缓冲(log buffer)，保存主机和备机的日志。主机会产生日志项到日志缓冲，备机从日志缓冲消费日志。**)，则停止执行，直到日志条目被清除为止，显然我们不希望Primary与backup之间状态差距太大，一般来说Primary速度更快。

# 缺陷

仅支持单处理器，多核对于Replicated state machine是极其不确定的。

# Reference

[1] Scales D J, Nelson M, Venkitachalam G. The design of a practical system for fault-tolerant virtual machines[J]. ACM SIGOPS Operating Systems Review, 2010, 44(4): 30-39.

[2] [MIT6.824 Notes. l-vm-tf.txt](https://pdos.csail.mit.edu/6.824/notes/l-vm-ft.txt)