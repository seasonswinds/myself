# Spring MVC

### DispatcherServlet

> 分发请求
>
> 所有的Web请求都要通过它来处理
>
> - 映射策略
>   - BeanNameUrlHandlerMapping(默认)
> - 持有一个以自己的Servlet名称命名的IoC容器，是一个WebApplicationContext对象，拥有自己的Bean定义空间

- 初始化，initServletBean()

- 响应HTTP请求，doService()，doDispatch()

  > 继承了FrameworkServlet和HttpServletBean，通过使用Servlet API来对HTTP请求进行响应

- 为HTTP请求找到相应的Controller控制器，HandlerMapping

  > HashMap，保存了URL请求和Controller的映射关系

- 把获得的模型数据交给特定的视图对象

### ContextLoaderListener

> 被定义为一个监听器，与Web服务器的生命周期相关联
>
> 负责完成IoC容器在Web环境中的启动工作

- contextInitialized，服务器启动时被调用
- contextDestroyed，服务器将要关闭时被调用

> 在ContextLoader中完成了两个IoC容器的建立过程
>
> - 在Web容器中建立父IoC容器
> - 生成相应的WebApplicationContext并初始化(XmlWebApplicationContext)
>   - 从Servlet事件中得到ServletContext
>   - 读取配置在web.xml中的各个相关的属性值
>   - 实例化WebApplicationContext，并完成载入和初始化

