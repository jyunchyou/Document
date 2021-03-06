## RFID智能仓库可视化部分设计
在系统的初期我们想要实现万物实时流动的可视化系统，在智能工厂改造中做到如下三点:


* 准确性：现场容器的信息、位置、状态等要与信息系统做到数据一致

* 实时性：任意时刻的入库出库对于我们的系统都能感知

* 可视化：变化中的数据能够体现在可视化界面中，满足多样化展示需求

为了满足上述需求，在物理层，我们采用了rfid射频技术，在被感应的容器上安装标签，同时每个库位安装上天线，读写器通过天线不断读取标签，再将读取数据发送到软件层。

* 1.0 诞生

1.0是一个流式消费的场景，大量读写器通过socket与客户端进行交互，一个线程分管一个socket。启动时加载所有读写器ip,port。由单独一个线程负责对所有读写器发起连接。连接成功后交予分管线程，每间隔一段时间发起寻卡指令，sleep几秒后（睡眠时间保证读写器能收集到所有端口的数据），将收到的队列中的数据发往消息队列。

扩大到整个系统来看，上面与读写器交互的部分单独作为一个服务，整个系统大致分为六个服务。分别是
- 读写器服务
- 业务中心服务
- 灯控服务
- 管理平台服务
- 可视化服务
- 手持机服务
- 监控服务

在消息队列的消费端是业务中心服务，负责出入库判定，（在实际开发中，出入库判定与业务判定耦合在了一起），在出入库判定结束后判断消息来自哪个库房哪个库位或工位，接着会做库房的业务判断，然后将消息推往灯控服务、wms服务，以及其它信息系统。

每次收到消息队列上的最新感应消息，业务中心服务会将容器的入库状态以及最后一次访问时间记录在redis缓存上，由离库判定线程轮询这部分状态数据，当它的最后一次访问时间距离当前时间超过一定时间，对它做出库处理。


我们在试点第一个库房的时候，经常出现误读问题，同一个容器在两个库位之间来回横跳的情况。当时不管是测试手段还是现场的复杂度不足以我们发现除此之外更多的问题，1.0架构中的准确性没有达到我们预期。
随着其它库房的接入，数据量开始增加，开始出现了消息堆积，处理速度不足的问题。于是2.0着重解决上述问题。

* 2.0 优化

首先是误读问题，为了避免同一标签被多个天线读到，或一个天线读取了多种标签数据，我们上游做了统计处理，先是对次数进行hash去重，避免一轮寻卡时间内多余的重复数据打到下游，再在一轮寻卡的结果内进行读取次数统计，读取次数越多代表信号越强，保留每个天线读取次数最多的标签，且标签被当前天线读取次数最多，否则进行误读丢弃。

误读统计用到的数据结构：

![image](https://github.com/jyunchyou/Document/blob/main/2.png)

消费端我们将消息处理线程扩大到200个线程，部署增加到两台，加快了消息处理速度，并且尽量将数据缓存到redis,减少mysql访问次数，mysql的cpu占用率从500%+降到100%+，缩短了每个消息占用的阻塞时间。离库判定线程增加到20个。

经过这次优化，消息处理速度有了质的提升，但由于针对单台读写器读取到的数据做了统计，误读的问题仍然存在于不同读写器之间，有可能当前库位上的标签被别的读写器读到，并且统计的时间区间较短，还不能达到我们想要的100%显示准确率。

* 3.0

回到设计之初，系统一直有个关于一致性的缺陷。容器状态我们缓存在redis, 而业务参照的是数据库,redis和mysql之间很容易出现数据不一致的问题，我们用了新的线程模型来进行状态的同步，新的消息处理模型采用SEDA的架构如下图所示：

![image](https://github.com/jyunchyou/Document/blob/main/1.png)

SEDA架构将服务器的处理划分多个阶段，每个阶段通过队列进行异步通信。它的优势是在低io消耗场景，通过减少线程切换提高系统吞吐量。
在面向客户端请求的场景里，线程需要大量访问数据库，通过增加线程数，从而提升cpu占用率。而在面向消息的场景，大部分消息能被缓存过滤，io消耗占少数，减少线程数更有利于提高吞吐量。
同时，我们数据同步的思路发生了转变，从根据容器id求余获取锁变为：线程去处理对应容器id的消息。
单机系统瓶颈从计算资源变为内存资源。