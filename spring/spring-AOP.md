# Spring-AOP

[TOC]

## AOP

AOP（Aspect Oriented Programming，面向切面编程），我们在处理不同业务逻辑时，可能会遇到一些公共、抽象的方法，比如日志，事务，鉴权等，可以抽象为一个切面，方便维护

基本概念



通常有两种实现方式

- spring AOP
- AspectJ

## Spring AOP

### 使用

### 原理

## AspectJ

### 使用

### 原理



## 总结

- **复杂度和目标不同**

Spring Apo并非一个完整AOP的解决方案。仅通过Spring IoC提供一个简单的AOP实现，用于Spring容器管理的beans。

- **原理不同**

AspectJ使用的是编译期和类加载时进行织入，它引入了自己的编译期，称为AspectJ compiler (ajc)。通过它，我们编译应用程序，然后通过提供一个小的（<100K）运行时库运行它

- 编译时织入：AspectJ编译器同时加载我们切面的源代码和我们的应用程序，并生成一个织入后的类文件作为输出。
- 编译后织入：这就是所熟悉的二进制织入。它被用来编织现有的类文件和JAR文件与我们的切面。
- 加载时织入：这和之前的二进制编织完全一样，所不同的是织入会被延后，直到类加载器将类加载到JVM。

Spring AOP利用的是基于动态代理。包括

- JDK代理
- CGLIB代理

**性能**

考虑到性能问题，编译时织入比运行时织入快很多。Spring AOP是基于代理的框架，因此应用运行时会有目标类的代理对象生成。另外，每个切面还有一些方法调用，这会对性能造成影响。

AspectJ不同于Spring AOP，是在应用执行前织入切面到代码中，没有额外的运行时开销。

**参考文档**

[比较Spring AOP与AspectJ](https://juejin.im/post/5a695b3cf265da3e47449471)

[Intro to AspectJ](https://www.baeldung.com/aspectj)