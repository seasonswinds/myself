# Spring IoC

### IoC容器初始化过程(refresh方法)

- BeanDefinition的Resources定位(由ResourceLoader完成)

- BeanDefinition载入

  > - 把定义好的Bean表示成IoC容器内部的数据结构——BeanDefinition
  >
  > - 对象的实例化是在依赖注入时完成的
  > - 在同一个Bean中有同名的property设置，起作用的只是第一个

- 向IoC容器注册BeanDefinition(通过调用BeanDefinitionRegistry接口的实现完成)

  > - Bean定义的载入和依赖注入是两个独立的过程。依赖注入一般发生在应用第一次通过getBean向容器获取Bean的时候。但有一个例外，如果Bean设置了lazyinit属性，这个Bean的依赖注入在IoC容器初始化时就预先完成了。
  >
  > - BeanDefinition数据在IoC容器中通过一个HashMap来保持和维护
  >
  > - DefaultListableBeanFactory中，是通过一个ConcurrentHashMap来持有载入的BeanDefinition的
  >
  >   - 本身是ConcurrentHashMap，为什么还要加锁？
  >
  >     > 要对多个集合进行操作，需要保证操作的原子性。例如，BeanDefinition的名称列表beanDefinitionNames
  >
  > - 如果遇到同名的BeanDefinition，需要依据allowBeanDefinitionOverriding(是否允许覆盖)的配置来完成
  >
  >   > - 检查是否有同名的BeanDefinition已经在IoC容器中注册
  >   > - 如果有，但又不允许覆盖(allowBeanDefinitionOverriding == false)，抛出异常
  >
  > - 注册的过程需要synchronized保证数据一致性(对整个ConcurrentHashMap加锁)
  >
  > - 完成了BeanDefinition的注册，就完成了IoC容器初始化的过程

### IoC容器的依赖注入

> - 依赖注入的过程，是用户第一次向IoC获取Bean时触发。也有例外，可以通过控制lazy-init属性让容器完成对Bean的预实例化预实例化也是一个依赖注入的过程。
> - 依赖注入的发生是在容器中的BeanDefinition数据已经建立好的前提下进行的。

- 首先从缓存中获取，处理那些单件模式的Bean

- 查找BeanDefinition(递归，当前BeanFactory中取不到，则到父BeanFactory中查找)

- 获取当前Bean的所有依赖Bean(递归调用getBean)

- 创建Bean实例

  > 如果Bean配置了PostProcessor，那么这里返回的是一个proxy

  - createBeanInstance，创建实例
    - CGLIB
    - BeanUtils，Java反射
  - populateBean，依赖关系解析及依赖Bean注入
  - initializeBean，Bean初始化
    - 把BeanName、BeanClassLoader以及BeanFactory注入到Bean中
    - postProcessBeforeInitialization调用
    - 调用初始化方法
    - postProcessAfterInitialization调用

- 类型检查

- 返回Bean实例

### 容器的初始化和销毁

- 初始化
  - 配置ClassLoader、PropertyEditor和BeanPostProcessor等
- 关闭
  - 发出容器关闭信号
  - 将Bean逐个关闭
  - 关闭容器自身

### Bean的生命周期

- Bean实例创建
- 为Bean实例设置属性
- 调用Bean的初始化方法
- 应用可以通过IoC容器使用Bean
- 当容器关闭时，diaoyongBean的销毁方法
  - 调用postProcessBeforeDestruction
  - 调用Bean的destroy方法
  - 调用Bean的自定义销毁方法

### FactoryBean



### BeanPostProcessor

> Bean后置处理器，是一个监听器，可以监听容器触发的事件。将它向容器注册后，容器中管理的Bean具备了接收IoC容器事件回调的能力。
>
> - postProcessBeforeInitialization，在Bean初始化前提供回调入口；
> - postProcessAfterInitialization，在Bean初始化后提供回调入口；
>
> 这两个Bean后置处理器定义的接口方法，一前一后，围绕着Bean定义的init-method方法调用。

### Bean的依赖检查

> 在Bean定义中设置dependency-check属性。会对Bean的Dependencies属性进行检查，如果不满足要求，抛出异常

- none(默认)
- simple
- object
- all

### Bean对IoC容器的感知

- BeanNameAware，在Bean中获取它在IoC容器中的名称
- BeanFactoryAware，在Bean中获取它所在的IoC容器
- ApplicationContextAware，在Bean中获取它所在的应用上下文
- MessageSourceAware，在Bean中获取消息源
- ApplicationEventPublisherAware，在Bean中获取应用上下文的事件发布器
- ResourceLoaderAware，在Bean中获取ResourceLoder