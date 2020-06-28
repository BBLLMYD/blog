
关于SPI机制好像被提及的概率并不是很高，
我认为主要原因是其实很多组件的实现都应用到了或者说可以体现SPI机制的抽象的思想，
比如JDBC、Dubbo等，而这些具体的应用可能不会太明显的能体现得到SPI机制相关的名词，或者可以说

但是从SPI名字的全称（Service Provider Interface）就可以看出来，**SPI是一种面向接口/服务的扩展机制**。
而Java作为一种有着运行时动态分派特性的语言，面向对象/接口编程易扩展的特性也正是其优势。
**所以我认为与其说SPI是一种机制，不如说SPI实际上是利用Java语言特性而做的一种实现上的约定或者规范模式。**

SPI机制在wikipedia的一段描述：

> A service is a well-known set of interfaces and (usually abstract) classes. A service provider is a specific implementation of a service. The classes in a provider typically implement the interfaces and subclass the classes defined in the service itself. Service providers can be installed in an implementation of the Java platform in the form of extensions, that is, jar files placed into any of the usual extension directories. Providers can also be made available by adding them to the application's class path or by some other platform-specific means.


