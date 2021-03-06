作为使用Java语言的程序开发者，对[Spring](https://spring.io/)应该都是不能再熟悉了，
甚至有说法Java开发眼中的面向对象编程实际上就是面向spring编程。
我觉得这种说法虽然有点极端，但是其实也有一定的道理，
从分布上看，Java业务后台在基于Spring生态下的编程是最常见的，
Spring优秀的设计已经让Spring成为了大多数业务后台技术选型的第一或者默认的选择。

> Spring框架是 Java 平台的一个开源的全栈（Full-stack）应用程序框架和控制反转容器实现，一般被直接称为 Spring。该框架的一些核心功能理论上可用于任何 Java 应用，但 Spring 还为基于Java企业版平台构建的 Web 应用提供了大量的拓展支持。

关于Spring的源码，或者说不管是啥开源项目的源码，个人认为认为一定是要有目的的去看，
而且有个重要的前置条件就是对正在看源码的系统功能的应用程度，是需要有一定程度的熟练的。这样才能知道你在看的是什么。

我认为对于读源码的目的，就是提炼出优秀部分的设计的思想再加以应用。
比如Spring中有很多处优雅的抽象和设计，**而把它这些设计的思想再抽象出来，
而能够合理舒适的在我们当下的业务系统等的场景再具体应用，** 学以致用可能会更加深理解。

用马老板的话说，**Spring的出现并不是必然，IOC和AOP模式的出现是必然。
实现IOC和AOP的可能是Spring，也可能是Summer，或者是Autumn。**

这篇文章尝试主要从下面的几个角度来分别讨论关于Spring的设计。

- [1. Spring的编程模型](https://github.com/BBLLMYD/blog/blob/master/blogs/%E5%85%B3%E4%BA%8E%E9%9D%A2%E5%90%91Spring%E7%BC%96%E7%A8%8B.md#1-spring%E7%9A%84%E7%BC%96%E7%A8%8B%E6%A8%A1%E5%9E%8B)
- [2. Spring的规范整合](https://github.com/BBLLMYD/blog/blob/master/blogs/%E5%85%B3%E4%BA%8E%E9%9D%A2%E5%90%91Spring%E7%BC%96%E7%A8%8B.md#2-spring%E7%9A%84%E8%A7%84%E8%8C%83%E6%95%B4%E5%90%88)
- [3. Spring的扩展点](https://github.com/BBLLMYD/blog/blob/master/blogs/%E5%85%B3%E4%BA%8E%E9%9D%A2%E5%90%91Spring%E7%BC%96%E7%A8%8B.md#3-spring%E7%9A%84%E6%89%A9%E5%B1%95%E7%82%B9)
- [4. 总结](https://github.com/BBLLMYD/blog/blob/master/blogs/%E5%85%B3%E4%BA%8E%E9%9D%A2%E5%90%91Spring%E7%BC%96%E7%A8%8B.md#4-%E6%80%BB%E7%BB%93)

---

### 1. Spring的编程模型

维护模型的重要性，理解一个模型的设计是要解决什么问题，才能有针对性的设计出良好的抽象组合，并且以维护和拓展模型为驱动和基础方向，更好的适应未来的需求。

在解决对象创建和组装的问题上，Spring在实现IOC和DI的设计上抽象出了非常好的模型，在这种模型下编程用户体验舒适甚至感觉不到自己在某个特定的模型下，极大的简化和规范了开发。
理解模型提供的能力，了解模型内容和设计的来龙去脉，是了解一个应用的设计的根本，在这基础上再看对于接口和实现的设计。

Spring作为一个如次受欢迎的应用层框架，有着很多用着很舒适的特性。
从根本上看，Spring就是致力于管理你的业务对象的容器框架；再从你的业务对象类别和特性的角度出发来分开扩展。
总看来看我认为不光是要理解Spring中多个抽象层的设计，因为这些信息相对Spring整体的编程模型也是相对处在局部的。


> 模型 -> 接口 -> 实现

<div align=center><img src="https://github.com/BBLLMYD/blog/blob/master/images/14/1402.png?raw=true" alt="Spring编程模型" width="711"></div>
<div align=center>Spring编程模型</div>
<br>

Spring有Spring框架的编程模型；<br>
Netty有Netty的编程模型；<br>
Flink也有Flink的编程模型；<br>
业务应用也应该有着贴合当前业务的良好可扩展的编程模型。

---

### 2. Spring的规范整合

我认为Spring的成功很大程度上得益于整体模型的良好设计以及**对各个方向的规范进行的整合，**
也正是这些内容才促成了一定程度的"面向Spring"编程的现象。

<div align=center><img src="https://github.com/BBLLMYD/blog/blob/master/images/14/1401.png?raw=true" alt="Spring规范整合" width="711"></div>
<div align=center>Spring规范整合</div>
<br>

<br>

--- 

### 3. Spring的扩展点

> 阅读文档可以让你熟练的使用一个工具，阅读源码可以让你了解如何设计一个更好的系统。

Spring作为一个开源框架，实现中肯定有太多的细枝末节，如果过早的就陷进去，一方面会比较迷惘难理得清，
因为对当前细节所在的上下文还没有足够的了解和认识。

另一方面我认为在当前情况下纠结于很多细节的意义并不大，因为真正能体现设计思想或者决定设计方向的关键往往不在很多细节里，
而是在相对宏观的组件和模块上抽象。但这部分内容通常不会占过多的篇幅，却决定了实现的方向。

真正能对应用模块间的协调方式和整体走向的发展起到更决定性影响的，一定是这些关键抽象层良好的设计，所以此时最好不要或者说不必囿于细节。
我认为这也正是面向接口编程的一个意义，

Spring作为一个应用层容器框架在设计上不太涉及底层，也没有过于偏重性能的特性，Spring更具价值的地方在于对框架模型抽象的设计和实现上。
下面内容就以这些设计思想为出发点来简单的讨论一下。

---

### 4. 总结

要知道Spring庞大的生态，一定是有一个良好的根基作为扩展基础的，Spring各个组件所有的代码中80%可能都是依托于20%的核心代码，同时也是[局部性原理](https://github.com/BBLLMYD/blog/blob/master/blogs/%E6%8A%BD%E8%B1%A1%E4%B9%8B%E4%BA%8E%E2%80%9C%E5%B1%80%E9%83%A8%E6%80%A7%E5%8E%9F%E7%90%86%E2%80%9D.md)的一种体现，
甚至20%也多说了，因为良好的扩展插件机制，在业务系统中和框架整合方面的定制扩展已经变得非常容易。


