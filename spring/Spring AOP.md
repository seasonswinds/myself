# Spring AOP

### 什么是AOP?

> AOP是(Aspect-Oriented Programming，面向方面编程、面向切面编程)
>
> Aspect是一种用来描述分散在对象、类或者函数中的横切关注点(crosscutting concern)。使解决特定领域的代码从业务逻辑中独立出来，使得业务逻辑的代码中不再含有针对特定领域问题代码的调用。
>
> 业务逻辑和特定领域问题的关系通过切面来封装、维护

### AOP体系结构

- 基础(base)，待增强对象、目标对象；

- 切面(aspect)，对于基础的增强应用；

- 配置(configuration)，编织，将基础和切面结合起来；

  > Spring AOP中使用的是Java本身的语言特性，入Java Proxy代理、拦截器等技术，来完成AOP编织的实现

### Spring AOP中包含哪些实体？

- Advice(通知)，定义在连接点做什么，为切面增强提供织入接口

  - BeforeAdvice

    > 在调用目标方法时被回调
    >
    > 参数有：
    >
    > - Method对象，目标方法的反射对象；
    > - Object[]对象数组，目标方法的输入参数；

  - AfterAdvice

    - AfterReturningAdvice

      > 目标方法调用结束并成功返回的时候回调
      >
      > 参数有：
      >
      > - 目标方法返回结果
      > - 目标方法反射对象
      > - 目标方法调用参数

  - ThrowsAdvice

    > 抛出异常时被回调，使用反射机制实现

- Pointcut(切点)，决定Advice通知应该作用于哪个连接点

  > 通过Pointcut定义需要增强的方法的集合
  >
  > 需要返回一个MethodMatcher，由这个MethodMatcher对象来判断是否需要对当前方法调用进行增强，或者是否需要对当前调用方法应用配置好的Advice通知。

  - JdkRegexpMethodPointcut，正则表达式切点
  - NameMatchMethodPointcut，方法名匹配切点

- Advisor(通知器)，结合Advice与Pointcut

  > 通过Advisor，定义该使用哪个通知并在哪个关注点使用它