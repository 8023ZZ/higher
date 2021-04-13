### 对 Spring 的 IOC 的理解
控制反转，依赖注入

tomcat 启动的时候会直接启动 spring 容器，spring 容器根据 xml 配置或者注解，去实例化bean对象（@Controller、@Service、@Component、@Configuration+@Bean），然后根据 xml 配置或者注解(@Autowired、@Rescource)进行以来注入

底层实现原理是通过反射对类创建对象，完成类与类之间的彻底解耦

注入接口的原因：1.注入接口更加灵活 2.注入实现类会使用CGLIB代理，注入接口使用JDK动态代理，提高了实现类的扩展性和可替换性

![avatar](/static/springIOC.png)

IOC 初始化过程：
1. tomcat 启动时会触发容器初始化事件
2. Spring 的 contextLoaderListner 监听到这个事件后会执行 contextInitialized 方法，方法中会初始化 Spring 的根容器即 IOC 容器，初始化完成后，会将这个容器放到 servletContext 中以便获取
3. tomcat 在启动过程中还会去扫描加载 servlet，比如 spring mvc 中的dispatcher servlet（前端控制器），用来处理每一个servlet
4. servlet 一般采用延迟加载，当请求过来时发现 dispatcher servlet 还未初始化，则会调用它的 init 方法，初始化的时候会建立自己的容器 spring mvc，同时spring mvc 容器通过 servletcontext 上下文来获取到 spring ioc 容器，将ioc 容器设置为父容器
5. spring mvc 中容器可以访问 spring ioc 容器中的bean，反之不能，即在controller 中可以注入 service bean，但是service 中不能注入 controller bean

### 对 Spring 的 AOP 的理解
Aspect，通过动态代理的方式来生成其代理对象在方法执行前后加入自己的逻辑，针对的是 Spring 容器管理的 Bean

代理方式：JDK + CGLIB

AOP 分为两类：
* 静态 AOP，在编译阶段对程序源代码进行更改，生成了静态的AOP代理类（生成的*.class文件已经被修改），比如 AspectJ
* 动态 AOP，在运行阶段动态生成代理对象，JDK + CGLIB，如Spring AOP

Spring AOP 不能拦截对对象字段的修改，也不支持构造器连接点，无法在 Bean 创建时通知应用

AspectJ 本身是不支持运行期织入的，日常使用时候，我们经常回听说，spring 使用了aspectJ实现了aop，听起来好像spring的aop完全是依赖于aspectJ，其实spring对于aop的实现是通过动态代理（jdk的动态代理或者cglib的动态代理），它只是使用了aspectJ的Annotation，并没有使用它的编译期和织入器，Spring AOP 只支持方法级别的切面

### JDK 动态代理和 CGLIB 动态代理
动态创建一个代理类，然后创建这个代理类的实例对象，在实例对象里引用原来的类的方法，代理类负责进行一些代码上的增强

JDK 动态代理是生成一个同样接口的代理类，构造一个实例对象

CGLIB 动态代理是直接生成类的子类，可以动态生成字节码，覆盖原来的方法，在方法里加入加强的代码

CGLIB 创建动态代理对象比 JDK 动态代理对象的性能高，但是创建对象的时间长

### Spring 中的 Bean 是线程安全的吗
Spring 容器中的 bean 可以分为5个范围：
* singleton：默认，每个容器中只有一个 bean 实例
* prototype：为每一个 bean 请求提供一个实例
* request：每一个网络请求创建一个实例，在请求完成后bean失效并被GC
* session：每个 session 中有一个 bean 实例，在 session 过期后 bean 失效
* global-session

99.99% 的情况下都是用singleton单例作用域，是线程不安全的

### Spring 中的事务以及事务传播机制
如果使用 @Transcational 注解，此时Spring 会开启 AOP，方法执行之前先开启事务，执行完毕之后根据方法是否报错来决定回滚还是提交事务

主要分为三类，A和B需要在同一事务中；A和B不能在同一事物中；嵌套型事务 NESTED

