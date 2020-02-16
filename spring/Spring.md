# Spring

### 什么是Spring？有什么优点？

> Spring 是个java企业级应用的开源开发框架。
>
> Spring 框架目标是简化Java企业级应用开发，并通过POJO为基础的编程模型促进良好的编程习惯。

- 轻量
- 控制反转：通过控制反转，实现了松散耦合
- 面向切面：可以把业务逻辑和特定领域逻辑分开
- 容器：Spring包含并管理应用中对象的生命周期和配置
- MVC：
- 事务：
- 异常处理：Spring提供方便的API把数据库异常转换为一致的unchecked异常

### Spring由哪些模块组成

- Core module
- Bean module
- Context module
- Expression Language module
- JDBC module
- ORM module
- OXM module
- Java Messaging Service(JMS) module
- Transaction module
- Web module
- Web-Servlet module
- Web-Struts module
- Web-Portlet module

### Bean的生命周期

- Spring容器 从XML 文件中读取bean的定义，并实例化bean
-  Spring根据bean的定义填充所有的属性。
-  如果bean实现了BeanNameAware 接口，Spring 传递bean 的ID 到 setBeanName方法；
-  如果Bean 实现了 BeanFactoryAware 接口， Spring传递beanfactory 给setBeanFactory 方法；
- 如果有任何与bean相关联的BeanPostProcessors，Spring会在postProcesserBeforeInitialization()方法内调用它们
-  如果bean实现IntializingBean了，调用它的afterPropertySet方法
- 如果bean声明了初始化方法，调用此初始化方法。 
- 如果有BeanPostProcessors 和bean 关联，这些bean的postProcessAfterInitialization() 方法将被调用。
- 如果bean实现了 DisposableBean，它将调用destroy()方法。

### Bean的自动装配

- no：默认的方式是不进行自动装配，通过显式设置ref 属性来进行装配

- byName：通过参数名 自动装配

  > Spring容器在配置文件中发现bean的autowire属性被设置成byname，之后容器试图匹配、装配和该bean的属性具有相同名字的bean。

-  byType：通过参数类型自动装配

  > Spring容器在配置文件中发现bean的autowire属性被设置成byType，之后容器试图匹配、装配和该bean的属性具有相同类型的bean。如果有多个bean符合条件，则抛出错误。 

- constructor：这个方式类似于byType， 但是要提供给构造器参数，如果没有确定的带参数的构造器参数类型，将会抛出异常。 

- autodetect：首先尝试使用constructor来自动装配，如果无法工作，则使用byType方式。

### SpringBoot启动机制

