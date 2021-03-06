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

**IoC 容器设计：**

使用 `ApplicationContext`，它是 `BeanFactory` 的子类，更好的补充并实现了 `BeanFactory` 的功能。

BeanFactory 按需加载bean，而 ApplicationContext 则在启动时加载所有bean。因此，BeanFactory与ApplicationContext相比是轻量级的。

# 3.解释Bean的生命周期

**首先总结下如何记忆 Spring Bean 的生命周期：**

> - 首先是实例化、属性赋值、初始化、销毁这 4 个大阶段；
> - 再是初始化的具体操作，有 Aware 接口的依赖注入、BeanPostProcessor 在初始化前后的处理以及 InitializingBean 和 init-method 的初始化操作；
> - 销毁的具体操作，有注册相关销毁回调接口，最后通过DisposableBean 和 destory-method 进行销毁。

**Bean创建的三个阶段**

Spring在创建一个Bean时是分为三个步骤的

- 实例化，可以理解为new一个对象
- 属性注入，可以理解为调用setter方法完成属性注入
- 初始化，你可以按照Spring的规则配置一些初始化的方法（例如，`@PostConstruct`注解）

SpringBean和我们通常说的对象有什么区别？

> SpringBean是**由SpringIoC容器管理**的，是一个**被实例化，组装，并通过容器管理的对象**，可通过getBean()获取。**容器通过读取配置的元数据，解析成BeanDefinition，注册到BeanFactory中，加入到singletonObjects缓存池中**。

Bean的生命周期的过程，它大致分为

1. 实例化，创建一个Bean对象
2. 属性赋值，为属性赋值
3. 初始化

- 如果实现了`xxxAware`接口，通过不同类型的Aware接口拿到Spring容器的资源

> ​    若 Spring 检测到 bean 实现了 Aware 接口，则会为其注入相应的依赖。所以通过让bean 实现 Aware 接口，则能在 bean 中获得相应的 Spring 容器资源。
> Spring 中提供的 Aware 接口有：
>
> - BeanNameAware：注入当前 bean 对应 beanName；
> - BeanClassLoaderAware：注入加载当前 bean 的 ClassLoader；
> - BeanFactoryAware：注入 当前BeanFactory容器 的引用。

- 如果实现了BeanPostProcessor接口，则会回调该接口的`postProcessBeforeInitialzation`和`postProcessAfterInitialization`方法

> 常用场景有：
>
> - 对于标记接口的实现类，进行自定义处理。例如ApplicationContextAwareProcessor，为其注入相应依赖；再举个例子，自定义对实现解密接口的类，将对其属性进行解密处理；
> - 为当前对象提供代理实现。例如 Spring AOP 功能，生成对象的代理类，然后返回。

- 如果配置了`init-method`方法，则会执行`init-method`配置的方法

4. 销毁

   > - 容器关闭后，如果Bean实现了`DisposableBean`接口，则会回调该接口的`destroy`方法
   > - 如果配置了`destroy-method`方法，则会执行`destroy-method`配置的方法

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

# 6.Spring如何解决循环依赖的

[烂大街的Spring循环依赖该如何回答？](https://app.yinxiang.com/shard/s35/nl/7513061/f8aad314-b7fc-4408-a013-5537c49990dc)

循环依赖有三种：

1. 原型模式循环依赖（无法解决）；
2. 单例bean循环依赖，构造参数产生循环依赖（无法解决）；
3. 单例bean循环依赖，setter产生循环依赖（可以解决）

| 依赖情况               | 依赖注入方式                                                 | 是否解决 |
| :--------------------- | :----------------------------------------------------------- | :------- |
| AB相互依赖（循环依赖） | 均采用setter方法注入                                         | 是       |
| AB相互依赖（循环依赖） | 均采用属性自动注入                                           | 是       |
| AB相互依赖（循环依赖） | 均采用构造器注入                                             | 否       |
| AB相互依赖（循环依赖） | A中注入B的方式为setter方法，B中注入A的方式为构造器           | 是       |
| AB相互依赖（循环依赖） | B中注入A的方式为setter方法，A中注入B的方式为构造器,Spring在创建Bean时默认会根据自然排序进行创建，A会先于B进行创建 | 否       |

Spring通过三级缓存解决了循环依赖，Spring是通过**「三级缓存」**来解决上述问题的：

- `singletonObjects`：一级缓存 存储的是所有创建好了的单例Bean
- `earlySingletonObjects`：早期曝光对象 完成实例化，但是还未进行属性注入及初始化的对象
- `singletonFactories`：提前暴露的一个单例工厂，二级缓存中存储的就是从这个工厂中获取到的对象

当A、B两个类发生循环引用时，在A完成实例化后，就使用实例化后的对象去创建一个`对象工厂`，添加到三级缓存中，如果A被AOP代理，那么通过这个工厂获取到的就是A`代理后`的对象，如果A没有被AOP代理，那么这个工厂获取到的就是A实例化的对象。

当A进行属性注入时，会去创建B，同时B又依赖了A，所以创建B的同时又会去调用getBean(a)来获取需要的依赖，此时的getBean(a)会从缓存中获取：

> 第一步：先获取到三级缓存中的工厂；
>
> 第二步：调用对象工工厂的getObject方法来获取到对应的对象，得到这个对象后将其注入到B中。紧接着B会走完它的生命周期流程，包括初始化、后置处理器等。
>
> 第三步：当B创建完后，会将B再注入到A中，此时A再完成它的整个生命周期。至此，循环依赖结束！

# 7.Springboot自动装配流程

当我们的SpringBoot项目启动的时候，会先导入AutoConfigurationImportSelector，这个类会帮我们选择所有候选的配置，我们需要导入的配置都是SpringBoot帮我们写好的一个一个的配置类，那么这些配置类的位置，存在与META-INF/spring.factories文件中，通过这个文件，Spring可以找到这些配置类的位置，于是去加载其中的配置。ps.@ConditionalOnXXX:如果其中的条件都满足，该类才会生效。

![img](https://gitee.com/adambang/pic/raw/master/20210114161638.jpeg)

# 8.SpringBoot启动流程 

![img](https://gitee.com/adambang/pic/raw/master/20210114164556.png)