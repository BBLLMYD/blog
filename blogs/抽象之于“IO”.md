<br>

“I/O”词条在维基百科可以搜索到如下定义

>  I/O（英语：Input/Output），即输入／输出，通常指数据在存储器（内部和外部）或其他周边设备之间的输入和输出，是信息处理系统（例如计算机）与外部世界（可能是人类或另一信息处理系统）之间的通信。输入是系统接收的信号或数据，输出则是从其发送的信号或数据。该术语也可以用作行动的一部分；到“运行I/O”是运行输入或输出的操作。

“I/O”更像是一个动名词，简单说I/O的出现是为了实现信息交换和传输，是计算机实现通信的基础。进行实现通信的目的的过程中以及不同的场景下出现了不同的机制和模式。
<br>
因为我们编写的应用程序如需和其他程序进行通信，首先程序要运行，就必然脱离不了操作系统，是同一个OS下的不同进程间通信，还是不同机房之间的通信，都有着同步/异步，阻塞/非阻塞的类型机制区别。**无论是I/O设备还是I/O接口，操作系统提供了通信的基础设施**。
所以不管是磁盘I/O还是网络I/O，如果我们单纯的了解我们应用程序内部的线程状态和分配策略，
而不对OS和App之间的交互基础有足够多的了解，是很容易迷茫的。先了解关于OS与用户程序之间的通信的种种模式，
才能根据我们的业务场景选型或实现更适合的通信组件。

下面是我们就IO相关部分内容按如下顺序分别做一些展开讨论：

