		Lamport发表该论文于1978年的"Communication of ACM"，并于2000年获得了首届PODC最具影响力论文奖，于2007年获得了ACM SIGOPS Hall of Fame Award。本文包含了两个重要的想法，每个都成为了主导分布式计算领域研究十多年甚至更长时间的重要课题。***经典必读***

### 文章首先定义说明不同进程之间的事件之间的关系为偏序关系。

*定义*→为偏序关系关系需要满足一下三个条件：

- （1）如果事件a、b在一个进程中，a发生于b之前，那么a→b；
- （2）如果a是发送一条消息的事件，而b是接收这条消息的事件，那么a→b；
- （3）假如a→b成立，且b→c成立，那么a→c成立。
- 如果两个事件a、b不满足a→b或者b→a，那么认为a、b两个事件是*并发*的。我们假设a→a是不成立的（一个事件发生于自己之前显得没有什么意义），那么→是不具备自反性的偏序关系。

<img src="images/Time, Clocks and the Ordering of Events in a Distributed/Figure1.png" alt="Fugure1" style="zoom:50%;" />

Fig1中：P,Q,R为进程，Pi,Qi,Ri分别为对应进程中的事件。每条垂直线上的进程事件从下往上执行。其中如果两个事件中及有因果关系的话，就可以定义这两个事件可以定义为由事件a导致了事件b的发生(a->b)或者可以理解为图中的(p1->q2)，否则，则定义两个事件为并行发生的。如p3和q3，虽然图上画的看着是p3要晚于q3，但是进程P在其事件p4被进程Q中的q5引起之前，进程P是不知道q3的发生时间的。

### 逻辑时钟

> 逻辑时钟算法和向量时钟算法见有道云笔记

逻辑时钟不涉及物理时钟的关系，是从逻辑上说明的关于不同进程之间顺序关系的一种概念。

我们为进程Pi定义一个时钟Ci为进程中的每个事件分配一个编号**Ci<a>**。系统的全局时钟为**C**，对任意事件b，它的时间为C<b>，如果b是进程j中的事件，那么C<b>=Cj<b>。目前为止，对Ci我们没有引入任何物理时钟的依赖，所以我们可以认为Ci是逻辑上的时钟，和物理时钟无关。它可以采用计数器的方式实现而不需要任何计时机制。

**Clock Condition.\* For any event a and b:**   **if a → b then C<a> < C<b>**

这个命题的***逆命题并不成立***。因为就像Fig1所示那样对于p3和q3而言，并没有指出他们两个事件之间的顺序，会当成两者并发执行。但如果实际上的处理时间并不一定相同，所以如果C<q3> < C<p3>，也不能推出来q3->p3。

<img src="images/Time, Clocks and the Ordering of Events in a Distributed/Figure2.png" alt="Figure2" style="zoom:50%;" /><img src="images/Time, Clocks and the Ordering of Events in a Distributed/Figure3.png" alt="Figure3" style="zoom:50%;margin-left:120px" />

Fig2中的虚线表示各自进程中表示的时间是相同的时候。而Fig3是将Fig2中的虚线拉直后的状态。这样可以更可观一些。