* PROPAGATION_REQUIRED: 如果当前没有事务，就创建一个新事物，如果当前存在事务，那么就加入该事务。这个设置为默认设置
* PROPAGATION_SUPPORTS: 支持当前事务，如果当前存在事务，就加入该事务，如果不存在该事务，就以非事务执行
* PROPAGATION_MANDATORY：支持当前事务，如果当前存在事务，就加入该事务，不存在事务就抛出异常
* PROPAGATION_REQUIRES_NEW: 创建新事务，无论当前是否存在事务
* PROPAGETION_NOT_SUPPORT: 以非事务方式执行操作，如果当前存在事务，就把当前事务挂起
* PROPAGATION_NEVER：以非事务方式执行，如果当前存在事务，则抛出异常
* PROPAGATION_NESTED: 如果当前存在事务，则在嵌套事务内执行，如果当前没有事务，则按REQUIRED属性执行

嵌套事务，外层事务如果回滚会导致内层事务也会滚，但是内层事务如果回滚，仅仅回滚自己的代码

私有方法不能开启事务，因为 JDK 代理不支持 private 方法。JDK的动态代理，只有被动态代理直接调用时才会产生事务。在SpringIoC容器中返回的调用的对象是代理对象而不是真实的对象。而方法中的 this 是真实对象而不是代理对象。

### Spring 核心架构
Spring 核心点主要是IOC和AOP，核心技术是反射和代理
spring bean 生命周期，从创建和初始化bean->使用->销毁（两个回调函数）
1. 实例化Bean
对于ApplicationContext 容器，当容器启动结束后，根据xml配置或者注解，通过获取 BeanDefination，通过反射实例化所有Bean
createBeanInstance() 
2. 设置对象属性（依赖注入）
populateBean() -> 装载属性赋值
根据Bean依赖关系，bean注入
3. 处理 Aware 接口
如果这个Bean实现了 ApplicationContextAware接口，spring 容器会调用bean的setApplicationContext(ApplicationContext applicationContext) 方法，传入Spring 上下文
4. BeanPostProcessor
在bean实例构建好后，对bean进行自定义处理，通过调用postProcessBeforeInitialization(Object object, String s)
5. InitializingBean 与 init-method：
bean的初始化方法initializeBean()
6. BeanPostProcessor
调用postProcessAfterInitialization(Object object)方法
7. DisposableBean：
Bean不再被需要时进行清理，调用DiaposableBean接口的destroy()方法
8. destory-method
调用配置的销毁方法

![avatar](/static/springbean生命周期.png)

### Spring Bean 循环依赖
单例作用域的bean，通过构造器注入时会产生循环依赖问题
因为创建实例对象无法完成，而通过set/get注入不会出问题，因为先创建了实例，spring会缓存一下创建好的bean，然后再去注入到对应依赖的bean中

增加了三级缓存：
* singletonFactories ： 单例对象工厂的cache
* earlySingletonObjects ：提前暴光的单例对象的Cache
* singletonObjects：单例对象的cache

三级缓存的前提是执行了构造器，做了不完全的初始化，所以构造方法的循环依赖没法解决

### Spring 中的设计模式
* 工厂模式：所有的bean实例都放在spring容器（工厂）中，如果使用bean直接通过spring容器就可以了
* 工厂方法模式：如果将应用程序自己的工厂对象交给Spring管理,那么Spring管理的就不是普通的bean,而是工厂Bean
* 单例模式：bean单例模式，双重校验锁
* 代理模式：AOP
* 模板方法模式：定义一个操作中的算法的骨架，而将一些步骤延迟到子类中。 模板方法使得子类可以不改变一个算法的结构即可重定义该算法的某些特定步骤的实现方式（AQS）。Spring 中 jdbcTemplate、hibernateTemplate 等以 Template 结尾的对数据库操作的类，它们就使用到了模板模式。一般情况下，我们都是使用继承的方式来实现模板模式，但是 Spring 并没有使用这种方式，而是使用Callback 模式与模板方法模式配合，既达到了代码复用的效果，同时增加了灵活性。
* 观察者模式：观察者模式是一种对象行为型模式。它表示的是一种对象与对象之间具有依赖关系，当一个对象发生改变的时候，这个对象所依赖的对象也会做出反应。Spring事件驱动模型=定义(监听者+发布者+事件)
* 适配器模式：适配器模式(Adapter Pattern) 将一个接口转换成客户希望的另一个接口，适配器模式使接口不兼容的那些类可以一起工作，其别名为包装器(Wrapper)。DispatcherServlet根据请求信息调用 HandlerMapping，解析请求对应的 Handler。解析到对应的 Handler（也就是我们平常说的 Controller 控制器）后，开始由HandlerAdapter 适配器处理。HandlerAdapter 作为期望接口，具体的适配器实现类用于对目标类进行适配，Controller 作为需要适配的类。
* 装饰器模式：装饰者模式可以动态地给对象添加一些额外的属性或行为。相比于使用继承，装饰者模式更加灵活。简单点儿说就是当我们需要修改原有的功能，但我们又不愿直接去修改原有的代码时，设计一个Decorator套在原有代码外面。其实在 JDK 中就有很多地方用到了装饰者模式，比如 InputStream家族，InputStream 类下有 FileInputStream (读取文件)、BufferedInputStream (增加缓存,使读取文件速度大大提升)等子类都在不修改InputStream 代码的情况下扩展了它的功能。Spring 中配置 DataSource 的时候，DataSource 可能是不同的数据库和数据源。我们能否根据客户的需求在少修改原有类的代码下动态切换不同的数据源，这个时候就要用到装饰者模式。Spring 中用到的包装器模式在类名上含有 Wrapper或者 Decorator。这些类基本上都是动态地给一个对象添加一些额外的职责