- [关于“IO”模型分类](https://gitehub.com/BBLLMYD/blog/blob/mastaolunter/blogs/%E6%8A%BD%E8%B1%A1%E4%B9%8B%E4%BA%8E%E2%80%9CIO%E2%80%9D.md#1%E5%85%B3%E4%BA%8Eio%E6%A8%A1%E5%9E%8B%E5%88%86%E7%B1%BB)
- [关于“多路复用”几种实现](https://github.com/BBLLMYD/blog/blob/master/blogs/%E6%8A%BD%E8%B1%A1%E4%B9%8B%E4%BA%8E%E2%80%9CIO%E2%80%9D.md#2%E5%85%B3%E4%BA%8E%E5%A4%9A%E8%B7%AF%E5%A4%8D%E7%94%A8%E5%87%A0%E7%A7%8D%E5%AE%9E%E7%8E%B0)
- [关于“sendfile”、"mmap"的特性及应用场景](https://github.com/BBLLMYD/blog/blob/master/blogs/%E6%8A%BD%E8%B1%A1%E4%B9%8B%E4%BA%8E%E2%80%9CIO%E2%80%9D.md#3%E5%85%B3%E4%BA%8Esendfilemmapdma%E7%9A%84%E7%89%B9%E6%80%A7%E5%8F%8A%E5%BA%94%E7%94%A8%E5%9C%BA%E6%99%AF)
- [关于应用层对上述技术的实际应用](https://github.com/BBLLMYD/blog/blob/master/blogs/%E6%8A%BD%E8%B1%A1%E4%B9%8B%E4%BA%8E%E2%80%9CIO%E2%80%9D.md#4%E5%85%B3%E4%BA%8E%E5%BA%94%E7%94%A8%E5%B1%82%E5%AF%B9%E4%B8%8A%E8%BF%B0%E6%8A%80%E6%9C%AF%E7%9A%84%E5%AE%9E%E9%99%85%E5%BA%94%E7%94%A8)

尽量还是从抽象的角度来它们的特性、区别、联系及实际应用。


* * *


### 1.关于“IO”模型分类

<br>
主要先就讨论Linux下的关于网络I/O的内容展开做一些讨论，常被提及的同步/异步，阻塞/非阻塞网上有很多解释，我觉得主要是因为大家在解释这个问题的前置设定可能不一致，导致很多说法好像都对但看起来又好像不是一回事，明确了上下文来谈这两个问题才会更清晰。
I/O的流程可分为两步：
<br>

* 1.等待数据准备好
* 2.从内核向进程copy数据

**我们常讲的同步/异步指的其实是第二个动作，阻塞/非阻塞指的是第一个动作。**<br>
各个I/O模型也是在两个步骤的实现方式不同而区分的，大致是下图的思路。
<br>
<div align=center><img src="https://github.com/BBLLMYD/blog/blob/master/images/03/3-7-7-0.png?raw=true" width="937"></div>

<br>
其中的信号驱动I/O和异步I/O实际上应用的比较少，它们的实现方式相对和操作系统更耦合而不可控是很大一部分原因。
<br>

<br>
实际上第二个copy数据行为并不是一个动作，而是一系列动作，关于数据在内核缓冲区和用户缓冲区甚至DMA和protocol engine之间的拷贝，涉及几次上下文切换才可以完成。虽然看起来很麻烦，但由于内核缓冲区在核心位置起的重要作用让这些动作在性能上也并不会那么慢，但是当需要传输的数据远远大于内核缓冲区的大小时，内核缓冲区就会成为瓶颈，此时zerocopy技术又成为了I/O密集型应用的这种场景下一个好的解决方案。但是事情总是相对的，zerocopy虽然提升了性能免去了一些上下文的切换成本，但是少了一层用户态控制，可能会导致传输数据相对不可控。

<br>
再来分别看一下这5种Linux网络I/O模型

<br>

<br>

**1.同步阻塞I/O**
<br><br>
Linux中默认的socket就是阻塞式I/O，也就是上述两个阶段都是阻塞的方式来进行，
对进程的线程资源很不友好，
当大量连接时容易将线程资源耗尽无法处理新的连接。但是系统调用次数较少，在流量不大且平稳的时候比较适用。
<br>
<div align=center><img src="https://github.com/BBLLMYD/blog/blob/master/images/03/3-1.png?raw=true" width="444"></div>
<div align=center>同步阻塞I/O过程</div>
<br>


**2.同步非阻塞I/O**
<br><br>
和上一个同步阻塞I/O相比，同步非阻塞I/O在第一阶段的阻塞等待变成轮询的方式，
相对之下避免了一下线程消耗，但是系统调用次数过多，CPU消耗也比较明显，这种模型实际应用也比较少。
<br>
<div align=center><img src="https://github.com/BBLLMYD/blog/blob/master/images/03/3-2.png?raw=true" width="444"></div>
<div align=center>同步非阻塞I/O</div>
<br>

**3.多路复用I/O模型**
<br><br>
多路复用建立在系统提供的事件分离函数select，poll，epoll之上，先更新select的socket监控列表，然后等待函数返回（此过程是阻塞的），关于这几个系统函数实现方式的特性的不同，也是适用不同的场景下。但是多路复用I/O在上层的实际应用很多，像Java的NI/O，Redis以及Netty通信框架都采用了这种模型，不过像Netty框架在多路复用的基础上又做了一下zerocopy等一些更细致的有针对性的优化方案。<br>
<br>
<div align=center><img src="https://github.com/BBLLMYD/blog/blob/master/images/03/3-3.png?raw=true" width="444"></div>
<div align=center>多路复用I/O</div>
<br>

**4.信号驱动I/O**
<br><br>
应用进程使用 sigactI/On 系统调用，内核立即返回，应用进程可以继续执行，也就是第一阶段是不阻塞的，等待系统向应用进程发送 SIGI/O信号，再来同步的向进程拷贝数据。CPU 利用率更高，但是过于依赖OS能力，在大量I/O操作的情况下可能造成信号队列溢出导致信号丢失，造成严重后果，所以实际应用也很少。
<br>
<div align=center><img src="https://github.com/BBLLMYD/blog/blob/master/images/03/3-4.png?raw=true" width="444"></div>
<div align=center>信号驱动I/O</div>
<br>

**5.异步I/O**
<br><br>
两个阶段都是不阻塞，第二阶段系统直接向进程通知I/O已经完成，此时进程可以直接使用数据。异步模型效率较高，但是还未足够成熟，同时也过于依赖操作系统，实际应用的也并不多。
<br>
<div align=center><img src="https://github.com/BBLLMYD/blog/blob/master/images/03/3-5.png?raw=true" width="444"></div>
<div align=center>异步I/O</div>
<br>

**为了低耦合，更可控，异步I/O和信号驱动I/O模型应用并不多见，更常见的是向对更灵活的多路复用和传统I/O模型。**
<br>

**总体比较**
<br><br>
可以比较直观的发现在两个阶段中，各个模型分别的处理方式和阻塞情况
<br>
<div align=center><img src="https://github.com/BBLLMYD/blog/blob/master/images/03/3-9.png?raw=true" width="555"></div>
<div align=center>比较</div>
<br>


为了避免我们的应用程序由于I/O导致线上故障，应用中的I/O操作最好用线程池隔离限制，避免由此引起整个进程的全局故障。




* * *

### 2.关于“多路复用”几种实现

上述提到的多路复用技术，是基于操作系统提供的几个function的实现，
select/poll/epoll做为实现I/O多路复用的手段，出现的时序是select->poll->epoll
<br>
select 和 poll 的功能现实上大致相同，在一些实现细节上有所不同
<br>

**select**
<br>

````
int select(int n, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);
````
**poll**
<br>

````
int poll(struct pollfd *fds, unsigned int nfds, int timeout);
````

**epoll**
<br>
````
int epoll_create(int size);
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)；
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
````
epoll_ctl()函数用于向内核注册新的描述符或者是改变某个文件描述符的状态。已注册的描述符在内核中会被维护在一棵红黑树上，通过回调函数内核会将I/O准备好的描述符加入到一个链表中管理，进程调用epoll_wait()函数便可以得到事件完成的描述符。
epoll比select和poll更加灵活而且没有描述符数量限制。

**上面三个实现虽然看起来像一个进化的过程，但并不是后面出现的一定比前面的好，事实他们的特性不同有各自的适用场景。**
<br>
<div align=center><img src="https://github.com/BBLLMYD/blog/blob/master/images/03/3-8-0.png?raw=true" width="937"></div>
<div align=center>适用场景</div>
<br>


* * *

### 3.关于“sendfile”、"mmap"、"DMA"的特性及应用场景

如果我们始终让CPU来参与进行各种数据传输工作，会比较浪费。我们的数据传输工作用不到多少CPU核新的“计算”功能。另外CPU的运转速度和I/O操作的速度也并不匹配，会降低CPU的使用效率。
<br><br>
关于零拷贝（零复制）维基百科中有如下的描述：

>零复制（英语：Zero-copy；也译零拷贝）技术是指计算机执行操作时，CPU不需要先将数据从某处内存复制到另一个特定区域。这种技术通常用于通过网络传输文件时节省CPU周期和内存带宽。


#### 3.1 关于DMA

说一下DMA的背景，DMA其实是为了让CPU更专注于计算本身，而不再过多的用在数据传输上，因为CPU的运转速度比I/O操作要快很多，
并且IO过程中需要多次中断和切换上下文的操作，过多的用在传输上是有浪费资源的情况发生并且这不是很合理的实现。有了DMA的数据传输机制，既可以大幅提升I/O的吞吐率，
也可以一定程度的解放CPU，**让CPU避免做具体的"数据搬运"工作，而去做更多的做好计算的职责。**DMA直接同内存发生成块的数据交换，因此I/O效率比较高，但是由于DMA窃取了时钟周期，
占据了访问内存的数据总线，可能会影响CPU处理效率（但是此时CPU的一级二级缓存会比较发挥作用，~~CPU直接参与IO~~），这和DMA控制器的性能也有很大关系，总的来说DMA在解放CPU在IO的工作这件事上已经是一种相对有效的处理方式。


    所以"零拷贝"实际上不是不做copy，数据传输是不可避免要经过copy的，"零拷贝"是为了在传统的IO模式下尽量减少多余的拷贝次数，
    或者尽量减少拷贝过程中CPU的参与，DMA在一些"零拷贝"的实现过程中可以视为是一个重要的工具。"零拷贝"通常也是以一个抽象的概念出现，
    因为不同的上下文场景下说的"零拷贝"可能不一定完全是一回事儿，如netty、kafka、Spark、JavaNIO等等他们的零拷贝机制的实现由于
    各个组件在各自场景的上下文环境并不全都一致，所以选择的用来实现各自"零拷贝"特性的基础技术也未必一致，
    但是不变的是他们的目的都是为了减少上下文切换次数和数据饿的copy次数，并且所依赖的基础技术的支持也都来自于操作系统给上层提供的接口。


#### 3.2 mmap文件映射
虚拟映射只支持文件，在进程的非堆内存区域开辟一块内存空间，和OS内核空间的一块内存进行映射，
**应用程序就可以直接对Page Cache层（可以直接理解为磁盘文件）中的数据进行读写操作，可以让应用程序像是访问常规变量的方式一样访问文件中的数据。**
但由于Page Cache也是文件系统的再上一层，所以在读写操作时候相对会有一定的一致性的风险。
可以抽象的理解为"指针"，不一样的是从应用程序的角度这里就是直接操作"指针"本身。
基于mmap技术的特性可以实现很多功能，比如多进程数据共享和通信、跨进程锁等。

#### 3.3 sendfile(in,out)
sendfile的实际应用很多，数据直接在内核完成输入和输出，不需要拷贝到用户空间再写出去。
**sendfile可以直接将Page Cache中某个fd的一部分数据传递给另外一个fd，而不用经过到用户空间的两次copy。其中sendfile(in,out)中的"in"只可以是从磁盘文件，而"out"可以是磁盘句柄也可以是socket句柄。**
kafka就是基于此特性实现了从broker向consumer传输数据过程的"零拷贝"（其实是减少了关于用户缓冲区的copy并且）。
sendfile有一个天然的特性或者限制，就是应用层无法再对数据进行加工，而只能直接传输，而这种弊端在适合的场景下也有可能成为应用层的优势。

- 传统I/O ：硬盘—>内核缓冲区—>用户缓冲区—>内核socket缓冲区—>协议引
- sendfile（ DMA 收集拷贝）：硬盘—>内核缓冲区—>协议引擎

下图是sendfile在读取本读数据发送到socket文件场景下的的一个应用实践。

<br>
<div align=center><img src="https://github.com/BBLLMYD/blog/blob/master/images/03/3-123.png?raw=true" width="777"></div>
<div align=center>sendfile()应用前后</div>
<br>


#### 优化手段的比较

    应用层关于实现磁盘/网络IO的"零拷贝"优化总要依赖的底层技术实现通常包括但是不限于直接内存映射、直接内存读写、DMA拷贝、sendfile()等手段，
    他们分别有着不同的适用场景供上层应用来封装，同时也或多或少着存在一些弊端在实际应用中需要注意。

<br>
<div align=center><img src="https://github.com/BBLLMYD/blog/blob/master/images/03/3-11-0.png?raw=true" width="777"></div>
<div align=center>IO过程优化</div>
<br>

* * *

### 4.关于应用层对上述技术的实际应用

实际应用层的实践中，就是以减少冗余的copy次数、提升CPU吞吐率为目的基于传统的IO方式进行优化。
而具体如何选择运用操作系统提供的上述基础支持的api，就要看当前应用类型和所需要实现的特性了。
总的来说在IO密集型的上层应用，提高CPU的吞吐率的一个原则就是尽量要去减少或控制没必要的线程阻塞，
因为阻塞的线程在操作系统层面和可支配线程不是在同一个队列中保存的，过多的阻塞会更浪费资源大大降低吞吐率。

* * *

### PS:关于"上下文切换"的一点讨论

我们常常会提到上下文切换所带来的开销，
当发生上下文切换时CPU需要花费大量的时间用于处理如:保存和恢复寄存器和内存页表、更新内核相关数据结构等操作。
但是上下文切换是操作系统内核优化的一个关键参数指标，上下文切换实际上是为了让CPU利用率更高的一种手段。
其实是不可能避免的，比如当前正在执行的任务完成，系统的CPU正常调度下一个任务就必然要经过上下文的切换。
关于上下文切换的开销细节在[另一篇关于锁的文章]()中有更详细的介绍。
**所以我们应该关注的是如何避免不合理或消除没必要的切换**，这样才更合理的提高多线程程序的运行效率，
比如多个任务并发抢占锁资源时避免线程挂起而是以CAS的方式竞争或者无锁的并发编程；比如协程的应用等。







































