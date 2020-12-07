# 1.Spring MVC执行原理

1）客户端发送请求到DispatcherServlet

2）DispatcherServlet查询handlerMapping找到处理请求的Controller

3）Controller调用业务逻辑后，返回ModelAndView

4）DispatcherServlet查询ModelAndView，找到指定视图

5）视图将结果返回到客户端

![img](https://gitee.com/adambang/pic/raw/master/20201106090650.webp)

# 2.什么是IOC？

控制反转就是把创建和管理 bean 的过程转移给了第三方，IOC容器。

> Spring 通过一个配置文件描述 Bean 及 Bean 之间的依赖关系，利用 Java 语言的反射功能实例化
> Bean 并建立 Bean 之间的依赖关系。 Spring 的 IoC 容器在完成这些底层工作的基础上，还提供
> 了 Bean 实例缓存、生命周期管理、 Bean 实例代理、事件发布、资源装载等高级服务。  

### IoC 容器设计：

使用 `ApplicationContext`，它是 `BeanFactory` 的子类，更好的补充并实现了 `BeanFactory` 的功能。

BeanFactory 按需加载bean，而 ApplicationContext 则在启动时加载所有bean。因此，BeanFactory与ApplicationContext相比是轻量级的。

# 3.解释Bean的生命周期

**Bean创建的三个阶段**

Spring在创建一个Bean时是分为三个步骤的

- 实例化，可以理解为new一个对象
- 属性注入，可以理解为调用setter方法完成属性注入
- 初始化，你可以按照Spring的规则配置一些初始化的方法（例如，`@PostConstruct`注解）

Bean的生命周期的过程，它大致分为

- 实例化
- 属性赋值
- 初始化
- 销毁

![preview](https://gitee.com/adambang/pic/raw/master/20201106090645.jpeg)

# 4.Spring APO 原理  

AOP：Aspect oriented programming 面向切面编程，AOP 是 OOP（面向对象编程）的一种延续。

AOP术语。

- 连接点（join point）：对应的是具体被拦截的对象，因为Spring只能支持方法，所以被拦截的对象往往就是指特定的方法，例如，我们前面提到的HelloServiceImpl的sayHello方法就是一个连接点，AOP将通过动态代理技术把它织入对应的流程中。
- 切点（point cut）：有时候，我们的切面不单单应用于单个方法，也可能是多个类的不同方法，这时，可以通过正则式和指示器的规则去定义，从而适配连接点。切点就是提供这样一个功能的概念。
- 通知（advice）：就是按照约定的流程下的方法，分为前置通知（beforeadvice）、后置通知（after advice）、环绕通知（around advice）、事后返回通知（afterReturning advice）和异常通知（afterThrowing advice），它会根据约定织入流程中，需要弄明白它们在流程中的顺序和运行的条件。
- 目标对象（target）：即被代理对象，例如，约定编程中的HelloServiceImpl实例就是一个目标对象，它被代理了。
- 引入（introduction）：是指引入新的类和其方法，增强现有Bean的功能。•织入（weaving）：它是一个通过动态代理技术，为原有服务对象生成代理对象，然后将与切点定义匹配的连接点拦截，并按约定将各类通知织入约定流程的过程。
- 切面（aspect）：是一个可以定义切点、各类通知和引入的内容，Spring AOP将通过它的信息来增强Bean的功能或者将对应的方法织入流程。

![image-20201105184753689](https://gitee.com/adambang/pic/raw/master/20201106090633.png)

Spring 提供了两种方式来生成代理对象: JDKProxy 和 Cglib，具体使用哪种方式 生成由
AopProxyFactory 根据 AdvisedSupport 对象的配置来决定。默认的策略是如果目标类是接口，
则使用 JDK 动态代理技术，否则使用 Cglib 来生成代理。

**JDK 动态接口代理**   

JDK 动态代理主要涉及到 java.lang.reflect 包中的两个类： Proxy 和 InvocationHandler。
InvocationHandler 是一个接口，通过实现该接口定义横切逻辑，并通过反射机制调用目标类
的代码，动态将横切逻辑和业务逻辑编制在一起。 Proxy 利用 InvocationHandler 动态创建
一个符合某一接口的实例，生成目标类的代理对象。  

**CGLib 动态代理**  

CGLib 全称为 Code Generation Library，是一个强大的高性能， 高质量的代码生成类库，
可以在运行期扩展 Java 类与实现 Java 接口， CGLib 封装了 asm，可以再运行期动态生成新
的 class。和 JDK 动态代理相比较： JDK 创建代理有一个限制，就是只能为接口创建代理实例，
而对于没有通过接口定义业务方法的类，则可以通过 CGLib 创建动态代理  

# 5.Spring事务传播行为

事务传播行为用来描述由某一个事务传播行为修饰的方法被嵌套进另一个方法时事务如何传播。

**Spring 中七种事务传播行为**

![image-20201106091654165](https://gitee.com/adambang/pic/raw/master/20201106091654.png)