### SpringMVC 的核心架构
1. tomcat工作线程将请求转交给 Spring MVC 框架的DispatcherServlet（前端控制器）
2. DispatcherServlet 查找 @Controller 注解的 controller，通过配饰器模式定位对应的controller
3. 根据@RequestMapping定位对应的方法
4. 调用方法进行处理
5. 返回处理：
    jsp： 返回页面模板名称，spring mvc使用模板技术对前端页面进行渲染
    json： 直接返回json对象
6. 把渲染以后的 html 页面返回给浏览器进行显示

### Spring IOC 的初始化过程
ApplicationContext 继承自 beanFactory，但是他不是 BeanFactory 的实现类，而是说其内部持有了一个 BeanFactory。以后所有跟 BeanFactory 相关的操作其实是委托给这个实例来进行处理
1. ClassPathXmlApplicationContext 的 refresh（）方法，refresh 方法可以重建 ApplicationContexxt，会将原来的 ApplicationContext 销毁，然后再重新执行初始化操作
2. obtailFreshBeanFactory ，初始化 BeanFactory，解析 BeanDefinatiion，注册到 BeanFactory 中
3. preparebeanFactory，手动注册一些特殊的Bean
4. 设置 BeanFactory 的类加载器
5. BeanDefinitionRegistryPostProcessor 可以动态注册 bean
6. Bean 如果实现了 BeanFactoryPostProcessor，会调用 postProcessorBeanFactory，此时所有的 Bean 都加载并注册完成了，但是还没开始初始化。可以添加一些特殊的 BeanFactoryPostProcessor 的实现类或者做些什么事
7. 注册 BeanPostProcessor
8. 初始化 ApplicationContext 的 MessageSource
9. 初始化 ApplicationContext 的事件广播器
10. 模版模式钩子方法 onRefresh，具体的子类可以在这里初始化一些特殊的 Bean （在初始化 singleton beans 之前）
11. 注册事件监听器，实现 ApplicationListener 接口
12. finishBeanFactoryInitialization（beanFactory）- createBeanInstance实例化所有 singleton beans，lazy-init 除外
13. populateBean 方法进行属性设值，处理依赖
14. 执行回调，如果实现了 Aware 接口，回调invokeAwareMethods方法，回调 BeanPostProcessor 的 postProcessBeforeInitialization 方法 ，执行 init-method，回调 BeanPostProcessor 的 postProcessAfterInitialization 方法
15. 广播事件 ApplicationContext 初始化完成

### 动态代理底层原理
（1）用户通过Proxy.newProxyInstance方法，传入ClassLoader、接口数组、和InvocationHandler实现类（包含具体被代理对象和对其具体处理逻辑）；

（2）底层根据接口数组和InvocationHandler在运行时生成代理类字节码，即代理类实现接口数组，同时组合InvocationHandler，对被代理类添加额外功能；

（3）然后通过传入的ClassLoader进行加载，再实例化后返回，即得到代理对象。