---
layout: post
title: "协程并发模型及使用感受"
category: other
tags: gevent
comments: true
---


协程可以简单理解为更轻量的线程，但有很多显著的不同：

* 不是OS级别的调度单元，通常是编程语言或库实现
* 可能需要应用层自己切换
* 由于切换点是可控制的，所以对于CPU资源是非抢占式的
* 通常用于有大量阻塞操作的应用，例如大量IO

协程与actor模式的实现有一定关系。由于协程本身是应用级的并发调度单元，所以理论上可以大量创建。在协程之上做队列及通信包装，即可得到一个actor框架，例如[python-actor](https://github.com/jrydberg/python-actors)

最近1年做了一个python项目。这个项目中利用gevent wsgi对外提供HTTP API，使用gevent greelet来支撑上层应用的开发。当可以使用协程后，编程模型会得到一定简化，例如相对于传统线程池+队列的并发实现，协程可以抛弃这个模型，直接一个协程对应于一个并发任务，例如网络服务中一个协程对应一个socket fd。

但是python毕竟是单核的，这个项目内部虽然有大量IO请求，但随着业务增长，CPU很快就到达了瓶颈。后来改造为多进程结构，将业务单元分散到各个worker进程上。

python gevent中的协议切换是自动的，在遇到阻塞操作后gevent会自动挂起当前协程，并切换到其他需要激活的协程。阻塞操作完成，对应的协程就会处于待激活状态。

在这个项目过程中，我发现协程也存在很多陷阱。
<!-- more -->
## 协程的陷阱

* 死循环

普通的死循环很难遇到，间接的死循环一旦发生，就会一直占用CPU资源，导致其他协程被饿死。

* 留意非协程化的阻塞接口

gevent中通常会将python内置的各种阻塞操作green化，也就是我这里说的协程化，例如socket IO接口、time.sleep、各种锁等待。如果在系统中引入一个不能被协程化的库，例如[MySQL-python](https://github.com/farcepest/MySQLdb1)。当协程被阻塞在这种库的接口时，协程不能被切走，而是等到python内线程的抢占式切换，实际上对于gevent的协程调度其总计可用的CPU就不是100%了。在压力较大的情况下，协程就可能出现延迟调度。意思是在协程阻塞操作完成后，在负载较小的情况下，该协程会立即得到切换。

这里有一个小技巧，可以写一个time.sleep延时的协程，检查真实的延时情况和time.sleep的延时参数相差多少，就可以衡量整个系统中协程切换的延时情况。

* 注意不同角色协程的CPU资源分配

这个问题本质上类似于在基于线程的应用中，需要为不同角色的线程设定不同的优先级。在多核程序中由于总的CPU资源比较多，所以一般也不会遇到需要分配不同优先级的情况。但在基于协程的单核程序中，由于单核CPU资源很快就会被压榨到80-90%，所以就需要关注不同角色协程的优先级。

例如，系统中有用于服务HTTP API的协程集，有用于做耗时任务的协程集。耗时任务正常情况下可能需要分钟级，所以做任务的协程就算慢几秒也没什么关系。但是对外提供API的协程，本身API时延就在毫秒到秒级，如果晚几秒到几十秒，对上游系统或者用于就会造成不良的影响，表现为服务质量差。

但是通常协程库是没有设定优先级的功能的。所以这个时候就要从应用层解决。例如前面的耗时任务例子，一般情况下，为了编程简单，我们会为每一个任务分配一个协程去做。由于所有协程优先级相同，大家被切换的机会是均等的，那么当任务增多后，API相关的协程获得的切换机会更少，影响服务质量。所以这个时候，就会创建一个用于完成耗时任务的协程池，以限制耗时任务占用的总协程数量。这就又回到了基于线程的并发模型中。

* 留意协程切换

在gevent这种协程切换不需要程序员显示操作的协程库中，程序员会慢慢忘掉自己是在协程环境下编程。前面的例子中，我们创建了一个协程池去限制耗时任务可用的协程数量。在实际项目中可能会对调度做一些包装，让应用层只关注自己的业务代码。那么，在业务代码中，对于一些需要重试的失败操作，我sleep一段较长的时间也很合情理吧。这个时候如果由于外部依赖服务异常，而导致部分业务协程失败，处于sleep中。这个时候，协程池内有限的协程都被挂起了。导致很多本来可以获得CPU资源的任务无法得到消费，导致整个系统的吞吐量下降。

## 总结

协程会在低CPU系统中获得不少易于编程的好处，但是当系统总CPU上去后就需要付出等价于甚至大于多线程编程中的代价。


