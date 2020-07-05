
现在云原生的概念很火，也是业界的一个趋势。
不同于以往我们的Java应用直接部署在Linux服务器，现在越来越多的应用容器化，
也就是应用和操作系统直接多了一个容器作为中间角色。

这种多了一层的内容好像随之就多了一份风险和损耗，同时增加了复杂度。
既然成本看起来这么高，为啥还有这么多的各种应用支持和趋向容器化呢，容器化到底能从哪些方面带来多少好处呢？
容器化的弊端可能有哪些？想要自由踏实的将你的应用容器化需要哪些基础知识的储备？



- [1. 容器的本质]()
- [2. 运行和编排]()

---

### 1.容器的本质

容器的本质其实是一个特殊的进程。

主要特殊在两个方面，一是对进程可用资源边界的约束，而是对进程视图的上下文做更改。

NameSpace（更改视图）技术和Cgroups（制造约束），以上两点特性的实现都本质上都是基于操作系统的内核提供的能力来封装。

- NameSpace

NameSpace概念的本质其实是在Linux下创建进程时，指定CLONE_NEW*这个可选参数。
````
int pid = clone(main_function, stack_size, CLONE_NEW* | SIGCHLD, NULL); 
````
Linux下根据视图内容已经提供了几种不同的Namespace类型，如下图：

<br>                                     
<div align=center><img src="https://github.com/BBLLMYD/blog/blob/master/images/13/1301.png?raw=true" alt="Namespace类型" width="555"></div>
<div align=center>Namespace类型</div>
<br>

这些各个NameSpace相应的数据结构和算法实现都在内核中封装，容器技术实现时只需要做系统调用。
所以容器本质上是通过基于内核的NameSpace机制将文件系统、网络协议栈等信息在视图上和主机隔离。

- Cgroups

Linux下实现容器化的技术有很多，Docker是一个里程碑式的容器技术，
容器技术的本质是约束进程的资源边界，修改容器视图的上下文。

Linux内核进程模型。




