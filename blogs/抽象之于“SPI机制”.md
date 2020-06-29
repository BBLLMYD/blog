
关于Java中的SPI机制貌似被提及的概率并不是很高，
我认为主要原因是其实很多组件的实现都应用到了或者说可以体现SPI机制的抽象的思想，
比如JDBC、Dubbo等，而这些具体的应用可能不会太明显的能体现得到SPI机制相关的名词，或者说SPI本身就是比较隐示的应用机制。

但是从SPI名字的全称（Service Provider Interface）就可以看出来，**SPI是一种面向接口/服务的扩展机制**。
而Java作为一种有着运行时动态分派特性的语言，面向对象/接口编程易扩展的特性也正是其优势。
**所以我认为与其说SPI是一种机制，不如说SPI实际上是利用Java语言特性而做的一种实现上的约定或者规范模式。**
SPI是体现Java语言特性和抽象思想的一个典范。

SPI机制在wikipedia的一段描述：

> A service is a well-known set of interfaces and (usually abstract) classes. A service provider is a specific implementation of a service. The classes in a provider typically implement the interfaces and subclass the classes defined in the service itself. Service providers can be installed in an implementation of the Java platform in the form of extensions, that is, jar files placed into any of the usual extension directories. Providers can also be made available by adding them to the application's class path or by some other platform-specific means.

SPI机制其实和SpringBoot的约定大约配置的思想类似，都是简化或者缺省了显示配置的步骤。
下面分别从SPI机制的思想和一些应用示例来展开讨论和拓展一下。

- [1.SPI机制的应用]()
- [2.SPI思想的拓展]()
- [3.总结]()


* * *

### 1.SPI机制的应用

SPI机制在很多框架中都有相应实现，我们可以根据JDK的实现机制来做一个示例，然后看其他开源组件分别是如何实现/体现的。

#### 1.1使用JDK的SPI机制示例

调用方依赖的是接口定义，实现方可以做不同的扩展。这样客户端就可以无需配置实现可拔插的选择。
对SPI机制应用的开源组件有很多，我们自己也可以做相关的实现方式。

- （1） 定义接口：
```
    public interface HelloWorld {
        void helloWorld();
    }
```
- （2） 提供不同的接口实现类：
```
    public class HelloWorldImpl1 implements HelloWorld {
        @Override
        public void helloWorld() {
            System.out.println(this.getClass().getName());
        }
    }
    
    public class HelloWorldImpl2 implements HelloWorld {
        @Override
        public void helloWorld() {
            System.out.println(this.getClass().getName());
        }
    }
```
- （3） 客户端代码：
```
    public static void main(String[] args) {
        ServiceLoader<HelloWorld> load = ServiceLoader.load(HelloWorld.class);
        Iterator<HelloWorld> iterator = load.iterator();
        while (iterator.hasNext()){
            HelloWorld helloWorld = iterator.next();
            helloWorld.helloWorld();
        }
    }
```
在resources目录下添加文件名为com.skr.spi.HelloWorld的文件，文件内容为两行分别是com.skr.spi.HelloWorldImpl1和com.skr.spi.HelloWorldImpl2

- （4） 执行输出结果：

com.skr.spi.HelloWorldImpl1<br>
com.skr.spi.HelloWorldImpl2

<br>

以上就是使用JDK的SPI机制实现的接口服务发现机制示例，客户端代码中的ServiceLoader存在下面一行代码：
private static final String PREFIX = "META-INF/services/";
也就是说"META-INF/services/"目录就是ServiceLoader和客户端约定的一个目标目录。

ServiceLoader本身其实就是一个java.lang.Iterable接口的实现，会来迭代配置中的实现类和以及校验和初始化实例等工作。
基于这个思路其实就比较容易扩展类似的功能了。

#### 1.2开源组件的SPI机制应用

- JDBC

比如JDBC的扩展方式，在java.sql.DriverManager中初始化Driver实现的loadInitialDrivers()方法中，指定了java.sql.Driver接口。
<div align=center><img src="https://github.com/BBLLMYD/blog/blob/master/images/11/1102.png?raw=true" width="666"></div>
<div align=center>DriverManager指定的抽象层</div>
<br>

在MySQL的驱动实现中根据加载规范来进行配置，如下图。
<div align=center><img src="https://github.com/BBLLMYD/blog/blob/master/images/11/1103.png?raw=true" width="444"></div>
<div align=center>MySQL驱动的配置</div>
<br>

和JDBC类似定义规范的实现模式还有如apache的common-logging等。

- Dubbo

Dubbo没有使用Java原生的SPI机制，是对其进行了多维度的增强，除了支持原来的加载接口实现类，还增加了Dubbo的IOC和AOP等特性。
Dubbo采用SPI机制实现了在注册中心、监控中心、网络传输、负载均衡等等几乎RPC所有相关组件基于抽象层的扩展方式。

<div align=center><img src="https://github.com/BBLLMYD/blog/blob/master/images/11/1105.png?raw=true" width="666"></div>
<div align=center>Dubbo的SPI扩展</div>
<br>


具体的Dubbo SPI介绍可以直接看Dubbo官方文档
的[开发者指南](http://dubbo.apache.org/zh-cn/docs/source_code_guide/dubbo-spi.html)及[源码导读](http://dubbo.apache.org/zh-cn/docs/source_code_guide/dubbo-spi.html)部分。
文档中不仅有对框架本身设计的说明，同时包含很多设计上的理念，这些内容对我们学习框架本身以及框架之外的设计思想都是很有启发的。

* * *

### 2.SPI思想的拓展

SPI本质上就是把抽象层和实现层分离，并且约定注册方式的一种方案。
实际上这种思想在Java的开源项目里是很常见到的，因为这种思想天然符合Java的语言特性。
不仅是在RPC场景有比较天然的服务注册和发现，
其实Spring、SpringBoot或者一些工具的集成插件都直接应用着SPI机制或间接用着SPI机制设计的模式。

<div align=center><img src="https://github.com/BBLLMYD/blog/blob/master/images/11/1106.png?raw=true" width="555"></div>
<div align=center>SPI机制相关</div>
<br>

* * *

### 3.总结

原生SPI机制的技术原理好像并不复杂，但是简单思维也可以拓展出像Dubbo一样全面也精彩的设计应用。
它对于抽象和实现分离管理的机制和思想是具有启发性和通用性的，我认为这也是SPI机制更大的价值。







