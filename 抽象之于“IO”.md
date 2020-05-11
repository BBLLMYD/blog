<br>

“I/O”词条在维基百科可以搜索到如下定义

>  I/O（英语：Input/Output），即输入／输出，通常指数据在存储器（内部和外部）或其他周边设备之间的输入和输出，是信息处理系统（例如计算机）与外部世界（可能是人类或另一信息处理系统）之间的通信。输入是系统接收的信号或数据，输出则是从其发送的信号或数据。该术语也可以用作行动的一部分；到“运行I/O”是运行输入或输出的操作。

“I/O”更像是一个动名词，简单说I/O的出现是为了实现信息交换和传输，是计算机实现通信的基础。进行实现通信的目的的过程中以及不同的场景下出现了不同的机制和模式。

<br>
因为我们编写的应用程序如需和其他程序进行通信，首先程序要运行，就必然脱离不了操作系统，无论是I/O设备还是I/O接口，操作系统提供了通信的基础设施。
所以不管是磁盘I/O还是网络I/O，如果我们单纯的了解我们应用程序内部的线程状态和分配策略，
而不对OS和App之间的交互基础有足够多的了解，是很容易迷茫的。先了解关于OS与用户程序之间的通信的种种模式，
才能根据我们的业务场景选型或实现更适合的通信组件。

<br>
不管是同一个OS下的不同进程间通信，还是不同机房之间的通信，都有着同步/异步，阻塞/非阻塞的类型机制区别。本质还是要依赖操作系统提供的I/O相关的函数给用户进程去调用。
这篇主要就讨论Linux下的关于网络I/O的内容展开做一些讨论。

<br>
常被提及的同步/异步，阻塞/非阻塞网上有很多解释，我觉得主要是因为大家在解释这个问题的前置设定可能不一致，导致很多说法好像都对但看起来又不一样，明确了上下文来谈这两个问题才会更清晰。
I/O的流程可分为两步：
<br>

* 1.等待数据准备好
* 2.从内核向进程copy数据

我们常讲的同步/异步指的其实是第二个动作，阻塞/非阻塞指的是第一个动作。<br>
各个I/O模型也是在两个步骤的实现方式不同而区分的。

<br>
其中的信号驱动I/O和异步I/O实际上应用的比较少，它们的实现方式相对和操作系统更耦合而不可控是很大一部分原因。

<br>
实际上第二个copy数据行为并不是一个动作，而是一系列动作，关于数据在内核缓冲区和用户缓冲区甚至DMA和protocol engine之间的拷贝，涉及几次上下文切换才可以完成。虽然看起来很麻烦，但由于内核缓冲区在核心位置起的重要作用让这些动作在性能上也并不会那么慢，但是当需要传输的数据远远大于内核缓冲区的大小时，内核缓冲区就会成为瓶颈，此时zerocopy技术又成为了I/O密集型应用的这种场景下一个好的解决方案。但是事情总是相对的，zerocopy虽然提升了性能免去了一些上下文的切换成本，但是少了一层用户态控制，可能会导致传输数据相对不可控。

<br>
再来分别看一下这5种Linux网络I/O模型

<br>

<br>
**同步阻塞I/O**
<br><br>
Linux中默认的socket就是阻塞式I/O，也就是上述两个阶段都是阻塞的方式来进行，对进程的线程资源很不友好，当大量连接时容易将线程资源耗尽无法处理新的连接。但是系统调用次数较少，在流量不大且平稳的时候比较适用。
<br>
<div align=center><img src="https://github.com/BBLLMYD/blog/blob/master/images/03/3-1.png?raw=true" width="444"></div>
<div align=center>同步阻塞I/O过程</div>
<br>


**同步非阻塞I/O**
<br><br>
和上一个同步阻塞I/O相比，同步非阻塞I/O在第一阶段的阻塞等待变成轮询的方式，相对之下避免了一下线程消耗，但是系统调用次数过多，CPU消耗也比较明显，这种模型实际应用也比较少。
<br>
<div align=center><img src="https://github.com/BBLLMYD/blog/blob/master/images/03/3-2.png?raw=true" width="444"></div>
<div align=center>同步非阻塞I/O过程</div>
<br>

**多路复用I/O模型**
<br><br>
多路复用建立在系统提供的事件分离函数select，poll，epoll之上，先更新select的socket监控列表，然后等待函数返回（此过程是阻塞的），关于这几个系统函数实现方式的特性的不同，也是适用不同的场景下。但是多路复用I/O在上层的实际应用很多，像Java的NI/O，Redis以及Netty通信框架都采用了这种模型，不过像Netty框架在多路复用的基础上又做了一下zerocopy等一些更细致的有针对性的优化方案。<br>
<br>
<div align=center><img src="https://github.com/BBLLMYD/blog/blob/master/images/03/3-3.png?raw=true" width="444"></div>
<div align=center>多路复用I/O过程</div>
<br>

**信号驱动I/O**
<br><br>
应用进程使用 sigactI/On 系统调用，内核立即返回，应用进程可以继续执行，也就是第一阶段是不阻塞的，等待系统向应用进程发送 SIGI/O信号，再来同步的向进程拷贝数据。CPU 利用率更高，但是过于依赖OS能力，在大量I/O操作的情况下可能造成信号队列溢出导致信号丢失，造成灾难性后果，所以实际应用也很少。
<br>
<div align=center><img src="https://github.com/BBLLMYD/blog/blob/master/images/03/3-4.png?raw=true" width="444"></div>
<div align=center>信号驱动I/O过程</div>
<br>

**异步I/O**
<br><br>
两个阶段都是不阻塞，第二阶段系统直接向进程通知I/O已经完成，此时进程可以直接使用数据。异步模型效率较高，但是还未足够成熟，同时也过于依赖操作系统，实际应用的也并不多。
<br>
<div align=center><img src="https://github.com/BBLLMYD/blog/blob/master/images/03/3-5.png?raw=true" width="444"></div>
<div align=center>异步I/O过程</div>
<br>

**为了低耦合，更可控，异步I/O和信号驱动I/O模型应用并不多见，更常见的是向对更灵活的多路复用和传统I/O模型。**

为了避免我们的应用程序由于I/O导致线上故障，应用中的I/O操作最好用线程池隔离限制，避免由此引起整个进程的全局故障。

* * *

select/poll/epoll做为实现I/O多路复用的手段，出现的时序是select->poll->epoll
<br>
select 和 poll 的功能现实上大致相同，在一些实现细节上有所不同：
* select()会修改描述符，而poll()不会；
* poll()提供了更多的事件类型，并且对描述符的重复利用上比select相对要高;
...
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