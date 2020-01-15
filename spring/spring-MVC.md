# Spring-MVC





MVC是一种设计模式。

- M（model）数据
- V（view）视图，界面；
- C（controller）行为



## 使用



## 原理

启动流程



**处理请求流程：**

1.用户HTTP请求 ——> DispatcherServlet（调度器根据请求进行调度）

2.DispatcherServlet ——> HandlerMapping(找到对应的处理器)

3.HandlerMapping ——> Controller(调用对应的处理器)

4.Controller ——>业务层（调用业务层完成相应的业务）

5.业务层 ——> ModelAndView(处理结果数据)

6.ModelAndView ——>DispatcherServlet ——>ViewResolver(视图解析器做处理)

7.ViewResolver ——> View(模型数据显示) ——>用户