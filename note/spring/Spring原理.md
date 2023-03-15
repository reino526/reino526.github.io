# SpringIOC



## BeanFactory和ApplicationContext

​	**BeanFactory**是**ApplicationContext**的父接口，它才是主要的**核心容器**，主要的ApplicationContext都【**组合**】了它的功能，BeanFactory实际上会是ApplicationContext里的一个**成员变量**，如果调用如getBean这种方法时实际上是调用这个**成员变量里的getBean方法**

​	BeanFactory表面上只有getBean方法，实际上**控制反转**、基本的**依赖注入**、**Bean的生命周期**的各种功能都是由**它的实现**提供，即**DefaultListableBeanFactory**，这个类还继承了很多类，其中有一个**DefaultSingletonBeanRegistry**类里有一个**singletonObjects**字典变量用来存放单例bean。扩展：私有变量可以使用**反射**获取其值，代码如下（用过滤使只显示以component为开头的类）：

```java
Field singletonObjects = DefaultSingletonBeanRegistry.class.getDeclaredField("singletonObjects");
singletonObjects.setAccessible(true);
ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
Map<String,Object> map = (Map<String,Object>)singletonObjects.get(beanFactory);
map.entrySet().stream().filter(stringObjectEntry -> stringObjectEntry.getKey().startsWith("component"))
    .forEach(stringObjectEntry -> System.out.println(stringObjectEntry.getKey() + "=" + stringObjectEntry.getValue())
);
```

​	Application实现了四个主要提供功能的接口，分别是**MessageSource**、**ResourcePatternResolver**、**EnvironmentCapable**和**ApplicationEventPublisher**

​	**MessageSource**主要是负责**国际化信息翻译**，在**message.propeties**中会存放有不同语言通用的信息，而**messages_en.properties**则存放的是英文相关的翻译，在ApplicationContext对象中使用**getMessage**方法传入对应的键名和**Locale**的枚举，如中文就是**Locale.CHINA**，测试代码如下（hi为自己定义的翻译信息，即自定义的message.propeties）

```java
context.getMessage("hi",null, Locale.CHINA);
```

​	**ResourcePatternResolver**接口主要是负责**通配符匹配资源**的获取，对应在ApplicationContext中是**getResources**方法，下面是使用代码

```java
Resource[] resources = context.getResources("classpath*:META-INF/spring.factories");
for (Resource resource : resources) {
    System.out.println(resource);
}
```

​	**EnvironmentCapable**接口提供**获取配置信息**的方法，包括系统环境变量、application.yml之类的配置信息，对应在ApplicationContext中是**getEnvironment**方法，使用代码如下

```java
System.out.println(context.getEnvironment().getProperty("java_home"));
Systen.out.println(context.getEnvironment().getProperty("server.port"));
```

​	**ApplicationEventPublisher**接口是负责**发布事件**的，对应的是ApplicationContext中**publishEvent**方法，在自定义继承**ApplicationEvent**类的事件类后，可以使用这个方法来发布，然后在一个**参数为该事件类型**的方法中加上**@EventListener**注解就可以将这个方法作为**监听这个事件**的方法



## BeanFactory实现

​	BeanFactory的重要实现就是之前提到的**DefaultListableBeanFactory**，但是其实beanfactory类**不提供注释解析**等功能，这些功能主要是通过**后处理器**来实现的，**添加后处理器**是通过使用**AnnotationConfigUtils**工具类里的**registerAnnotationConfigProcessors**方法将后处理器注册到beanFactory中，然后使用beanfactory创建bean需要使用其**registerBeanDefinition**方法传入**beanDefinition**，而beanDefinition的定义需要用到**BeanDefinitionBuilder**里的静态方法**genericBeanDefinition**，测试代码如下

```java
// 新建一个BeanFactory对象
DefaultListableBeanFactory beanFactory = new DefaultListableBeanFactory();
// bean的定义（class，scope，初始化，销毁），使用
AbstractBeanDefinition beanDefinition = BeanDefinitionBuilder.genericBeanDefinition(String.class).setScope("singleton").getBeanDefinition();
beanFactory.registerBeanDefinition("string",beanDefinition);

// 在beanFactory里面添加一些常用的后处理器
AnnotationConfigUtils.registerAnnotationConfigProcessors(beanFactory);

// 运行所有的后处理器（根据父接口BeanFactoryPostProcessor获取所有的BeanFactory后处理器）
beanFactory.getBeansOfType(BeanFactoryPostProcessor.class).values().stream().forEach(
    beanFactoryPostProcessor -> {
    beanFactoryPostProcessor.postProcessBeanFactory(beanFactory);
});
```

​	**BeanFactory**的后处理器只可以解析类似于**@Bean**这样的注解，而如果要解析**@Autowired**这种注解需要使用**Bean**的后处理器，运行的时候是使用**beanFactory**的**addBeanPostProcessor**方法，测试代码如下

```java
// Bean后处理器，针对bean的生命周期的各个阶段提供扩展，例如@Autowired、@Resource...
beanFactory.getBeansOfType(BeanPostProcessor.class).values().forEach(beanFactory::addBeanPostProcessor);
```

​	一般对于单例对象都是**延迟创建**，即**第一次使用**时才会创建，在这之前只会有bean的**定义信息**，如果想提前创建好可以使用**preInstantiateSingletons**方法提前创建BeanFactory中的**所有单例**对象

​	可以得出，**BeanFactory不会做**的事：

​		1.不会主动调用BeanFactory后处理器，

​		2.不会主动**添加Bean后处理器**，

​		3.不会主动**初始化单例**，

​		4.不会主动**解析注释**包括**${}**和**#{}**

​	所以BeanFactory是一个**比较基础**的类，而这些事会在实现了BeanFactory接口的**ApplicationContext类**里**一起做**

扩展：

​	**registerBeanDefinition**方法**按顺序**注册的后处理器，如果冲突了**只会执行先注册的**后处理器，由于处理**@Autowired**的后处理器**AutowiredAnnotationBeanPostProcessor**比处理**@Resource**的后处理器**CommonAnnotationBeanPostProcessor**要**先注册**，所以若两个注解冲突**以@Autowired的为准**，当然这个是由于**使用了流的处理**顺序（即按顺序），可以通过流的**sorted**方法**更改顺序**进行处理，而**排序的比较器**可以使用beanFactory的**getDependencyComparator**方法获取，这个比较器是在**registerBeanDefinition**方法执行的时候**就设置进去**的，这个比较器会通过每个注解的**order**值进行排序，而**使用了这个比较器**后是**@Resource为准**



## ApplicationContext实现

​	**ClassPathXmlApplicationContext**实现类中需要通过**XmlBeanDefinitionReader**类去读取xml配置文件里的信息，将其传给BeanFactory创建bean，模拟代码如下

```java
// 新建BeanFactory
DefaultListableBeanFactory beanFactory = new DefaultListableBeanFactory();
// 读取xml配置文件的类，将读取到的bean定义信息直接传给BeanFactory
XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(beanFactory);
// 真正读取
reader.loadBeanDefinitions(new ClassPathResource("application.xml"));
```

​	在**FileSystemXmlApplicationContext**实现类里实际上就是将**ClassPathResource**改成**FileSystemResource**

​	在**AnnotationConfigApplicationContext**实现类中直接就把上面BeanFactory需要的**后处理器添加好了**不需要再进行配置了，而**前两种**实现都需要**在xml配置文件**配置**形如\<context:anntation-config/\>**的配置项才能**启用这些后处理器**

​	在**AnnotationConfigServletWebApplicationContext**实现类里由于是web项目，需要对web进行一些配置，所以需要像下面这样写配置类

```java
@Configuration
public class WebConfig {
    // web服务器工厂，建立了内嵌的Tomcat服务器（web容器就是指这个）
    @Bean
    public ServletWebServerFactory servletWebServerFactory(){
        return new TomcatServletWebServerFactory();
    }

    // 前控制器，所有访问请求通过它来进行请求分发到不同的控制器
    @Bean
    public DispatcherServlet dispatcherServlet(){
        return new DispatcherServlet();
    }

    // 将前控制器注册到服务器里，路径为/即应用于所有路径的请求
    @Bean
    public DispatcherServletRegistrationBean registrationBean(DispatcherServlet dispatcherServlet){
        return new DispatcherServletRegistrationBean(dispatcherServlet,"/");
    }
}
```

​	

## Bean的生命周期

​	bean生命周期里是先执行**构造方法**，然后执行**依赖注入**，然后是**初始化**，最后就是**销毁**，bean的后处理器可以重写6个方法，分别代表着在**销毁前**执行（如**@PreDestroy**）、**实例化前**执行、**实例化后**执行（返回**false**会**跳过依赖注入**）、**依赖注入时**执行（@Autowired、@Value、@Resource）、**初始化前**执行（**@PostConstruct**、**@ConfigurationProperties**）、**初始化**后执行（代理增强）

​	bean的周期中使用了**模板方法**，将**不能确定**或者**可拓展**的部分**比如后处理器**的执行使用了**for循环集合**加**接口**来实现，使代码可扩展性大大增强，**初级软件实作**里的**实体管理器**就是这样搞的



## Bean的后处理器

​	**AutowiredAnnotationBeanPostProcessor**主要是解析**@Autowired**和**@Value**注解，这个如果beanFactory使用的是**默认**的**AutowireCandidateResolver**则会报错**无法执行**，因为**不支持@Value**，需要使用**ContextAutowireCandidateResolver**

​	**CommonAnnotationBeanPostProcessor**主要是解析**@Resource**、**@PostConstruct**、**@PreDestroy**注解

​	**ConfigurationPropertiesBindingPostProcessor**主要是解析**@ConfigurationProperties**，前面几个注册方法可以使用**ApplicationContext**的**registerBean**方法将**类的字节码传入**，而这个需要使用**自己**该类的**静态方法register**将ApplicationContext里的**BeanFactory传入**



### **@Autowired**和**@Value**的解析

​	测试代码

```java
// 新建BeanFactory
DefaultListableBeanFactory beanFactory = new DefaultListableBeanFactory();
// 注册几个bean，这里省略
// ...
// 设置AutowireCandidateResolver，使@Value可以正常运行
beanFactory.setAutowireCandidateResolver(new ContextAnnotationAutowireCandidateResolver());
// 设置${}的解析器
beanFactory.addEmbeddedValueResolver(new StandardEnvironment()::resolvePlaceholders);

// 新建AutowiredAnnotationBeanPostProcessor后处理器
AutowiredAnnotationBeanPostProcessor processor = new AutowiredAnnotationBeanPostProcessor();
// 将BeanFactory与其绑定
processor.setBeanFactory(beanFactory);
// 执行依赖注入，解析@AutoWired和@Value，需要传入bean，这里是乱写
processor.postProcessProperties(null,beanObject,"beanName");
```

调用**postProcessProperties**方法时第一个调用的方法就是**findAutowiringMetadata**，如下所示

```java
InjectionMetadata metadata = this.findAutowiringMetadata(beanName, beanObject.getClass(), pvs);
```

**findAutowiringMetadata**会将该bean中所有加了**@Autowired**或**@Value**注解的方法或属性**都存入InjectionMetadata**对象并返回，调用该对象的**inject**方法即可**完成依赖注入**

```java
metadata.inject(beanObject,"beanName",null);
```

inject方法中需要实现**按类型查找**值，测试代码如下

```java
// 测试前提：假设一个Bean类，里面有一个需要注入的属性bean1，还有一个方法setBean2的第一个bean2参数也需要注入
// 获取类里的bean1属性
Field bean1 = Bean.class.getDeclaredField("bean1");
// 创建一个注入的依赖描述,第二个参数是设置是否是必须的，必须的如果找不到就会报错
DependencyDescriptor descriptor = new DependencyDescriptor(bean1, false);
// 然后通过这个依赖描述可以调用BeanFactory的doResolveDependency方法中进行查找符合依赖描述可以注入的bean，最后返回值就是这个对象
Object o = beanFactory.doResolveDependency(descriptor, null, null, null);

// 对于方法的里参数的注入则需要另一种方法获取，后面的参数为要提取的这个方法里参数的类型
Method setBean2 = Bean.class.getDeclaredMethod("setBean2", Bean2.class);
// 创建和这个依赖描述的时候就需要传入一个MethodParameter对象，指定是哪个方法里的第几个参数
DependencyDescriptor descriptor1 = new DependencyDescriptor(new MethodParameter(setBean2, 0), false);
// 同样在BeanFactory中进行查找符合依赖描述可以注入的bean并返回
Object o = beanFactory.doResolveDependency(descriptor1, null, null, null);
```



## BeanFactory的后处理器

​	**ConfigurationClassPostProcessor**主要是解析**@ComponentScan**、**@Bean**、**@Import**、**@ImportResource**注解，不管是BeanFactory还是Bean的**后处理器**，都会在ApplicationContext里每次执行**refresh**的时候**执行**



### @Component的解析

​	对@Component的解析原理的测试代码如下（模拟其实现）

```java
// 在类里找是否有指定的注解
ComponentScan componentScan = AnnotationUtils.findAnnotation(Config.class, ComponentScan.class);
// 如果有，可以对注解的属性进行遍历
if(componentScan != null){
    // 这里使用的是@ComponentScan注解，所以遍历其需要扫描包的值
    for (String p : componentScan.basePackages()) {
        // 假设这里得到的p的类名为com.example.spring_high.component，在这里就需要转化为
        // classpath*:com/example/spring_high/component/**/*.class去加载类
        String path = "classpath*:" + p.replace(".","/") + "/**/*.class";
        // 通过前面提到的ApplicationContext里实现的ResourcePatternResolver接口可以获取这些类
        Resource[] resources = context.getResource(path);
        // 一个可以读取类的元信息的工厂
        CachingMetadataReaderFactory readerFactory = new CachingMetadataReaderFactory();
        // 创建一个可以为创建的bean生成默认名字的对象，在注册bean的时候可以使用
        AnnotationBeanNameGenerator nameGenerator = new AnnotationBeanNameGenerator();
        for (Resource resource : resources) {
            // 读取每一个类的元信息，MetadataReader可以读取很多类的元信息，这个类里的方法不走类加载，效率比反射要高
            MetadataReader reader = readerFactory.getMetadataReader(resource);
            // 可以通过reader.getAnnotationMetadata()获取到注解的元信息
            // 可以通过reader.getClassMetadata()获取到类的元信息
            // 判断是否加了@Component注解
            boolean b = reader.getAnnotationMetadata().hasAnnotation(Component.class.getName());
            // 判断是否加了@Component其派生的注解
            boolean b1 = reader.getAnnotationMetadata().hasMetaAnnotation(Component.class.getName());
            // 如果存在上面说的两种情况之一就注册这个bean到容器里
            if(b || b1){
                // 创建BeanDefinition
                AbstractBeanDefinition beanDefinition = BeanDefinitionBuilder
                        .genericBeanDefinition(reader.getClassMetadata().getClassName())
                        .getBeanDefinition();
                // 获取BeanFactory
                DefaultListableBeanFactory beanFactory = context.getDefaultListableBeanFactory();
                // 生成一个名字
                String beanName = nameGenerator.generateBeanName(beanDefinition, beanFactory);
                // 注册bean
                beanFactory.registerBeanDefinition(beanName,beanDefinition);
            }
        }
    }
}
```



### @Bean的解析

​	对@Bean的解析原理的测试代码如下（模拟其实现）

```java
 // 新建读元信息的工厂
    CachingMetadataReaderFactory readerFactory = new CachingMetadataReaderFactory();
    // 将需要解析@Bean注解的类传进去得到元数据（这里用了硬编码，主要是为了模拟，源码中应该是循环获取来解析）
    MetadataReader reader = readerFactory.getMetadataReader(new ClassPathResource("com/example/spring_high/config/Config.class"));
    // 获取所有含有@Bean注解的方法元数据
    Set<MethodMetadata> methods = reader.getAnnotationMetadata().getAnnotatedMethods(Bean.class.getName());
    for (MethodMetadata method : methods) {
        // 对于bean里的属性解析可以通过getAnnotationAttributes方法获取属性值，这里只是解析了initMethod的属性值
        String initMethod = method.getAnnotationAttributes(Bean.class.getName()).get("initMethod").toString();

        // 这里的bean不是直接注册类，而是通过工厂方法，先创建一个builder
        BeanDefinitionBuilder builder = BeanDefinitionBuilder.genericBeanDefinition();
        // 调用setFactoryMethodOnBean方法设置工厂方法，第二个参数是工厂名字是谁，即拥有这个工厂方法的bean的名字
        builder.setFactoryMethodOnBean(method.getMethodName(),"config");
        // 设置工厂可以使用自动装配，即工厂的参数可以注入
        builder.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_CONSTRUCTOR);
        // 如果配置了初始化方法的名字，就使用初始化的方法
        if(initMethod.length() > 0){
            builder.setInitMethodName(initMethod);
        }
        // 然后就可以获取beanDefinition
        AbstractBeanDefinition beanDefinition = builder.getBeanDefinition();
        // 注册bean，这里省略beanName的生成，可以和上面@Conponent注解的做法一样
        // ...生成beanName
        context.getDefaultListableBeanFactory().registerBeanDefinition(beanName,beanDefinition);
    }
}
```



## Aware与InitializingBean接口

​	**Aware**接口用于注入一些与**容器相关信息**，**BeanNameAware**实现就是注入**bean的名字**，**BeanFactoryAware**实现注入**BeanFactory容器**，**ApplicationContextAware**实现注入**ApplicationContext容器**，**EmbeddedValueResolverAware**实现**解析${}**

​	**InitializingBean**接口实现后的那个方法会在该类初始化的时候调用

​	虽然上述**Aware**接口实现的**也可以用@Autowired**实现，但是@Autowired的使用需要**用到bean的后处理器**，属于**扩展**功能，在一些**特殊情况会失效**，而Aware接口属于**内置功能**，不加任何扩展也能够被Spring识别，**不会失效**，**InitializingBean**接口也是**一样**

​	特殊情况例子：若**类里定义**了一个**BeanFactoryPostProcessor**的bean，本来**应该先执行BeanFactoryPostProcessor解析注解**的但是由于类还**没实例化**这个**类里**的BeanFactoryPostProcessor**执行不了**，只能**提前实例化**即**跳过了**内置BeanFactoryPostProcessor**解析注解**的阶段导致@Autowired等注解失效，但是这种情况下使用**Aware**和**InitializingBean**接口的方法就**不受干扰**，所以一般**Spring框架内部的类**经常使用它们



## Bean的初始化与销毁

​	**三种初始化**方法按**执行顺序**排列分别是：**@PostConstruct**、**InitializingBean接口**的实现方法、**@Bean的属性InitMethod**指定的初始化方法，而**Aware接口**的方法是在**@PostConstruct和InitializingBean接口之间**进行



## Scope

​	目前有五种：**singleton**（单例）、**prototype**（多例）、**request**（请求域，请求开始时创建，请求结束销毁）、**session**（会话域）、**application**（应用程序域，指servletContext）

​	**Scope失效**解决：当一个**单例类里@Autowired**的是其它scope的就会**有失效的情况**，比如**如果是prototype**由于单例对象**依赖注入仅发生一次**，所以用的**一直是同一个**，解决方法是在@Autowired头上**再加一个@Lazy**注解，代理是同一个，但是**每次调用代理的方法**时会**由代理创建新的**被代理的对象，**另一个解决方法**是在**prototype**的那个类里的**@Scope注解**再加一个**属性proxyMode = ScopedProxyMode.TARGET_CLASS**，**也是**一样是**依靠代理**，其实还有**很多解决方法**，比如每次**调用get方法实质**上是通过**工厂方法返回**一个**新对象**，或者直接**注入ApplicationContext从上下文里拿**，用**代理**会有**性能损耗**，所以**尽量**可以使用**不用**代理的方法



# SpringAOP



## 编译阶段实现增强（ajc增强）

​	如果装上**aspectj-maven-plugin**插件可以使用**aspectj编译器**增强（简称acj增强），这种增强**不是基于代理**的，而是**直接在编译时**在要被增强的方法中直接**修改源代码**，**添加调用通知的代码**，这种增强**可以用于static静态方法的增强**，因为static方法**不能够被子类重写**所以**基于代理的**AOP**做不到**，另外**@Aspect**这种注解是**和Spring无关**的，是aspectj的



## 类加载阶段实现增强（agent增强）

​	运行时在VM options中加入 **-javaagent:C:/User/H/.m2/repository/org/aspectj/aspectjweaver/1.9.7/aspectjweaver-1.9.1.jar**，就可以实现在类加载阶段增强，一样**也是不基于代理**，也是修改源代码，但是在**运行过程中**才能**看到被修改的源代码**，这种情况下就算是**在被增强的方法里再调用另一个被增强的方法**也**能够增强**，打破了一般增强的限制



## 基于代理实现增强（proxy增强）

​	分为**jdk代理**和**cglib代理**，前者只能针对接口进行代理



### 基于jdk的代理

​	**目标**对象与**代理**对象是**兄弟**关系，目标对象**可以是final类**，即**没有子类**，简单实现如下

```java
// 需要增强的目标对象
Target target = new Target();

// 由于代理是在运行中直接生成的字节码，需要传入一个加载器加载这个字节码
ClassLoader classLoader = TestJdkProxy.class.getClassLoader();
// 新建代理对象，参数2为该代理对象实现的接口,是一个数组可以同时实现多个接口，返回的对象就是代理
Foo proxy = (Foo) Proxy.newProxyInstance(classLoader, new Class[]{Foo.class}, new InvocationHandler() {
    // 这个封装具体增强的行为，代理对象被执行时就会调用这个方法
    // 三个参数分别为代理对象本身、正在执行的方法对象，方法传过来的实际参数
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("before");
        // 调用目标的方法
        Object result = method.invoke(target, args);
        // 返回目标方法返回的内容
        return result;
    }
});
// 在代理上执行增强后的目标方法
proxy.foo();
```

​	**模拟**jdk**生成的代理**的源代码如下

```java
public class $Proxy0 implements Foo{
    // 这里实际的代理对象会继承一个Proxy的父类，而在这个父类里面就已经有InvocationHandler这个成员变量了
    private InvocationHandler h;
	// 定义需要增强的方法，真正的代理类里还会含有toString、hasnCode等Object的方法
    static Method foo;

    static {
        try {
            // 获取方法
            foo = Foo.class.getMethod("foo");
        } catch (NoSuchMethodException e) {
            throw new NoSuchMethodError(e.getMessage());
        }
    }

    public $Proxy0(InvocationHandler h) {
        // 在真正的代理中这里可以直接调用父类构造函数super(h)
        this.h = h;
    }

    @Override
    public Integer foo() {
        try {
            // 实际调用时就直接调用接口实现的方法
            return (Integer)h.invoke(this, foo, new Object[0]);
        } catch (RuntimeException | Error e) {
            // 对于运行时错误则直接抛出
            throw e;
        } catch (Throwable e){
            // 把检查异常转换为运行时异常抛出，因为在这个接口方法声明的地方可能没有实现里抛出的那种Throwable异常
            throw new UndeclaredThrowableException(e);
        }
    }
}

// 定义一个接口，在创建时可以通过实现这个接口的方法传入具体增强的内容
interface InvocationHandler{
    Object invoke(Object proxy,Method method,Object[] args) throws Throwable;
}
```

​	实际上代理的生成是**直接生成字节码**，并**没有通过编译**的过程，这种直接生成字节码的技术叫做**ASM**，在**运行阶段动态生成字节码**，在**Spring框架**里**大量使用**了这种技术，里面是用到**ClassWriter**这个类来**写字节码**存到一个**byte数组**并返回

```java
// 通过ASM技术获取到动态生成了字节码以后，这里假设在dump里面，就要进行类加载
// 这里的动态生成主要是设置成员变量，成员函数、static代码块，因为这些是需要定制的，而增强的方法是在生成代理对象时才需要设置
// 创建一个类加载器（真正的jdk代理中应该是使用了最初传入的类加载器，这里只是单独指加载字节码）
ClassLoader loader = new ClassLoader(){
    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        // 加载类的字节码数组，假设已经存在dump里
        return super.findClass(name,dump,0,dump.length);
    }
};
// 加载器调用loadClass方法才是真正的加载，返回了代理的这个类
Class<?> proxyClass = loader.loadClass("com.example.spring_high.test.$Proxy0");
// 获得代理类的构造器，参数为构造函数所需要的参数类型
Constructor<?> constructor = proxyClass.getConstructor(InvocationHandler.class);
// 构造方法拿到了，创建实例，创建的参数就是源代码里实现的InvocationHandler接口，这里就不写了
Foo proxy = (Foo) constructor.newInstance(new InvocationHandler() {
    @Override
    public Integer invoke(Object proxy, Method method, Object[] args) throws Throwable {
        return null;
    }
});
// 最终就能运行了
proxy.foo();
```

​	jdk会**优化反射调用**，当**同一个方法反射调用多次**时，会动态创建一个**代理类**，只为是**正常调用这个方法**，减少消耗



### 基于cglib的代理

​	**目标**对象与**代理**对象是**父子**关系，目标对象**不可以是final类**，要增强的**方法也不能是final**的，代理可以转化为目标对象的类型，简单实现如下

```java
// 目标对象
Target target = new Target();
// 创建代理对象
Target proxy = (Target) Enhancer.create(Target.class, new MethodInterceptor() {
    // 要增强的具体行为，第一个参数是代理自己，第二个参数是代理类执行的方法，第三个参数是方法执行时的参数
    @Override
    public Object intercept(Object o, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
        System.out.println("before");
        // 调用目标方法
        Object result = method.invoke(target, args);
        // 下面这种版本内部不会用到反射调用，反射调用性能会差一点
        // Object result = methodProxy.invoke(target, args);
        // 下面这种也是没有用到反射，但是不需要传入目标对象而是传入代理对象就可以了
        // Object result = methodProxy.invokeSuper(o, args);
        return result;
    }
});
// 从代理调用目标方法
proxy.foo();
```

​	**模拟**cglib**生成的代理**的源代码与jdk类似，最重要的主要是**多了一个MethodProxy**可以不进行反射调用，模拟代码如下

```java
public class Proxy extends Target {

    private MethodInterceptor methodInterceptor;

    public Proxy(MethodInterceptor methodInterceptor) {
        this.methodInterceptor = methodInterceptor;
    }

    static Method foo;
    // 相比较jdk为每一个方法多定义了一个静态成员变量，可以称作为方法代理
    static MethodProxy fooProxy;
    static {
        try {
            foo = Target.class.getMethod("foo");
            // 除了获取了一般这种方法，还得获取一种不反射调用的方法，使用MethodProxy的静态方法create获取，第一个参数是目标类
            // 型，第二个参数是代理的类型，第三个参数()表示无参V表示返回类型为void， 第四个参数是增强的方法，第五个是原始方法
            fooProxy = MethodProxy.create(Target.class,Proxy.class,"()V","foo","fooSuper");
        } catch (NoSuchMethodException e) {
            throw new NoSuchMethodError(e.getMessage());
        }
    }

    // 带原始功能的方法
    public void fooSuper(){
        super.foo();
    }

    // 带增强功能的方法
    @Override
    public void foo() {
        try {
            // 第四个参数就传入静态代码块了获取到的方法代理
            methodInterceptor.intercept(this,foo,new Object[0],fooProxy);
        } catch (Throwable e) {
            throw new UndeclaredThrowableException(e);
        }
    }
}
```



#### MethodProxy原理	

​	对于**MethodProxy**的**生成**是使用了一个**FastClass**的类，这个类也是在运行中**动态生成字节码**的，对于MethodProxy的**invoke**和**invokeSuper**分别创建了**两个代理类**，下面用代码**模拟**一下

```java
// 这个类主要是对于MethodProxy的invoke方法，即参数为目标的方法
// 主要只模拟FastClass里的getIndex和invoke方法的实现
public class TargetFastClass {

    // 这里的方法签名不需要在这里定义，在MethodProxy的create时就传入
    // 同一个类的方法使用MethodProxy的create时会在同一个类里加上一个，即维护同一个TargetFastClass对象
    static Signature s0 = new Signature("foo","()V");

    // 这个方法主要是获取目标方法的编号
    // Signature是方法的签名信息，主要包括方法名字、参数返回值
    // MethodProxy的create方法里的后面几个参数就是为了生成目标方法的签名
    public int getIndex(Signature signature){
        if(s0.equals(signature)){
            return 0;
        }
        // 如果有s1可以返回1
        return -1;
    }

    // 在真正运行的时候，在最外层调用的那个MethodProxy.invoke会知道是哪个签名，然后通过getIndex获取对应的index
    // 根据方法编号正常调用目标方法对象
    public Object invoke(int index,Object target,Object[] args){
        // 根据index调用不同的方法
        if (index == 0){
           return ((Target)target).foo();
        }
        else {
            throw new RuntimeException("无此方法");
        }
    }
}
```

```java
// 这个类主要是对于MethodProxy的invokeSuper方法，即参数为目标的方法
// 主要只模拟FastClass里的getIndex和invoke方法的实现
public class ProxyFastClass {
    // 这里得换
    static Signature s0 = new Signature("fooSuper","()V");

    // 这个方法主要是获取代理方法的编号
    public int getIndex(Signature signature){
        if(s0.equals(signature)){
            return 0;
        }
        // 如果有s1可以返回1
        return -1;
    }

    // 根据方法编号在代理中调用原始方法对象
    public Object invoke(int index,Object proxy,Object[] args){
        // 根据index调用不同的方法
        if (index == 0){
           return ((Proxy)proxy).fooSuper();
        }
        else {
            throw new RuntimeException("无此方法");
        }
    }
}
```


## Spring选择代理

​	**两个切面**概念，**aspect**是通知（advice）+切点（pointcut）可以**有多组**，**advisor**是更细粒度的切面，包含**一个通知和切点**，在最终**生效之前**，aspect会**被拆解成advisor**

​	**Spring最终使用Advisor创建代理**的过程**模拟**代码如下

```java
// 1.备好切点（spring有许多可以定义切点的类，这里就选用aspectj表达式，切点就是筛选需要增强的方法）
AspectJExpressionPointcut pointcut = new AspectJExpressionPointcut();
pointcut.setExpression("execution(* foo())");
// 2.备好通知（spring也有许多定义通知的类，这里使用的这个是最重要的，很多通知最终都会转化为这个类型）
MethodInterceptor advice = new MethodInterceptor() {
    // 注意这个接口不是cglib包里的，是另一个同名接口
    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        System.out.println("before");
        Object result = invocation.proceed();  // 调用目标
        System.out.println("after");
        return result;
    }
};
// 3.备好切面（选择advisor一个比较简单的实现）
DefaultPointcutAdvisor advisor = new DefaultPointcutAdvisor(pointcut,advice);
// 4.创建代理（spring提供了一个很方便的代理工厂类ProxyFactory）
ProxyFactory factory = new ProxyFactory();
// 设置创建目标
factory.setTarget(target);
// 设置切面
factory.addAdvice(advice);
// 创建代理对象
Foo proxy = (Foo) factory.getProxy();
// 调用代理增强后的方法
proxy.foo();
```

​	Spring选择**不同方法创建代理**的依据：

​		a.	**proxyTargetClass** = **false**且目标**实现了接口**，用**jdk**实现

​		b.	proxyTargetClass = **false**且目标**没有实现**接口，用**cglib**实现

​		c.	proxyTargetClass = **true**，总是用**cglib**实现

​	像上面的**模拟代码**里**虽然目标对象实现了接口**但是**ProxyFactory不知道**，所以最终会**使用cglib**方法创建，需要使用**下面这句**才能**告诉它**目标对象**实现了**哪些接口

```java
factory.setInterfaces(target.getClass.getInterfaces());
```

​	而如果再加**下面这句**就会变成**cglib**实现（c情况）

```java
factory.setProxyTargetClass(true);
```



## SpringAOP切点匹配

​	对于上面使用的**AspectJExpressionPointcut**切点类可以使用其**matches**方法传入方法判断其**是否符合筛选条件**，**实际上**当增强的时候就是**逐个方法**都去**判断**，但是这个切点类**只能筛选**在**方法上的注解**，Spring的**@Transactional**注解是可以**标注在类上**代表整个类里的方法都开启事务，所以**Spring自己实现**了另一个**切点类**，下面模拟Spring的切点匹配

```java
// 继承StaticMethodMatcherPointcut类并实现它的接口，这里是模拟，Spring主要就是实现了MethodMather接口
StaticMethodMatcherPointcut pt = new StaticMethodMatcherPointcut() {
    // 只需要实现matches方法后就可以进行筛选了，参数一是方法，参数二是方法所在类
    @Override
    public boolean matches(Method method, Class<?> targetClass) {
        // 这里使用了Spring的工具类MergedAnnotations，判断类或方法是否含有指定的注解
        MergedAnnotations annotations = MergedAnnotations.from(method);
        // 判断是否含有@Transactional注解
        if (annotations.isPresent(Transactional.class)){
            return true;
        }
        // 判断类上是否加了@Transactional注解
        // 第二个参数是搜索策略，默认是DIRECT，不会搜索父类或者实现的接口，而下面指定的就会
        annotations = MergedAnnotations.from(targetClass, MergedAnnotations.SearchStrategy.TYPE_HIERARCHY);
        if(annotations.isPresent(Transactional.class)){
            return true;
        }
        return false;
    }
};
```



## @Aspect和Advisor

​	@Aspect是**高级**的切面，Advisor是**低级**切面，高级切面最终会**转换为低级**切面，当使用了**@Aspect**和**@After**之类的这种**高级切面的注解**，就需要用到另一个**bean的后处理器**来解析，这个后处理器就是**AnnotationAwareAspectJAutoProxyCreator**，下面通过**主动调用方法来理解**这个后处理器

```java
// 从容器里获取到这个后处理器主动调用其方法，主要是了解其中的两个关键方法
AnnotationAwareAspectJAutoProxyCreator creator = context.getBean(AnnotationAwareAspectJAutoProxyCreator.class);
// 调用其findEligibleAdvisors方法（这个方法是受保护在外面调用不了的，这里只是形式一下）
// 这个方法传入一个类，然后在容器中所有的advisor里面找出有资格增强这个类里的方法的advisor
// 结果包含的都是低级切面，方法里会将高级切面转换成低级切面，第二个参数是bean的名字
List<Advisor> advisors = creator.findEligibleAdvisors(Target.class, "target");
// 这个函数内部会调用findEligibleAdvisors方法来判断是否需要为这个类创建代理，返回的集合为空就不创建
// 如果创建就返回代理对象，如果不创建就啥也不干返回目标对象，参数里的目标对象在实际中会由Spring容器传进来
Object o = creator.wrapIfNecessary(new Target(), "target", "target");
```

​	这两个方法可以看出这个后处理器**解析高级切面注解并自动创建代理**，它是在bean生命周期里的**依赖注入前**和**初始化后**调用，当**无循环依赖**的时候（即你中有无我中有你）则是在**初始化后**创建代理，但是如果**有循环依赖**的话就需要**提前创建**即在依赖注入前创建并**暂存于二级缓存**中，所以依赖注入的方法和初始化的方法不应该被增强，因为这两个方法属于是准备bean，一般是准备好bean以后才能增强

​	对于执行顺序**高级切面**可以用**@Order**注解来设置，但是这个注解只有**用在@Aspect下**面有用，即**不能单独**对于每个切面类里的**方法**进行排序，**只能是不同切面类之间**的顺序，而**低级切面**如果是**实现了Order接口**的advisor就可以**调用setOrder方法**来设置优先级



### 高级切面转换为低级切面

​	如下面的模拟代码

```java
// 高级切面转低级切面（Aspect类是自定义一个使用了@Aspect注解的切面类）
// 一个切面的实例工厂，这里选用的是单例的实例化工厂
AspectInstanceFactory factory = new SingletonAspectInstanceFactory(new Aspect());
// 遍历切面类里的所有方法
for (Method method : Aspect.class.getDeclaredMethods()) {
    // 这里只转换了有@Before注解的方法，其余的是类似的，如果方法上含有该注解
    if (method.isAnnotationPresent(Before.class)){
        // 获取其中的切点表达式
        String expression = method.getAnnotation(Before.class).value();
        // 创建一个切点
        AspectJExpressionPointcut pointcut = new AspectJExpressionPointcut();
        // 传入切点表达式
        pointcut.setExpression(expression);
        // 根据不同的通知类型选用不同的类，这里是创建了前置通知类，
        // 第一个参数是通知具体方法，第二个参数是切点（主要是为了解析表达式可能带有的其他条件参数）
        // 第三个参数是将来切面的实例工厂，需要实例才能反射调用
        AspectJMethodBeforeAdvice advice = new AspectJMethodBeforeAdvice(method,pointcut,factory);
        // 创建切面，将切点和通知传入
        DefaultPointcutAdvisor advisor = new DefaultPointcutAdvisor(pointcut, advice);
    }
}
```



### 统一转换为环绕通知

​	转换为低级切面然后对它们**进行排序后**就**通过ProxyFactory创建代理对象**，会将所有的通知**转换为环绕通知**，因为无论是哪种方法创建代理，最终**都是MethodInvocation**对象干活，都需要**实现MethodInterceptor接口**，而里面的方式就是环绕，对于**环绕通知**它**已经实现了**MethodInterceptor接口就**不需要转换**了，转换的方法是通过**ProxyFactory**里的一个方法**getInterceptorsAndDynamicInterceptionAdvice**，如下

```java
// 第一个参数是目标对象的方法（因为是环绕通知，要在通知里调用方法的），第二分参数是目标类的类型
List<Object> methodInterceptorList = proxyFactory.getInterceptorsAndDynamicInterceptionAdvice(Target.class.getMethod("foo"), Target.class);
```

​	这里的**转换**体现的是**设计模式**中的**适配器模式**，对外是为了**方便**使用要**区分**before、afterReturning，对内统一都是环绕通知**统一使用MethodInterceptor表示**

**MethodBeforeAdviceAdapter**适配器将@Before的**AspectJMethodBeforeAdvice**适配为**MethodBeforeAdviceInterceptor**

**AfterReturningAdviceAdapter**适配器将**AspectJAfterReturningAdvice**适配为**AfterReturningAdviceInterceptor**



### 创建并执行调用链

​	调用链主要由**环绕通知和目标**组成，主要由**MethodInvocation的实现类**负责，模拟代码如下

```java
// 有几个实现，最基本的就是这个ReflectiveMethodInvocation，核心代码都在这里
// 第一个参数是代理，简化起见设为null，第二个参数是目标对象，第三个参数是目标的方法
// 第四个参数是目标方法的实参，第五个参数是目标对象类型，第六个参数就是前面转换好的环绕通知集合
MethodInvocation methodInvocation = new ReflectiveMethodInvocation(null,target,Target.class.getDeclaredMethod("foo"),new Object[0],Target.class,methodInterceptorList);
// 创建好调用链以后，就可以调用它的proceed方法来执行，是一个递归调用过程
methodInvocation.proceed();
// 然后还需要将这个methodInvocation放在一个公共的位置，因为每个advice调用时都需要获取这个对象
// 这个公共位置就是当前线程，放入当前线程这个操作其实也是一个环绕通知，就最外层的通知
// 这种通知Spring已经写好了一个，只需要将其放入ProxyFactory最外层即最先添加就行了
// 就是ExposeInvocationInterceptor这个通知，直接调用INSTANCE就可以获取实例
// proxyFactory.addAdvice(ExposeInvocationInterceptor.INSTANCE);
```

​	下面用代码模拟一下**调用链的执行**，**自己实现**的一个**MethodInvocation**

```java
static class MyInvocation implements MethodInvocation{

    private Object target;  // 目标
    private Method method;  // 目标方法
    private Object[] args;  // 目标方法的实参
    List<MethodInterceptor> methodInterceptorList;  // 要调用的通知列表
    private int count = 1;  // 当前调用次数

    // 构造器
    public MyInvocation(Object target, Method method, Object[] args, List<MethodInterceptor> methodInterceptorList) {
        this.target = target;
        this.method = method;
        this.args = args;
        this.methodInterceptorList = methodInterceptorList;
    }

    // 返回目标方法
    @Override
    public Method getMethod() {
        return method;
    }

    // 返回方法实参
    @Override
    public Object[] getArguments() {
        return args;
    }

    // 调用每一个环绕通知，调用目标
    @Override
    public Object proceed() throws Throwable {
        // 如果调用次数已经大于调用链长度
        if(count > methodInterceptorList.size()){
            // 调用目标，返回被结束递归
            return method.invoke(target, args);
        }
        // 获取需要调用的通知并使调用次数增加
        MethodInterceptor methodInterceptor = methodInterceptorList.get(count++ - 1);
        // 调用通知（这里返回给上一层 下一层的执行结果）
        return methodInterceptor.invoke(this);;
    }

    // 返回目标对象
    @Override
    public Object getThis() {
        return target;
    }

    @Override
    public AccessibleObject getStaticPart() {
        return method;
    }
}

// 给一个通知作对照，理解怎么递归
static class Advice implements MethodInterceptor{
    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        System.out.println("before");
        // 每个通知都会获取这个MethodInvocation对象再次proceed一次，所以是递归
        Object result = invocation.proceed();
        System.out.println("after");
        return result;
    }
}
```



### 动态通知调用

​	**前面**说的**都是静态**通知调用，**动态**通知调用需要**参数绑定**，即看**通知方法有没有参数**就可以区分，**切点表达式**里面也会**有一个args**参数绑定的语句，**静态**在**执行时不需要切点**，**动态需要**，而且动态的**性能消耗比较大**，对于**动态的通知**会生成的是**InterceptorAndDynamicMethodMatcher**的实例，它**本身没有实现MethodInterceptor**接口，但是它**有一个MethodInterceptor的成员变量**，同时**还有一个MethodMatcher的成员变量**，这就是动态通知执行时需要**切点和环绕通知组合**起来执行，因为需要**动态的解析参数而确定切点**



# SpringMVC



​	SpringMVC**需要的配置**

```java
@Configuration
@ComponentScan
public class WebConfig {
    // 内嵌容器工厂
    public TomcatServletWebServerFactory tomcatServletWebServerFactory(){
        return new TomcatServletWebServerFactory();
    }
    // 创建DispatcherServlet
    @Bean
    public DispatcherServlet dispatcherServlet(){
        return new DispatcherServlet();
    }
    // 注册DispatcherServlet，Spring MVC的入口
    @Bean
    public DispatcherServletRegistrationBean dispatcherServletRegistrationBean(DispatcherServlet dispatcherServlet){
        DispatcherServletRegistrationBean registrationBean = new 
            DispatcherServletRegistrationBean(dispatcherServlet, "/");
        return registrationBean;
    }
}
```



## DispatcherServlet的初始化



### 初始化时机

​	**DispatcherServlet**虽然是由**Spring来创建**，但是只有当首次使用的时候才会由**Tomcat服务器初始化**，如果在注册DispatcherServlet的时候调用

```java
registrationBean.setLoadOnStartup(1);
```

​	就可以实现Tomcat服务器**一启动**的时候**就初始化**DispatcherServlet，数字大于0就可以（默认值是-1），如果有**多个DispatcherServlet**数字代表着优先级，**数字越小的优先级越高**，然后对于在yml里配置的值挨个**用@Value读取太繁琐**，可以像下面这样

```java
@EnableConfigurationProperties({WebMvcProperties.class, ServerProperties.class})
```

​	**WebMvcProperties**会自动读取**以spring.mvc开头**的配置项，ServerProperties则会读取以server开头的配置项，所以上上那一句可以**改成这样**（**webMvcProperties**对象可以**成员变量**然后**注入**或者直接加在**函数的参数**里）

```java
registrationBean.setLoadOnStartup(webMvcProperties.getServlet().getLoadOnStartup());
```



### 初始化过程

​	DispatcherServlet的**初始化**主要就是调用其**onRefresh方法**，该方法里又调用了一个**initStrategies**方法，其源代码如下

```java
protected void initStrategies(ApplicationContext context) {
    this.initMultipartResolver(context);  // 初始化文件上传解析器
    this.initLocaleResolver(context);  // 初始化本地化信息解析器（使用哪国语言）
    this.initThemeResolver(context);
    this.initHandlerMappings(context);  // 初始化路径映射解析器（重点）
    this.initHandlerAdapters(context);  // 初始化不同控制器方法调用的适配器（重点）
    this.initHandlerExceptionResolvers(context);  // 初始化异常解析器
    this.initRequestToViewNameTranslator(context);
    this.initViewResolvers(context);
    this.initFlashMapManager(context);
}
```

​	所有初始化的方法其实都是**类似**的，首先先是在**当前容器**或者**及其父容器**（**过度**设计）里找**有没有实现类**，如果没找到就会去一个**DispatcherServlet.properties**文件里读取**默认的实现的类**



## RequestMappingHandlerMapping

​	建立**请求路径**和**控制器**方法之间的一个**映射**关系，就是**通过@RequestMapping**注解建立的联系，在RequestMappingHandlerMapping**初始化的时候**就会把这些**信息收集**起来，下面是测试代码

```java
// 按默认读取配置文件的话会将其直接存入DispatcherServlet的成员变量里，不会在容器里，所以得先用@Bean搞进去，这里省略
// 作用：解析@RequestMapping以及派生注解，生成路径与控制器方法的映射关系，在初始化时就生成
RequestMappingHandlerMapping handlerMapping = context.getBean(RequestMappingHandlerMapping.class);
// 这个方法获取映射结果，RequestMappingInfo是路径和请求方式的信息，HandlerMethod是控制器方法信息
Map<RequestMappingInfo, HandlerMethod> handlerMethods = handlerMapping.getHandlerMethods();
// 若请求来了，获取控制器方法，参数为一个http的请求，返回处理器执行链对象，里面除了包含控制器方法还有拦截器方法
HandlerExecutionChain chain = handlerMapping.getHandler(httpServletRequest);
```



## RequestMappingHandlerAdapter

​	这个就是去**调用控制器方法**的，通过**invokeHandlerMethod**方法**调用**上面提到的**HandlerMethod**

```java
// 调用控制器方法，第一个参数是http请求，第二个参数是http回应，第三个参数就是HandlerMethod对象，可以用前面获得的chain获得
handlerAdapter.invokeHandlerMethod(request,response,(HandlerMethod)chain.getHandler());
```

​	而如果像**@RequestParam**这种注解**不是由RequestMappingHandlerAdapter来解析**的，是由它里面的**其他解析器对象**完成解析，其它注解包括@RequestBody、@CookieValue等等都是**由不同的解析器完成的**，**handlerAdapter.getArgumentResolvers()**可以获取到**所有参数解析器**的列表，**handlerAdapter.getReturnValueHandlers()**可以得到所有**返回值处理器**的列表



### 自定义参数解析器

​	自定义一个注解**@Token**

```java
// 用途：可以获取请求头里token的值
// 可以加在方法的参数前面
@Target(ElementType.PARAMETER)
// 运行过程中都有效
@Retention(RetentionPolicy.RUNTIME)
public @interface Token {
}
```

​	解析器本体

```java
// @Token的解析器，解析器需要实现HandlerMethodArgumentResolver接口
public class TokenArgumentResolver implements HandlerMethodArgumentResolver {
    @Override
    // 判断是否支持某个参数
    public boolean supportsParameter(MethodParameter parameter) {
        // 判断这个参数上是否有@Token注解
        Token token = parameter.getParameterAnnotation(Token.class);
        return token != null;
    }

    @Override
    // 解析注解
    public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer,
                                  NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {
        // 获取请求头里的值
        String token = webRequest.getHeader("token");
        // 返回就行了
        return token;
    }
}
```

​	然后需要在**RequestMappingHandlerAdapter**中添加这个自定义的解析器

```java
// 使用setCustomArgumentResolvers方法添加自定义的解析器，参数是一个List
handlerAdapter.setCustomArgumentResolvers(List.of(new TokenArgumentResolver()));
```



### 自定义返回值处理器

​	这里注解的定义就省去了，注解是加在**方法上的@Yml**

```java
// 作用是将返回的java对象转成yml的格式，需要实现HandlerMethodReturnValueHandler接口
public class YmlReturnValueHandler implements HandlerMethodReturnValueHandler {
    @Override
    public boolean supportsReturnType(MethodParameter returnType) {
        // 通过方法参数拿到方法上的注解，判断是否支持
        Yml yml = returnType.getMethodAnnotation(Yml.class);
        return yml != null;
    }

    @Override
    // 这里这个ModelAndViewContainer参数就是mvc容器，储存中间结果的
    public void handleReturnValue(Object returnValue, MethodParameter returnType, ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {
        // 用第三方包将返回的结果转化为yaml字符串
        // 这里使用第三方包，省略...转化结果存到str里
        // 拿到原始的响应对象
        HttpServletResponse response = webRequest.getNativeResponse(HttpServletResponse.class);
        // 设置编码格式刷
        response.setContentType("text/plain;charset=utf-8");
        // 将yaml字符串写入响应
        response.getWriter().print(str);
        // 设置请求已经处理完毕（不然还得去视图解析器然后找视图）
        mavContainer.setRequestHandled(true);
    }
}
```



## 参数解析器

​	下面只挑了**RequestParamMethodArgumentResolver**一个解析器进行测试

```java
// 数据绑定工厂，这个是默认实现，可以自动转换成需要的类型
DefaultDataBinderFactory factory = new DefaultDataBinderFactory(null);
// 解析@RequestParam注解的解析器，第二个是是否可以省略注解也能解析
// 第一个参数是BeanFactory这里定为null，主要是为了拿值，容器里的值或者环境和配置里的值，而且包括${}和#{}的解析器
RequestParamMethodArgumentResolver requestParamResolver
        = new RequestParamMethodArgumentResolver(null,true);
// 判定支持这个参数后对这个参数调用方法
if(requestParamResolver.supportsParameter(parameter)){
    // 支持此参数，第二个参数就是临时存储信息的ModelAndViewContainer容器，返回的值就是解析的结果
    // 第四个参数是数据绑定工厂BinderFactory，因为request里面的值都是字符串，可以使用这个转换为所需要的类型
    Object v = requestParamResolver.resolveArgument(parameter, container, request, factory);
}
```

​	挨个**挨个**这样调用解析器**太麻烦**了，所以Spring写了一个类**HandlerMethodArgumentResolverComposite**，用于**调用一堆解析器**，实现的是一种叫做**组合器**的**设计模式**，下面是测试代码

```java
// 创建这个组合模式的对象
HandlerMethodArgumentResolverComposite composite = new HandlerMethodArgumentResolverComposite();
// 添加解析器
composite.addResolvers(
        new RequestParamMethodArgumentResolver(null,true)
);
// 省略中间可能的过程...
// 然后在对参数判断时不需要用单个解析器的supportsParameter方法，执行也是不需要
if(composite.supportsParameter(parameter)){
    // 像这样就可以了
    Object v = composite.resolveArgument(parameter, container, request, factory);
}
```

​	添加的**顺序要注意**，主要是有一些解析器可以**不加注解也能解析**，这些**要放在最后**不然会导致**提前判断支持**参数而**轮不到**后面**本来应该执行**的解析器解析，一般来说**RequestParamMethodArgumentResolver可省略**的版本要放在**最后**

​	扩展：解析**@ModelAttribute**注解的解析器需要将**模型数据存在**一个**ModelAndViewContainer**里面，可以调用其**getModel**方法来**查看**目前容器里含有什么模型数据，然后里面的**每个模型**数据都会**有一个id**，如果在**注解中没有设置**值会**默认为类名的小写**



### 拓展：获取参数名

​	**一般编译**情况下编译出来的字节码再反编译是**不会还存在参数的名称**的，会用var1 var2代替，在用**javac编译**的时候加上**-parameters**选项再通过**用javap -c -v反编译**可以看到**参数名**会被**存**在一个**MethodParameters**的地方，这个地方的参数名**可以**通过使用**反射来获取**，另一种方法是在javac编译的时候加**-g**选项，同样用javap**反编译**可以看到**参数名会存**在**LocalVariableTable**的地方，这个地方就是**本地变量表**，这里的值**反射获得不了**，但是**可以用ASM获取**

​	在Spring中实现了一个类可以获取**LocalVariableTable**里的值，这个类就是**LocalVariableTableParameterNameDiscoverer**，使用方法就是**实例化以后**调用它的**getParameterNames**方法，这个方法需要**传入Method方法**对象，然后可以**得到参数名的字符串数组**

​	然后Spring里有一个**DefaultParameterNameDiscoverer**类将**两种方法都结合**了，不管是**使用哪种方法**存下来的参数名**都可以获取**，还有就是**本地变量表不能保存接口**上的参数名，**另一种方法可以**，在**HanderMethod**类里调用**getMethodParameters**方法获取该方法的所有参数**MethodParameter**以后可以对每个参数使用其类里的**initParameterNameDiscovery**方法设置**DefaultParameterNameDiscoverer**类，就能够使用**getParameterName**方法获取到参数名字



## 对象绑定与类型转换

​	**底层第一套转换接口（Spring提供的）：**

-  **Printer**把**其它类型转为String**

- Parser**把**String转为其它类型**

- Formatter**综合**Printer和Parser**功能

- Converter**把**类型S转为类型T**

- Printer、Parser、Converter经过**适配转换为GenericConverter**放入**Converters集合**


​	**FormattingConversionService**利用它们**实现转换**



​	**底层第二套转换接口（jdk提供的）：**

- **PropertyEditor**把**String与其它类型相互转换**
- PropertyEditorRegistry**可以**注册多个PropertyEditor**对象（和上面的集合意义是一样的）

- 与第一套接口**直接可以**通过FormatterPropertyEditorAdapter**来进行**适配**


​	现在Spring中是**两套接口并存**的，应该是**历史遗留**的原因，**一开始用jdk**的发现功能**不够全面**，就**自己实现**了一套**更复杂的**接口



​	**高层接口与实现：**

- 四大实现都是**实现了TypeConverter**这个高层转换接口，转换时会**用到TypeCOnverterDelegate**委派**ConversionService与PropertyEditorRegistry执行真正的转换**，这里用到了**Facade门面模式**的设计模式

- 执行过程中**首先**看有没有**自定义的转换器**，比如**@InitBinder**添加的就是自定义，自定义添加的会**用适配器把Formatter转为需要的ProperEditor**，然后会看有没有**ConversionService转换**，然后再利用**默认的PropertyEditor转换**，最后有一些**特殊处理**

- **SimpleTypeConverter**仅做**类型转换**
- **BeanWrapperImpl**为**bean的属性赋值**（比如从**配置文件里**的字符串获取**属性值**来赋值 ，**底层是反射**），当**需要时做类型转换**，走**Property**（**get和set方法**赋值）
- **DirectFieldAccessor**为**bean**的属性赋值，当**需要时**做类型转换，走**Field** （**私有成员变量**赋值）
- **ServletRequestDataBinder**为**bean**的属性执行绑定，当**需要时**做类型转换，根据**directFieldAccess**（这是一个**布尔变量**，为**真就走Field**）**选择走Property还是Field**，具备**校验**与**获取校验结果**功能

下面是四大实现类的一些测试代码

```java
// 一.仅有类型转换的功能
SimpleTypeConverter typeConverter = new SimpleTypeConverter();
// 照这样使用
Integer number = typeConverter.convertIfNecessary("123", int.class);


// 二.利用反射原理，为bean的属性赋值（创建时传入要赋值的bean）
BeanWrapperImpl wrapper = new BeanWrapperImpl(new Bean());
// 对属性赋值（第一个参数是变量名，第二个是指，需要进行转换时会自动转换）
wrapper.setPropertyValue("a","1");


// 三.利用反射原理，为bean的属性赋值（创建时传入要赋值的bean）
DirectFieldAccessor accessor = new DirectFieldAccessor(new Bean());
// 对属性赋值（第一个参数是变量名，第二个是指，需要进行转换时会自动转换），没有get和set方法也能赋值
accessor.setPropertyValue("a","1");


// 四.执行数据绑定器（非web环境下）
DataBinder dataBinder = new DataBinder(new Bean());
// 准备一个原始数据
MutablePropertyValues pvs = new MutablePropertyValues();
// 设置原始数据，需要进行转换时会自动转换
pvs.add("a","1");
// 可以设置directFieldAccess变量控制走Property还是Field，默认是假，调用这个方法后会改为真
dataBinder.initDirectFieldAccess();
// 绑定数据
dataBinder.bind(pvs);

// 五.执行数据绑定器（web环境下的）
ServletRequestDataBinder requestDataBinder = new ServletRequestDataBinder(new Bean());
// 准备一个原始数据（这里是通过http请求创建的原始数据）
MockHttpServletRequest request = new MockHttpServletRequest();
// 设置原始数据，需要进行转换时会自动转换
request.setParameter("a","1");
// 绑定数据（在这里才通过设置好的http请求创建好原始数据）
dataBinder.bind(new ServletRequestParameterPropertyValues(request));
// 那个@ModelAttribute注解的效果就是由这个转换器实现的（将request里的参数封装为一个java对象）
```



### 自定义转换器

​	有**两种自定义**转换器的方法，第一种是使用**ConversionService接口**配合**Formatter转换器**，第二种是jdk的方法，使用**PropertyEditorRegistry接口**和**PropertyEditor转换器**，这个就是**@InitBinder转换**，下面是自定义转换器的代码

```java
//一. 用工厂，无转换功能
// 这种转换器的工厂前面参数解析器的时候就用到过，这里不拓展功能所有直接传两个null参数
ServletRequestDataBinderFactory factory = new ServletRequestDataBinderFactory(null,null);
// 用工厂创建数据绑定对象（request是http请求，bean是需要绑定的对象，第三个参数是这个对象的名字）
WebDataBinder dataBinder = factory.createBinder(new ServletWebRequest(request), bean, "bean");
// 然后就可以绑定数据
dataBinder.bind(new ServletRequestParameterPropertyValues(request));

//二.用@InitBinder转换
// 需要在一个类的方法上加上@InitBinder注解，这个方法名无要求，返回值为空，参数必须是一个DataBinder，这里模拟一下
@InitBinder
public void aaa(WebDataBinder dataBinder){
    // 扩展dataBinder的转换器，添加自定义的转换器
    // 自定义的转换器要实现Formatter接口里面填一个范型，就是string和哪种类型之间的转换
    // 这个接口主要需要实现print（类型转字符串）方法和parse（字符串转类型）方法
    dataBinder.addCustomFormatter(new MyFormatter());
    // 这调用的方法会用FormatterPropertyEditorAdapter适配器将传入的Formatter转化为PropertyEditor
}
// 方法写好后就可以在创建DataBinder工厂的时候将该方法传进去，假设上面的方法是在MyController类里定义的
// 先获取这个方法，需要用InvocableHandlerMethod来封装这个方法
InvocableHandlerMethod method = new InvocableHandlerMethod(new MyController(),
        MyController.class.getMethod("aaa", WebDataBinder.class));
// 然后在创建工厂的时候传入
ServletRequestDataBinderFactory factory = new ServletRequestDataBinderFactory(List.of(method),null);

// 三.用ConversionService转换
// 用其中一个实现FormattingConversionService
FormattingConversionService conversionService = new FormattingConversionService();
// 添加自定义转换器
conversionService.addFormatter(new MyFormatter());
// conversionService不能直接添加到DataBindFactory里面，需要包装成一个初始化器
ConfigurableWebBindingInitializer initializer = new ConfigurableWebBindingInitializer();
// 设置这个初始化器的用ConversionService转换
initializer.setConversionService(conversionService);
// 在创建DataBindFactory的时候就传第二个参数就可以了
ServletRequestDataBinderFactory factory = new ServletRequestDataBinderFactory(null,initializer);

// 四.同时加了@InitBinder和用ConversionService转换
ServletRequestDataBinderFactory factory = new ServletRequestDataBinderFactory(List.of(method),initializer);
// 这时用了@InitBinder的转换器优先级更高，前面说过顺序

// 五.使用默认的ConversionService转换
// 是前面FormattingConversionService的子类，默认实现，如果是SpringBoot可以用ApplicationConversionService实现
// 这种情况下比如是Date和String的转换，那就需要在要转换的变量上加@DateTimeFormat注解
DefaultFormattingConversionService service = new DefaultFormattingConversionService();
ConfigurableWebBindingInitializer initializer = new ConfigurableWebBindingInitializer();
initializer.setConversionService(service);
```



### 拓展：获取范型的参数

​	有两种方法，一个是用jdk的api，一个是用Spring的api，下面是实现的代码

```java
// 获取父类的类型
// 一.jdk实现的版本
Type type = StudentDao.class.getGenericSuperclass();
// 判断该类型是否含有范型参数，即有范型参数的类型应该属于ParameterizedType
if (type instanceof ParameterizedType parameterizedType){
    // 获取这个类型中所有的范型参数
    Type[] typeArguments = parameterizedType.getActualTypeArguments();
}
// 二.spring实现的版本
// 使用一个工具类GenericTypeResolver，调用其resolveTypeArguments方法并传入子类和父类的类型可获取所有范型参数
Class<?>[] arguments = GenericTypeResolver.resolveTypeArguments(StudentDao.class, BaseDao.class);
```



## @ControllerAdvice

​	这个注解是**对控制器进行增强**的，即是写在**类上**的，与它一起**配合使用**的注解有**@InitBinder**，表示**所有控制器都需要拓展@InitBinder方法下自定义的转换器**，还有**@ExceptionHandler**注解，表示**所有控制器抛出的某些异常**就由这个注解下的方法进行**统一处理**，**@ModelAtrribute**方法会使这个方法的**返回值**会成为**模型数据补充**到这个**控制器的执行过程**中，然后就是**@ControllerAdvice和AOP没有半毛钱关系**



### @InitBinder

​	这个注解加在**没有@ControllerAdvice**注解的控制器里代表**只有这个控制器**拥有这个方法，这个方法由**RequestMappingHandlerAdapter**在**控制器方法首次执行**并记录，加在**有@ControllerAdvice**注解的控制器就代表**所有控制器**都有，这个方法由RequestMappingHandlerAdapter在**初始化的时候解析**时解析并记录

​	在**RequestMappingHandlerAdapter**里有**两个成员变量**来存，一个是**initBInderCache**存放每个**局部的**InitBinder的内容，一个是**InitBinderAdviceCache**是存放**全局的**InitBinder的内容，调用**afterPropertiesSet**方法**初始化**，里面就会**解析全局的InitBinder**



### @ModelAttribute

​	加在方法**参数前的@ModelAttribute**是将**请求里的参数转换成**参数所**需要的java对象**，这时是**参数解析器来解析**，而加**@ModelAtrribute注解的方法**是将**方法的返回值存入ModelAndViewContainer**中，这时是由**RequestMappingHandlerAdapter**（就是调用方法的那个）**解析**，**用ModelFactory将其存入**ModelAndViewContainer，默认以**返回值类型的小写**作为名字

​	下面是ModelFactory的演示代码

```java
// 通过调用RequestMappingHandlerAdapter的getModelFactory方法会自动创建一个ModelFactory，但是这个方法是私有的，只是演示
ModelFactory modelFactory = adapter.getModelFactory();
// 调用modelFactory的initModel方法就可以初始化模型数据，第一个参数是请求，第二个是ModelAndViewContainer，
// 第三个是ServletInvocableHandlerMethod，它里面就会在container中补充模型数据（这里迷惑还没执行方法怎么设置的返回值）
modelFactory.initModel(request,container,handlermethod);
```



### ResponseBodyAdvice

​	这是一个接口，**配合@ControllerAdvice**使用可以**使全部的@ResponseBody的方法统一**一个**返回的格式**（途径是创建一个**Result类**，将**返回的内容转换成Result对象**，就能**统一格式**），下面是演示实现代码

```java
// 实现RequestBodyAdvice接口，范型不关心
class ControllerAdvice implements ResponseBodyAdvice<Object> {
    // 判断满足某种条件才转换
    @Override
    public boolean supports(MethodParameter returnType, Class<? extends HttpMessageConverter<?>> converterType) {
        // 如果返回值类型上或者这个控制器方法的类上没有@ResponseBody注解就不需要转换
        if (returnType.getMethodAnnotation(ResponseBody.class) != null ||
                AnnotationUtils.findAnnotation(returnType.getContainingClass(), ResponseBody.class) != null
        ) {
            // returnType.getContainingClass().isAnnotationPresent(ResponseBody.class)可以判断类有没有
            // 但是无法判断包含的，比如@RestController注解内部就包含有，需要使用Spring的工具类AnnotationUtils
            return false;
        }
        return true;
    }

    // 对返回值的增强
    @Override
    public Object beforeBodyWrite(Object body, MethodParameter returnType, MediaType selectedContentType, Class<? extends HttpMessageConverter<?>> selectedConverterType, ServerHttpRequest request, ServerHttpResponse response) {
        // 如果返回值已经是Result类型就不需要转换直接返回
        if (body instanceof Result) {
            return body;
        }
        // 否则就转换成Result对象
        return new Result(body, "200");
    }
}
```



### @ExceptionHandler

​	**DispatcherServlet**在捕获到异常的时候只是**设置了一个变量dispatchException**为这个异常而不处理，等到**后面**才调用一个**processDispatchResult方法**里**判断这个变量是否为空**，**为空就会走视图渲染**的流程，**非空**就会进行**异常处理**流程，异常处理流程就是**将所有的异常处理器遍历找到**处理该异常的处理器进行处理，下面只对其中的**ExceptionHandlerExceptionResolver**（这个异常处理器是用于解析**@ExceptionHandler**注解的，将**注解的方法当作异常处理的方法**执行）进行代码演示

```java
// 新建一个ExceptionHandlerExceptionResolver实例
ExceptionHandlerExceptionResolver resolver = new ExceptionHandlerExceptionResolver();
// 因为它在解析@ExceptionHandler时可能还需要进行其它注解的解析或者一些类型转换，所以得给他添加好这些东西
// 添加MessageConverter，可以将处理结果转换为json字符串来响应
resolver.setMessageConverters(List.of(new MappingJackson2HttpMessageConverter()));
// 添加参数解析器和返回值处理器，这里可以调用afterPropertiesSet添加默认的参数解析器和返回值处理器
resolver.afterPropertiesSet();
// 执行异常处理，第一个参数是request，第二个是response，第三个是HandlerMethod，第三个是异常
// 可以通过HandlerMethod查看到那个控制器类，看看类里有没有@ExceptionHandler注解的方法
// 如果有，将异常和那个方法参数的异常作类型匹配，如果可以处理这个异常就会反射调用这个异常处理方法
// 调用过程中那些参数解析器、返回值处理器、转换器等等都会执行最终得到理想的处理结果
resolver.resolveException(request,response,handlerMethod,e);
```

​	对于**嵌套的异常**，处理过程中会**把一层一层异常都拿出来**（使用异常的**getCause**方法获得**起因异常**）变成**异常数组去匹配**处理方法

```java
// 这种嵌套异常
Exception e = new Exception("e1", new RuntimeException("e2", new IOException("e3")));
```

​	下面是**层层拿异常**的**ExceptionHandlerExceptionResolver**里的源码

```java
ArrayList exceptions = new ArrayList();
for(Object exToExpose = exception; exToExpose != null; exToExpose = cause != exToExpose ? cause : null) {
    exceptions.add(exToExpose);
    cause = ((Throwable)exToExpose).getCause();
}
```

​	在**@ControllerAdvice**注解类中的**@ExceptionHandler**方法可以**处理所有控制器的异常**，否则只能处理自己类里的异常，而获取**这种全局的@ExceptionHandler**也是在ExceptionHandlerExceptionResolver的**afterPropertiesSet**方法里，里面会调用一个**initExceptionHandlerAdviceCache**方法，里面会**获取所有**有**@ControllerAdvice注解的bean**，然后再**遍历这些bean**找到**有@ExceptionHandler注解的方法**并**记录**下来存到**exceptionHandlerAdviceCache**成员变量里，同时这个方法还会将**实现了ResponseBodyAdvice接口**的@ControllerAdvice注解类**存入responseBodyAdvice**成员变量

​	和这个**类似的是ReuqestMAppingHandlerAdapter**的**afterPropertiesSet**方法，一样也是将类似的**类上的注解存到成员变量**里，这个**需要存的东西更多**



## 控制器方法执行流程

​	**HandlerMethod**就是需要执行的**控制器方法**

​	HandlerMethod**需要**

- **bean**，即是**哪个Controller**
- **method**，即是Controller中的**哪个方法**

​	但是HandlerMethod**不能**够**直接调用**运行，需要**封装成ServletInvocableHandlerMethod**对象

​	ServletInvocableHandlerMethod**需要**

- **WebDataBInderFactory**负责**对象绑定**、**类型转换**
- **ParameterNameDiscoverer**负责**参数名解析**
- **HandlerMethodArgumentResolverComposite**负责**解析参数**
- **HaddlerMethodReturnValueHandlerComposite**负责**处理返回值**

​	**RequestMappingHandlerAdapter**首先会创建一个**WebDataBinderFactory进行数据绑定**，同时会去**解析@InitBinder**注解，然后会去创建一个**ModelFactory**同时**解析@ModelAttribute**注解并将控制器临时**产生的数据存入ModelAndViewContainer**中，然后就会去**调用ServletInvocableHandlerMethod**

​	调用ServletInvocableHandlerMethod时先**通过HandlerMethodArgumentResolverComposite获取方法的参数**即解析请求里的参数（会自动转换），这时会**解析@RequestBody注解**，**有的解析器**涉及数据绑定**生成模型数据存到ModelAndViewContainer**，得到参数后就通过**反射调用控制器方法**method.invoke(bean,args)得到**返回值**returnValue，将returnValue**交给HaddlerMethodReturnValueHandlerComposite处理**，有的处理器涉及**@ResponseBody注解**就会在这里解析，不同处理器处理完以后会将**Model数据**或**视图名**或**是否渲染**等**信息存入ModelAndViewContainer**，最后**ServletInvocableHandlerMethod**会去**找ModelAndViewContainer获得ModelAndView**

​	下面是通过代码演示一下ServletInvocableHandlerMethod的使用

```java
// 创建一个ServletInvocableHandlerMethod对象，参数为控制器对象和方法对象（bean和method）
ServletInvocableHandlerMethod handlerMethod =
        new ServletInvocableHandlerMethod(new Controller(),Controller.class.getMethod("foo"));
// 接下来把剩余的需要部分补充完整
// 创建WebDataBinderFactory，这里选用的是其中一个实现类，不需要扩展功能所以参数都为null
ServletRequestDataBinderFactory factory = new ServletRequestDataBinderFactory(null,null);
// 将WebDataBinderFactory设置进去
handlerMethod.setDataBinderFactory(factory);
// 设置参数名解析器，这里之前就做过
handlerMethod.setParameterNameDiscoverer(new DefaultParameterNameDiscoverer());
// 设置参数解析器，这里参考参数解析器那一章，这里传入的是空的HandlerMethodArgumentResolverComposite
handlerMethod.setHandlerMethodArgumentResolvers(new HandlerMethodArgumentResolverComposite());
// 返回值处理器暂时先不用
// 创建一个ModelAndViewContainer
ModelAndViewContainer container = new ModelAndViewContainer();
// 然后可以开始调用（参数一时请求对象，参数二是一个ModelAndViewContainer）
handlerMethod.invokeAndHandle(reequest,container);
```



## 返回值处理器

​	与参数解析器一样，通过一个类似的类**HandlerMethodReturnValueHandlerComposite**来添加**一堆的返回值处理器**统一处理，下面是**演示ModelAndViewMethodReturnValueHandler**返回值处理器的代码

```java
// 创建HandlerMethodReturnValueHandlerComposite
HandlerMethodReturnValueHandlerComposite composite = new HandlerMethodReturnValueHandlerComposite();
// 往composite里面添加ModelAndViewMethodReturnValueHandler处理器
composite.addHandler(new ModelAndViewMethodReturnValueHandler());
// 判断是否支持这个方法的返回值类型，这里需要用HandlerMethod的getReturnType方法
if (composite.supportsReturnType(handlerMethod.getReturnType())){
    // 如果支持则处理返回值，参数一是返回值，参数二是返回值类型，参数三是容器，参数四是请求
    composite.handleReturnValue(returnValue,handlerMethod.getReturnType(),container,request);
}
```

​	对于**ModelAndViewMethodReturnValueHandler返回值处理器**就是将**返回的ModelAndView类型**的数据**存入ModelAndViewContainer**里面

​	**ViewNameMethodReturnValueHandler返回值处理器**就是根据**返回的字符串**加上设置的前缀后缀去**查找视图并返回**这个视图

​	**ServletModelAttributeMethodProcessor返回值处理器**就是将**方法的返回值存入ModelAndViewContainer**，这个**处理结果**是**没有视图**的，会以**请求路径去找视图**

​	处理**HttpEntity**返回类型（java对象转成json）、**HttpHeaders**返回类型和带有**@ResponseBody**的方法的**返回值处理器**都会调用**ModelAndViewContainer**中的**setRequestHandled(true)**设置**请求已经处理完毕**，**不会继续走视图渲染和视图解析**了



## MessageConverter

​	使用的例子，对于**@ReqeustBody**的参数解析器和**@ResponseBody**、**HttpEntity**返回类型的返回值处理器需要让**java对象和json字符串互相转换**，这种情况下就是**用到了这个MessageConverter**，这些**器构造时**就需要**传入MessageConverter的列表类型参数**，下面是一个MessageConverter实现类**MappingJackson2HttpMessageConverter**的演示代码

```java
// 创建一个转换器实例
MappingJackson2HttpMessageConverter converter = new MappingJackson2HttpMessageConverter();
// 判断是否支持json格式的转换
if (converter.canWrite(Bean.class, MediaType.APPLICATION_JSON)){
    // 如果支持就转换，第三个参数是可以存放http的输出信息，即要把转换好的字符串存到哪里
    converter.write(new Bean("张三",18),MediaType.APPLICATION_JSON,message);
}
```

​	**json字符串转换为java对象**只需要将**canWrite和write**改为**canRead和read**就行了，然后message是**http的输入对象**不是输出对象，

**MappingJackson2XmlHttpMessageConverter**类实现的是**java对象和xml对象**之间的转换

​	MessageConverter有很多个的时候，**默认**情况下**先执行先加的那个**，除非是在**request设置了返回值的类型**（Accept头）就只能使用可以转换为指定类型的MessageConverter或者**response设置了返回类型**（content-type，可以在**@RequestMapping**的**product**属性设置），**这个优先级最高**



## Tomcat的异常处理

​	前面说的异常处理只能是**处理控制器抛出的异常**，如果**更上层的异常**（如**过滤器的异常**）是**处理不到**的，这个时候**需要Tomcat来处理**异常，但是它只会在网页上显示**一个详细的栈路径错误**，如果想要**自定义错误地址**如下所示

```java
// 修改Tomcat服务器默认的错误地址
@Bean
public ErrorPageRegistrar errorPageRegistrar(){
    return new ErrorPageRegistrar() {
        @Override
        // 这里的ErrorPageRegistry参数其实就是TomcatServletWebServerFactory
        public void registerErrorPages(ErrorPageRegistry registry) {
            // 增加新的错误路径，参数为路径，这里不是请求转发，即地址栏的路径不会变
            registry.addErrorPages(new ErrorPage("/error"));
        }
    };
}

// 只有上面的不够，还得再加一个bean，因为上面的不会主动执行
// 当TomcatServletWebServerFactory初始化之前就会执行这个ErrorPageRegistrarBeanPostProcessor
// ErrorPageRegistrarBeanPostProcessor会在容器里找到所有的ErrorPageRegistrar并调用
@Bean
public ErrorPageRegistrarBeanPostProcessor errorPageRegistrarBeanPostProcessor(){
    return new ErrorPageRegistrarBeanPostProcessor();
}
```

​	然后有一个**BasicErrorController**类需要了解

```java
@Bean
// 这个是实现好的一个Tomcat错误处理的控制器，它的映射路径就是上面的默认地址，出错时会自动转到这个控制器
// 第一个参数是要显示的错误属性，这里使用了默认的，第二个参数是一个配置，可以在properties文件里进行配置，这里不配
public BasicErrorController basicErrorController(){
    return new BasicErrorController(new DefaultErrorAttributes(),new ErrorProperties());
}
```

​	如果想要配置错误的网页文件，就得在加一个名为error的view进入容器，代码如下

```java
@Bean
// 一个方法名为error返回值类型为View的方法，bean的名字会是error
public View error(){
    return new View() {
        @Override
        // 需要实现一个渲染的方法，model是模型数据，request是请求，response是相应
        public void render(Map<String, ?> model, HttpServletRequest request, HttpServletResponse response) throws Exception {
            // 设置响应类型
            response.setContentType("text/html;charset=utf-8");
            // 设置响应内容
            response.getWriter().print("<h1>wdnmd</h1>");
        }
    };
}

// 光有上面的还不够，还得配置一个视图解析器，让DispatcherServlet按哪种方法去查找视图
@Bean
public ViewResolver viewResolver(){
    // 根据bean的名字去查找视图，因为BasicErrorController里设置视图就是一个字符串“error"
    return new BeanNameViewResolver();
}
```



## HandlerMapping和HandlerAdapter

​	**前面的所有**东西基本都是**基于RequestMappingHandlerAdapter和RequestMappingHandlerAdapter**这两个实现类的，然而**还有很多的实现类**

​	例如**BeanNameUrlHandlerMapping**实现类，是根据**路径**在**容器找相同名字的bean**来**处理请求**（bean的**名字必须是以斜杠开头**）

​	**SimpleControllerHandlerAdapter**实现类就是让所有控制器要**实现一个Controller接口**，当需要这个控制器处理的时候会**调用接口里的handleRequest方法**

​	下面用代码**实现一下上面说的两个类**



### 自定义HandlerMapping

```java
// 自定义HandlerMapping需要实现HandlerMapping接口
class MyHandlerMapping implements HandlerMapping{
    @Override
    // 这个方法需要用request来找到Controller
    public HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
        // 获取request的访问uri
        String uri = request.getRequestURI();
        // 直接在collect里找对应名字的bean，键名就是bean名
        Controller controller = collect.get(uri);
        // 如果没有找到直接返回null
        if (controller == null){
            return null;
        }
        // 包装成执行链返回，因为可能涉及一些拦截器控制器
        return new HandlerExecutionChain(controller);
    }

    // 需要一个容器对象，去找bean
    @Autowired
    private ApplicationContext context;

    // 存放符合控制器条件的bean
    private Map<String, Controller> collect;

    @PostConstruct
    // 初始化方法，在容器中先找到所有名字以斜杠/开头并且实现了Controller的bean
    public void init(){
        // 去找实现了Controller接口的bean
        Map<String, Controller> map = context.getBeansOfType(Controller.class);
        // 过滤掉名字不是以/开头的bean
        collect = map.entrySet().stream().filter(e -> e.getKey().startsWith("/"))
                .collect(Collectors.toMap(e -> e.getKey(), e -> e.getValue()));

    }
}
```



### 自定义HandlerAdapter

```java
// 自定义HandlerAdapter需要实现HandlerAdapter接口
class MyHandlerAdapter implements HandlerAdapter{
    @Override
    // 判断传过来的控制器是不是该适配器支持的
    public boolean supports(Object handler) {
        // 判断是否实现了Controller接口，是就是支持
        return handler instanceof Controller;
    }

    @Override
    // 执行控制器的方法
    public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        // 既然实现了Controller接口就转换为Controller接口
        if(handler instanceof Controller controller){
            // 执行Controller的方法，并把请求和相应传入
            controller.handleRequest(request,response);
        }
        // 返回null就不走视图渲染流程
        return null;
    }

    @Override
    // 这个方法已经过时了没用了，返回一个-1就可以
    public long getLastModified(HttpServletRequest request, Object handler) {
        return -1;
    }
}
```



### RouterFunctionMapping和HandlerFuncitonAdapter

​	这是一组**函数式控制器**，比较适合**业务逻辑简单**的控制器，可以写的非常的**简洁**，主要的逻辑是RouterFunctionMapping**通过RequestPredicate映射路径**，然后**控制器要实现HandlerFunction接口**，HandlerFuncitonAdapter会**调用**HandlerFunction**接口的方法**，下面是演示代码

```java
@Bean
// 将RouterFunctionMapping加入容器
public RouterFunctionMapping routerFunctionMapping(){
    return new RouterFunctionMapping();
}

@Bean
// 将HandlerFunctionAdapter加入容器
public HandlerFunctionAdapter handlerFunctionAdapter(){
    return new HandlerFunctionAdapter();
}

@Bean
// 返回一个RouterFunction路径的映射依据，这里面就已经包含handler，范型经常就是ServerResponse
public RouterFunction<ServerResponse> h(){
    // 使用RouterFunctions的静态方法来构造，第一个参数为条件，这里是路径为/h且GET请求
    // 第二个参数就是控制器类，这里就是直接新建了一个接口的匿名实现类
    return RouterFunctions.route(RequestPredicates.GET("/h"), new HandlerFunction<ServerResponse>() {
        @Override
        public ServerResponse handle(ServerRequest request) throws Exception {
            // ok为静态方法，表示成功
            return ServerResponse.ok().body("success!");
        }
    });
}
```

​	**RouterFunctionMapping**在**初始化**的时候就会将**容器里所有的RouterFunction记录**下来，请求来了就会**跟每个的条件进行匹配**找到处理器对象，然后由**HandlerFuncitonAdapter**调用**处理器的方法**并**将返回值返回给浏览器**



### SimpleUrlHandlerMapping和HttpReuqestHandlerAdapter

​	**静态资源处理**：SimpleUrlHandlerMapping做**映射**，ResourceHttpRequestHandler作为处理器**处理静态资源**，HttpRequestHandlerAdapter**调用处理器**，下面是演示代码

```java
@Bean
// 将SimpleUrlHandlerMapping放入容器
public SimpleUrlHandlerMapping simpleUrlHandlerMapping(ApplicationContext context){
    SimpleUrlHandlerMapping handlerMapping = new SimpleUrlHandlerMapping();
    // 这个类没有afterPropertiesSet方法，所以找所有的ResourceHttpRequestHandler得自己实现
    // 在容器中找出所有ResourceHttpRequestHandler类型的bean
    Map<String, ResourceHttpRequestHandler> map = context.getBeansOfType(ResourceHttpRequestHandler.class);
    // 找到后用setUrlMap存入路径映射
    handlerMapping.setUrlMap(map);
    return handlerMapping;
}

@Bean
// 将HttpRequestHandlerAdapter放入容器
public HttpRequestHandlerAdapter httpRequestHandlerAdapter(){
    return new HttpRequestHandlerAdapter();
}

// 将bean的名字定为这样，这样所有访问路径为/xxx的访问都会和它匹配然后在/static找对应名字的文件
// 这个路径是虚拟的，只是为了映射匹配使用
@Bean("/**")
// 将ResourceHttpRequestHandler放入容器
public ResourceHttpRequestHandler handler(){
    ResourceHttpRequestHandler handler = new ResourceHttpRequestHandler();
    // 设置资源的路径，使用ClassPathResource可以指定类加载路径下的resources文件夹为根目录
    handler.setLocations(List.of(new ClassPathResource("/static")));
    // 在ResourceHttpRequestHandler的afterPropertiesSet方法中只是设置了一个最基本的资源解析器
    // 可以换成spring里其它功能更强大的解析器来优化，这里加了三个资源解析器（责任链模式，顺序很重要，高级的套着低级的）
    handler.setResourceResolvers(List.of(
            // 第一个就是在读资源的时候加入缓存功能提高性能，参数是缓存实现
            new CachingResourceResolver(new ConcurrentMapCache("cacheName")),
            // 第二个就是可以读压缩资源减少网络传输的量提高性能
            new EncodedResourceResolver(),
            // 第三个就是那个最基本的资源解析器，只是从磁盘上读文件
            new PathResourceResolver()
    ));
    return handler;
}
```



### WelcomePageHandlerMapping

​	当**不输入路径**的时候会自动进入**欢迎页**，生成**内置**的**ParameterizableViewController**控制器，**不执行逻辑仅根据视图名找视图**，而视图名**固定为forward:index.html**（映射到/\*\*，和上面对上了），而这个控制器**实现了Controller接口**，所以可以**用SimpleControllerHandlerAdapter来调用**它，**调用时**要**处理/index.html**就会走上面**处理静态资源**的流程，下面是演示代码

```java
@Bean
// 将WelcomePageHandlerMapping放入容器
public WelcomePageHandlerMapping welcomePageHandlerMapping(ApplicationContext context){
    // 获取欢迎页的资源对象
    Resource resource = context.getResource("classpath:static/index.html");
    // 第一个参数与动态有关这里设为null，第二个为容器，第三个参数为资源对象，最后一个值都是/**
    return new WelcomePageHandlerMapping(null,context,resource,"/**");
    // 配好后他会生成一个实现类Controller接口的处理器，所以需要SimpleControllerHandlerAdapter来调用它
}

@Bean
// 将SimpleControllerHandlerAdapter放入容器
public SimpleControllerHandlerAdapter simpleControllerHandlerAdapter(){
    return new SimpleControllerHandlerAdapter();
}
```



### 总结

​	还有**很多映射器**，但是**SpingBoot**中也就用到了**上面**说的**5种映射器4种适配器**，**HandlerMapping**负责建立**请求与控制器之间的映射**关系，**HandlerAdapter**负责实现对各种各样的handler适配调用

​	这5种映射器还有顺序，boot中的顺序（越高越优先）：

- RequestMappingHandlerMapping（@RequestMapping）
- WelcomePageHandlerMapping（/）
- BeanNameUrlHandlerMapping（bean的名字匹配且/开头）
- RouterFunctionMapping（函数式RequestPredicate，HandlerFunction）
- SimpleUrlHandlerMappinjg（静态资源访问）



## MVC处理流程

当浏览器发送了一个请求到达服务器后，其处理流程是：

  1. 服务器提供了DispatcherServlet，它使用的是标准Servlet技术

    - 路径：**默认映射路径为**`/`，即会匹配所有请求URL，可作为请求的统一入口，也被称之为**前控制器**（**jsp不会匹配**到DispatcherServlet，它优先级更高）
    - 创建：在**Boot**中，由**DispatcherAutoConfiguration**这个**自动配置**类提供**DispatcherServlet**的bean
    - 初始化：DispatcherServlet**初始化时**会优先到容器里**寻找各种组件**，作为它的**成员变量**
    
      - **HandlerMapping**，初始化时**记录映射关系**
      - **HandlerAdapter**，初始化时准备参数**解析器**、**返回值处理**器、**消息转换**器
      - **HandlerExceptionResolver**，初始化时准备参数**解析器**、**返回值处理**器、**消息转换**器
      - **ViewResolver**
    
    2. DispatcherServlet会**利用RequestMappingHandlerMapping查找控制器方法**
     - 例如根据/hello路径找到@RequestMapping("/hello")对应的控制器方法
     - **控制器方法**会被**封装成HandlerMethod对象**，并结合匹配到的拦截器一起返回给DispatcherServlet
     - HandlerMethod**和拦截器合在一起**称为**HandlerExecutionChain（调用链）**对象
    
    3. DispatcherServlet接下来会：
     1. 调用拦截器的**preHandle**方法
     2. **RequestMappingHandlerAdapter**调用**handle**方法，准备**数据绑定工厂**、**模型工厂**、将**HandlerMethod完善为ServletInvocableHandlerMethod**
        - @ControllerAdvice增强1：补充模型数据（**@ModelAtrribute**）
        - @ControllerAdvice增强2：补充自定义类型转换器（**@InitBinder**）
        - 使用**HandlerMethodArgumentResolver**准备参数
        - @ControllerAdvice增强3：**ResquestBody增强**
        - **调用ServletInvocableHandlerMethod**
        - 使用**HandlerMethodReturnValueHandler**处理返回值
          - 如果**返回的ModelAndView为null**，**不走第4步视图解析及渲染**流程
            - 例如，标注了@ResponseBody的控制器方法，调用HttpMessageConverter来将结果转换为JSON，这时返回的ModelAndView就为null
          - 如果返回的ModelAndView**不为null**，会在第4步走**视图解析及渲染**流程
          - @ControllerAdvice增强4：**ResponseBody增强**
     3. 调用拦截器的**postHandle**方法
     4. **处理异常**或**视图渲染**
        - 如果1~3步骤**出现异常**，走ExceptionHandlerExceptionResolver**处理异常流程**
          - @ControllerAdvice增强5：**@ExceptionHandler**异常处理
        - **正常**，**走视图解析及渲染**流程
     5. 调用拦截器的**afterCompletion**方法

​	但是对于**拦截器**的实现没有讲到，有时间自己看看源代码



# SpringBoot



​	生成springboot的**pom文件**可以通过使用命令行然后`curl https://start.spring.io/pom.xml`来获取，需要**添加其它的依赖**可以在后面加上`-d dependencies=mysql,mybatis,web`这些参数就可以了，使用`-o pom.xml`可以保存到当前目录

​	springboot也可以**用war的打包方式运行**，用war的打包方式运行即用的是**外置的Tomcat服务器**，**jar是内置的**，而且一般用war打包主要是因为要用jsp技术因为**jsp只能用war**打包运行，需要在idea里面设置一下**运行配置**，然后用war打包方式的会多一个**ServletInitializer**类，这个类实现**SpringBootServletInitializer**接口，里面有一个**configure**方法就是在**外置Tomcat服务器启动**的时候**调用**的，**返回值类型**就是**SpringApplicationBuilder**，帮助**创建SpringApplication**

​	用**war打包**的也**可以用内置的Tomcat服务器运行**（就是直接去**SpringApplication**的类**运行main**函数），不过内置的Tomcat**缺少jsp的解析器**，需要**导入**一个**tomcat-embed-jasper**包



## Boot启动流程

### SpringApplication的构造

1. ​	获取**Bean Definition**源（就是最外面**调用run函数传的参**，但也可以是其他的**配置类**或者**配置文件**）
2. ​	**推断应用类型**（通过是否**存在某个类**，一共有**reactive**、**servlet**、**none**三种类型）
3. ​	**自动配置**ApplicationContext**初始化器**，是对功能进行扩展的（源码是从**spring.factories**里读取一堆bean）
4. ​	**自动配置监听器**（就是经典的**spring.factories**，自动配置里面设置好的监听器）
5. ​	**推断主类**（就是**运行main方法**的那个类，引导类）

​	其中可以用**setSource**方法给SpringApplication设置源，用**addInitializers**方法添加初始化器，用**addListener**方法添加监听器

### SpringApplication的run方法

1. [得到事件发布器SpringApplicationRunListener（发布application starting事件）](#事件发布相关)
2. [封装启动args](#启动参数相关)
3. [准备Environment添加命令行参数](#环境信息相关)
4. [ConfigurationPropertySources处理（发布application environment已准备事件）](#环境信息相关)
5. [通过EnvironmentPostProcessorApplicationListener进行env后处理](#环境信息相关)
6. [绑定spring.main到SpringApplication对象](#其它)
7. [打印banner](#其它)
8. [创建容器](#容器相关)
9. [准备容器](#容器相关)
10. [加载bean定义](#容器相关)
11. [refresh容器](#容器相关)
12. [运行runner](#启动参数相关)



#### 事件发布相关

​	对于第1步的**得到事件发布器SpringApplicationRunListener**，**run的流程**完全**靠监听器监听事件**来**控制进行**，所以事件的发布很重要，下面是测试事件发布器的代码

```java
// 通过传入接口名在spring.factories文件获取所有的事件发送器实现类名
// 里面的事件发送器实现类只有一个，就是EventPublishingRunListener
List<String> names = SpringFactoriesLoader.loadFactoryNames(SpringApplicationRunListener.class
        , Springboot0101QuickstartApplication.class.getClassLoader());
// 循环构造所有的事件发送器
for (String name : names) {
    // 加载这个类
    Class<?> clazz = Class.forName(name);
    // 由于是只有有参构造方法，不能直接用newInstance，得先获得构造方法对象（参数是构造方法的参数类型）
    Constructor<?> constructor = clazz.getConstructor(SpringApplication.class, String[].class);
    // 传入SpringApplication对象与main方法的参数，获得事件发布器实例
    SpringApplicationRunListener publisher = (SpringApplicationRunListener)constructor.newInstance(app, args);
    
    // 发布事件（下面的代码只是演示发布事件的顺序，与上面的整体逻辑无关，参数也全不是空，只是没加）
    publisher.starting(); // spring boot 开始启动
    // 设置环境信息源...
    publisher.environmentPrepared(); // 环境信息源准备完毕
    // 调用环境信息后处理器、初始化器...
    publisher.contextPrepared(); // 在spring容器创建，调用初始化器之后
    // 加载所有bean definition
    publisher.contextLoaded(); // 所有bean definition加载完毕
    // 调用refresh方法
    publisher.started(); // spring容器初始化完成，即refresh方法调用完毕
    publisher.running(); // spring boot 启动完毕

    
    publisher.failed(); // spring boot 启动出错
    // 这些事件都能通过监听器监听到
}
```



#### 容器相关

​	对于第8步**创建容器**是调用了一个**createApplicationContext**方法，通过前面**构造**SpringApplication时**推断出的应用类型**构造**不同的ApplicationContext**，如果是servlet类型那就是**AnnotationConfigServletWebServerApplicationContext**类型的容器，如果是reactive类型那就是**AnnotationConfigReactiveWebServerApplicationContext**类型的容器，如果是none类型那就是**AnnotationConfigApplicationContext**类型的容器

​	对于第9步**准备容器**就是通过调用容器的**getInitializers**方法获取所有的初始化器然后遍历**调用其initalize**方法**初始化容器**

​	对于第10步**加载bean定义**的模拟代码如下

```java
// 加载配置类的，需要的参数就是容器里的BeanFactory
AnnotatedBeanDefinitionReader reader1 =
        new AnnotatedBeanDefinitionReader(context.getDefaultListableBeanFactory());
// 设置读取哪个配置类
reader1.register(Config.class);

// 加载配置文件的，需要的参数也是容器里的BeanFactory
XmlBeanDefinitionReader reader2 = new XmlBeanDefinitionReader(context.getDefaultListableBeanFactory());
// 设置读取哪个配置文件
reader2.loadBeanDefinitions(new ClassPathResource("application.xml"));

// 直接组件扫描包的，需要的参数也是容器里的BeanFactory
ClassPathBeanDefinitionScanner scanner = new ClassPathBeanDefinitionScanner(context.getDefaultListableBeanFactory());
// 设置扫描哪个包
scanner.scan("com.example.spring_high");
```

​	对于第11步**refresh容器**就是`context.refresh();`



#### 启动参数相关

​	对于第12步**运行runner**，就是调用runner的run方法，runner一共需要实现两个接口，一个是**commandLinerRunner**接口，一个是**ApplicationRunner**接口，都是**函数式接口**，commandLinerRunner顾名思义就是命令行，其run方法的参数就是**字符串无处理过的main方法参数**，而ApplicationRunner的run方法参数是经过**封装成ApplicationArguments类型的参数**，运行时会**获取所有的runner的bean**（通过类型获取）**挨个进行**，**两种接口**的**都会进行**

​	对于第2步**封装启动args**时就**需要将原始启动args封装**成一个**ApplicationArguments**类型的对象，供**第12步运行runner时使用**



#### 环境信息相关

​	对于第3步**准备Environment添加命令行参数**，**spring**里使用的是一个**StandardEnvironment**实现，**boot**里用的则是继承其的**ApplicationEnvironment**实现，**web环境**还有**很多扩展**，环境信息有很多来源，可以来源于**系统环境变量**、**properties**、**yaml**等，但是**默认**情况下只会有**两个来源**：**系统变量**（**systemProperties**）和**系统属性**（**systemEnvironment**），下面是演示代码

```java
// 获取环境对象
ApplicationEnvironment env = new ApplicationEnvironment();
// 通过getPropertySource方法获取环境信息的所有来源，然后遍历输出
for(PropertySource<?> ps : env.getPropertySource()){
    // 这里输出结果只会有systemProperties和systemEnvironment
    System.out.println(ps);
}
// 可以通过getProperty获取对应的属性值
env.getProperty("JAVA_HOME");

// 可以添加外部的环境信息配置，比如添加properties文件
// 添加到最后是因为properties文件一般优先级最低
env.getPropertySource().addLast(new ResourcePropertySource
        (new ClassPathResource("application.properties")));

// 可以添加命令行的环境信息来源
// 命令行的参数一般都认为是最优先的所以添加到最前面（参数就是main方法的参数）
env.getPropertySource().addFirst(new SimpleCommandLinePropertySource(args));
```

​	对于第4步**ConfigurationPropertySources处理**就是**往环境源**中添加了一个**ConfigurationProperties**源，这个源可以让**不管在配置文件**里用的是**驼峰**命名还是**下划线**命名还是**减号**命名的变量全部**统一处理成减号**命名，这样**实现了**在**配置文件**中变量**命名的随机**性，因为不管是哪种最后都可以用减号命名读取，所以当我们写配置文件的时候就可以随便选一种写，就是加了下面这样一句代码直接将这个源**放在了最最优先**的地方

```java
// 调用了ConfigurationPropertySources类的静态方法attach，把环境对象传入
ConfigurationPropertySources.attach(env);
```

​	对于第5步**通过EnvironmentPostProcessorApplicationListener进行env后处理**就是通过**后处理器**给环境了**补充**一些新的**ProertySources**，其中**application.properties**就是在这个时候加入的，实际上**添加哪些**后处理器是通过**读取spring.factories**配置文件获取的而这个**读取并调用的过程**是EnviromentPostProcessorApplicationListener**监听器完成**的（监听的是**environmentPrepared**事件），当然这个**监听器本身也是从spring.factories**读取而来，下面是一些后处理器的演示代码

```java
// 构造EnvironmentPostProcessor接口的实现类就是后处理器类
// 下面这个类就是添加application.properties源的实现类
// 构造参数不是需要关心的
ConfigDataEnvironmentPostProcessor postProcessor1 =
        new ConfigDataEnvironmentPostProcessor(new DeferredLogs(), new DefaultBootstrapContext());
// 然后就执行后处理，需要环境对象和SpringApplication对象
postProcessor1.postProcessEnvironment(env,app);

// 下面这个类是可以添加一个random源
RandomValuePropertySourceEnvironmentPostProcessor postProcessor2 =
        new RandomValuePropertySourceEnvironmentPostProcessor(new DeferredLog());
// 执行后处理
postProcessor1.postProcessEnvironment(env,app);
// 然后就可以像这样获得一个随机的值
env.getProperty("random.int");
```



#### 其它

​	对于第6步**绑定spring.main到SpringApplication对象**，就是**将配置文件**中（application.**properties**，application.**yml**）以**spring.main开头**的配置项的值**绑定到**那个**SpringApplication**对象上，即让这个**配置生效**，下面是模拟代码

```java
// 使用Binder类的静态方法get获得环境信息并结合bind将spring.main开头的配置信息绑定到app中
Binder.get(env).bind("spring.main", Bindable.ofInstance(app));
```

​	对于第7步**打印banner**就是打印logo，没啥好说的下面是演示代码

```java
// 构造banner的打印器
SpringApplicationBannerPrinter printer =
        new SpringApplicationBannerPrinter(new DefaultResourceLoader(), new SpringBootBanner());
// 然后就可以输出了，第一个参数是环境对象，第二个参数是main方法所在类，第三个对象是输出流
printer.print(env,Springboot0101QuickstartApplication.class,System.out);
// 可以在配置文件里设置spring.banner.location指定banner文件的位置，可改图片banner或者文字banner
```



## 内嵌Tomcat容器

​	Tomcat服务器中**每个应用**都是一个**context**，对于context里的资源分为两类：**静态**资源（html、css、js）和**动态**资源（**jsp**、**第三方jar包**、**自己编写的**class文件），而class文件**不能直接被Tomcat识别**，Tomcat只能识别**servlet**、**filter**、**listener三大组件**，而且这三大组件**还得经过web.xml的配置**才能被Tomcat识别，自己写的class文件都是**通过三大组件间接地调用**，Tomcat**3.0版本以后**就**可以不需要web.xml**文件而是通过**编程的方法**来添加，**内嵌容器就是**通过编程的方法添加，下面就用编程的方法来**创建内嵌的Tomcat**

```java
// 构建一个Tomcat对象
Tomcat tomcat = new Tomcat();
// 设置一个基础目录（工作目录，即服务器运行期间临时文件的存放地）
tomcat.setBaseDir("tomcat");
// 创建项目的文件夹（真实的，docBase文件夹）
// 使用了java的工具类创建临时文件夹，前缀是boot.后面会生成一个随机的值
File docBase = Files.createTempDirectory("boot.").toFile();
// 并且设置这个文件夹在程序退出后就删除
docBase.deleteOnExit();
// 创建Tomcat的项目，就是context，第一个参数是虚拟访问路径，第二个参数就是对应的项目实际路径
Context context = tomcat.addContext("/", docBase.getAbsolutePath());

// 通过编程添加Servlet
// 添加一个Servlet的初始化器，这个是当启动Tomcat以后才回调执行
context.addServletContainerInitializer(new ServletContainerInitializer() {
    @Override
    public void onStartup(Set<Class<?>> set, ServletContext servletContext) throws ServletException {
        // 通过这个方法添加Servlet，第一个参数是Servlet名随便起，第二个是Servlet类或者对象
        // 最后需要添加以下映射路径，这个Servlet类继承HttpServlet接口就好了
        servletContext.addServlet("aaa",MyServlet.class).addMapping("root");
    }
});

// 启动Tomcat
tomcat.start();
// 创建连接器，设置监听窗口（监听什么协议什么端口）
// 指定了http2协议
Connector connector = new Connector(new Http11Nio2Protocol());
// 设置端口
connector.setPort(8080);
// 使连接器和Tomcat关联起来
tomcat.setConnector(connector);
```

#### Spring和内置Tomcat的关联

​	**AnnotationConfigServletWebServerApplicationContext**这个容器就已经**实现了Spring和内置Tomcat服务器之间**的联系，而**AnnotationConfigWebApplicationContext**这个容器就**没有实现**，下面模拟一下需要**怎么建立联系**

```java
// 只需要在Servlet的初始化器中初始化就行
context.addServletContainerInitializer(new ServletContainerInitializer() {
    @Override
    public void onStartup(Set<Class<?>> set, ServletContext servletContext) throws ServletException {
        // 获取Spring中的前控制器
        DispatcherServlet dispatcherServlet = springContext.getBean(DispatcherServlet.class);
        // 注册
        servletContext.addServlet("dispatcherServlet",dispatcherServlet).addMapping("/");

        // 但是上面这种方法写死了，一般应该采用下面这种写法（DispatcherServletRegistrationBean就是在这里用的）
        for (ServletRegistrationBean registrationBean : springContext.getBeansOfType(
            ServletRegistrationBean.class).values()) {
            // 获取所有ServletRegistrationBean的类调用其onStartup方法即可注册
            registrationBean.onStartup(servletContext);
        }
    }
});
```

​	**Spring**是在**refresh**方法中的**onRefresh**方法中**初始化好Tomcat**服务器，然后再**finishRefresh**方法里**启动Tomcat服务器**



## 自动配置

​	自动配置有关的可以[查询SpringBoot源码.md]( ./SpringBoot源码#SpringBoot原理篇)，这里说几点：

 1. 对于**加载spring.factories**里的自动配置类**为什么要**使用**[DeferredImportSelector]( ./SpringBoot源码#自动配置原理)延迟加载**？

    - 首先**spring默认允许同名的bean**进行覆盖，不过是**先注册的**bean**覆盖后注册的**
    - 然后就是**自动配置**很多都与**@Import**有关，而对于一个类@Import的**执行时期要比类**里的**@Bean要早**
    - 所以为了**使本地配置的bean优先**，让**第三方的bean**使用**延迟加载**DeferredImportSelector
    - **springboot默认不允许覆盖**，有同名的bean**会报错**，可以通过**BeanFactory**的**set方法设置**
    - 同时**第三方的包在定义bean**的时候可以加一个**@ConditionalOnMissingBean**，只有**没有同名的bean**时**才注册**

 2. 对于实现了**[ImportBeanDefinitionRegister](./SpringBoot源码#bean装配方式)**接口的类，在注册为bean后会通过**编程的方法去注册bean**，但是

    - 得告诉它要**扫描哪些包**里的类，这个可以通过**AutoConfigurationPackages.register方法设置**
    - **设置**的包名是**通用**的，即只要**实现了ImportBeanDefinitionRegister接口**的类在**注册为bean**后**都会去扫**
    - 在**类上加@AutoConfigurationPackage**注解和上面的**效果是一样的**
    - 注意是**编程的方法**，即可以**通过代码逻辑来判断是否注册**，与@ComponentScan的简单条件不一样

 3. 对于[添加自己的自动配置类](./SpringBoot源码#自己变更自动配置)时spring.factories里的键名其实是固定的，因为

    - Spring里有个**总的读取**自动配置文件的选择器**AutoConfigurationImportSelector**（万恶之源）
    - 这个ImportSelector在selectImports方法里通过**loadFactoryName**方法读取的是**固定的一个类名**
    - 这个类名就是**EnableAutoConfiguration**，想要spring**帮你自动配置就需要也用这个键名**

    - 所以键名必须是**org.springframework.boot.autoconfigure.EnableAutoConfiguration**

 4. 对于[bean的加载控制](./SpringBoot源码#bean的加载控制)，下面给出一个自己**实现Condition接口**的代码

```java
static class MyConditional implements Condition{
    @Override
    // 判断是否有Druid的依赖
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        return ClassUtils.isPresent("com.alibaba.druid.pool.DruidDataSource",null);
    }
}
// 下面这样使用就行了
@Conditional(MyConditional.class)

// 上面这样写的判断写死了，可以自定义一个注解，spring就是这样干的
// 自定义一个注解判断是否存在类或者是否不存在
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.METHOD,ElementType.TYPE})
@interface ConditionalOnClass{
boolean exist();  // true判断存在，false判断不存在
    String className();  // 要判断的类名
}

// 修改后的自定义Conditioinal
static class MyConditional2 implements Condition{
    @Override
    // 判断是否有Druid的依赖
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        // 获得自定义注解上所有的属性，存为map
        Map<String, Object> attributes =
                metadata.getAnnotationAttributes(ConditionalOnClass.class.getName());
        // 获得判断的类名
        String className = attributes.get("className").toString();
        // 获得是判断存在还是判断不存在
        boolean exists = (boolean)attributes.get("exists");
        // 判断类是否存在
        boolean present = ClassUtils.isPresent(className, null);
        return exists ? present : !present;
    }
}

// 最后用的时候
@ConditionalOnClass(exist = true,className = "com.alibaba.druid.pool.DruidDataSource")
```



# 查漏补缺



## FactoryBean的缺憾

​	可以写一个实现了**FactoryBean接口**的类来作**bean的工厂**，接口中需要实现**产品的类**、**是否单例**、**提供产品对象**的方法，在Context中通过使用**工厂bean的名字**来获取时会**获得产品bean对象**，如果想要用bean名字**获得工厂对象**需要在**名字前加**一个`&`，这个FactoryBean的**缺憾**就是**创建产品**的过程**spring没有介入**，即产品类里**@Autowired**、**@PostProduct**、**Aware接口回调**的方法**都不会调用**，对于Bean的**后处理器**也只有**初始化后的会运行**，而代理增强就是在初始化后创建的，即**可以使用代理增强**，**单例**的产品**不会存储**于BeanFactory中的**SingletonObjects**成员中，**而是**另一个**factoryBeanObjectCache**成员中，实际上**@Bean方法**已具备**等价**功能，**但**目前**此接口仍被大量使用**



## @Indexed注解

​	**组件扫描**的**消耗太大**，影响spring的**启动速度**，所以**spring5.0**做了一个**优化**解决这个问题，就是通过给**每个需要导入的类**加上**@Indexed注解**，然后在**编译的时候**找到**有这个注解的所有类导出**到**META-INF**目录下的**spring.components**文件中，等到**需要扫描**的时候**就不需要jar包扫描**而是**直接挨个读取**这个文件里所**记录下来的类**，使用这个需要**导入spring-context-indexed依赖**，**@Component注解包括了@Indexed**注解



## Spring代理的特点

​	**初始化**和**依赖注入**影响的是**原始对象**，不是代理对象，代理对象和目标对象是**两个对象**，**不共享成员变量**，spring的**单例容器中只**会**储存代理对象**，可以将**代理**对象**转换成Advised**类型，**调用**该类型的`getTargetSource().getTarget()`方法**拿到原始对象**，对于**static**、**private**、**final**类型的方法不能增强，代理只能**增强可以重写**的方法，**反射**也**可以增强**的，**非代理的增强**方法就可以**突破这个限制**



## @Value注入

​	测试代码如下

```java
// 测试bean类
public class Bean{
    @Value("${JAVA_HOME}")
    private String home;

    @Value("18")
    private int age;

    @Value("#{@bean}")  // SpringEl表达式
    private Bean bean;
}


public static void main(String[] args) throws NoSuchFieldException {
    // 为了演示创建的ApplicationContext对象
    AnnotationConfigApplicationContext context =
            new AnnotationConfigApplicationContext(wdnmd.class);
    // 这里因为是演示为了获取@Value的属性值用到这个类
    ContextAnnotationAutowireCandidateResolver resolver =
            new ContextAnnotationAutowireCandidateResolver();
    // 需要设置给它一个BeanFactory
    resolver.setBeanFactory(context.getDefaultListableBeanFactory());

    // 先获取需要的是哪个成员变量或者方法参数上的@Value封装成DependencyDescriptor
    Field field = Bean.class.getDeclaredField("home");
    Field field1 = Bean.class.getDeclaredField("age");
    Field field2 = Bean.class.getDeclaredField("bean");

    //测试@Value("${JAVA_HOME}")
    test1(context, resolver, field);
    // 测试@Value("18")
    test2(context, resolver, field);
    // 测试@Value("#{@bean}")
    test3(context, resolver, field);
}


// 测试@Value("${JAVA_HOME}")
private static void test1(AnnotationConfigApplicationContext context, ContextAnnotationAutowireCandidateResolver resolver, Field field) {
    // 第二个参数是false指即使获取不到也不会报错
    DependencyDescriptor descriptor = new DependencyDescriptor(field, false);
    // 获取@Value的属性值
    String value = resolver.getSuggestedValue(descriptor).toString();
    // 由于需要解析${}，可以传入environment对象里面解析，就解析成功了
    String home = context.getEnvironment().resolvePlaceholders(value);
}


// 测试@Value("18")
private static void test2(AnnotationConfigApplicationContext context, ContextAnnotationAutowireCandidateResolver resolver, Field field) {
    // 第二个参数是false指即使获取不到也不会报错
    DependencyDescriptor descriptor = new DependencyDescriptor(field, false);
    // 获取@Value的属性值
    String value = resolver.getSuggestedValue(descriptor).toString();
    // 需要将String类型转成需要的int类型，这里用的是BeanFactory里的一个TypeConverter
    // TypeConverter属于是高层的转换接口，这个方法会返回一个最简单的实现，只做类型转换的实现
    int age = (int) context.getBeanFactory()
            .getTypeConverter()
            .convertIfNecessary(value, descriptor.getDependencyType());
}


// 测试@Value("#{@bean}")，根据bean的名字找
private static void test3(AnnotationConfigApplicationContext context, ContextAnnotationAutowireCandidateResolver resolver, Field field) {
    // 第二个参数是false指即使获取不到也不会报错
    DependencyDescriptor descriptor = new DependencyDescriptor(field, false);
    // 获取@Value的属性值
    String value = resolver.getSuggestedValue(descriptor).toString();
    // 需要解析#{}，使用BeanFactory里的一个解析器，就可以找到这个bean
    Bean bean = (Bean) context.getBeanFactory().getBeanExpressionResolver()
            .evaluate(value, new BeanExpressionContext(context.getBeanFactory(), null));
}
```



## @Autowired注入

​	演示代码如下

```java
// 用于测试的bean类
public class Bean{
    @Autowired
    private Bean2 bean2;

    @Autowired
    public void setBean2(Bean2 bean){
        this.bean2 = bean;
    }

    @Autowired
    private Optional<Bean2> bean3;

    @Autowired
    private ObjectFactory<Bean2> bean4;

    @Autowired
    private Bean2 bean5;
}

public class Bean2{

}

public static void main(String[] args) throws Exception {
    // 为了演示创建的ApplicationContext对象
    AnnotationConfigApplicationContext context =
            new AnnotationConfigApplicationContext(wdnmd.class);
    // 获得context里的BeanFactory
    DefaultListableBeanFactory beanFactory = context.getDefaultListableBeanFactory();


    // 1. 根据成员变量的类型注入（参数二是Bean的名称，参数三是Bean2的名称，参数四时类型转换器，都不重要）
    DependencyDescriptor descriptor =
            new DependencyDescriptor(Bean.class.getDeclaredField("bean2"), false);
    Bean2 bean2 = (Bean2) beanFactory.
            doResolveDependency(descriptor, "bean", null, null);


    // 2. 根据参数的类型注入
    // 这里就需要获得方法参数的信息而不是成员变量的信息，先获取方法信息
    Method setBean2 = Bean.class.getDeclaredMethod("setBean2", Bean2.class);
    // 设置参数信息（第0个参数）
    DependencyDescriptor descriptor2 =
            new DependencyDescriptor(new MethodParameter(setBean2,0),false);
    // 然后就可以获取到
    Bean2 bean22 = (Bean2) beanFactory
            .doResolveDependency(descriptor2, "bean", null, null);


    // 3. 结果包装为Optional<Bean2>
    DependencyDescriptor descriptor3 =
            new DependencyDescriptor(Bean.class.getDeclaredField("bean3"), false);
    // 这里获取到的是Optional类型，可以用下面这个方法获得内层的类型
    descriptor3.increaseNestingLevel();
    // 然后就可以获取到
    Bean2 bean3 = (Bean2) beanFactory.
            doResolveDependency(descriptor3, "bean", null, null);


    // 4. 结果包装为ObjectFactory<Bean2>
    // 和3差不多，但是不同点是不需要立马获得这个bean2对象，因为是bean工厂，希望是在调用getObject的时候才获取
    DependencyDescriptor descriptor4 =
            new DependencyDescriptor(Bean.class.getDeclaredField("bean4"), false);
    descriptor3.increaseNestingLevel();
    // 所以得这样处理
    ObjectFactory<Bean2> bean4 = new ObjectFactory<Bean2>() {
        @Override
        public Bean2 getObject() throws BeansException {
            return (Bean2) beanFactory.
                    doResolveDependency(descriptor3, "bean", null, null);
        }
    };


    // 5. 对于@Lazy的处理
    // 由于加了@Lazy的成员变量是需要注入一个代理，直到需要使用它的时候才会去访问真正的对象
    DependencyDescriptor descriptor5 =
            new DependencyDescriptor(Bean.class.getDeclaredField("bean5"), false);
    // 这里使用和@Value一样的解析器
    ContextAnnotationAutowireCandidateResolver resolver = new ContextAnnotationAutowireCandidateResolver();
    resolver.setBeanFactory(beanFactory);
    // 这个解析器就会看看有没有@Lazy注解，有的话就会自动创建代理
    Object bean5Proxy = resolver.getLazyResolutionProxyIfNecessary(descriptor5, "bean5");
```

​	其实无论是**@Value**还是**@Autowired**最终**都会调用doResolveDependency**这个方法，然后还有下列几种情况

- 对于**接口数组类型**注入的就需要先用**DependencyDescriptor**的`getDependentType().isArray()`判断**是不是数组**类型，然后就用`getDependentType().getComponentType()`获取数组**装的什么类型**，然后使用`BeanFactoryUtils.beanNamesForTypeIncludingAncestors(beanFactory, componentType)`获取到所有的这个类型的bean名字，然后**先用List存放**这些bean，最后**转回数组类型**注入
- 对于**接口List类型**注入需要用**DependencyDescriptor**的`getResolvableType().resolve()`拿到**范型的类型**，拿到类型后剩下的就**和上面一样**
- 对于**特殊类型如ApplicationContext类型**的注入，首先最终**所有bean**都是存在**DefaultListableBeanFactory**里的**singletonObjects**中，**key是bean的名字**，但是这里是**没有这些特殊类型**的，特殊类型是**存放在resolvableDependencies**这个集合中，**key是对象的类型**，也是**refresh**方法执行中**加进去**的，所以**不能用getBean**获取，内部是直接在这个**成员变量上查询**，使用**class**之间的**isAssignableFrom**方法来判断**是否符合这个类型**（是否是方法参数的**子类**）
- 对于**泛型接口**的注入，首先`BeanFactoryUtils.beanNamesForTypeIncludingAncestors(beanFactory, dependentType)`获取所有**与描述类型相同的bean**名字，这里面的bean的类型**只会是实现同样的接口**，但是**范型不一定相同**，这时可以对**每个bean名字**调用**beanFactory**的**getMergeBeanDefinition**方法获得**BeanDefinition**，然后就可以用经典的**ContextAnnotationAutowireCandidateResolver**类的**isAutowireCandidate**方法判断**范型**类型**是否匹配**，就像这样`resolver.isAutowireCandidate(new BeanDefinitionHolder(beanDefinition,name),dependencyDescriptor)`
- 对于使用了**@Qualifier注解**的注入就是**根据bean的名字**注入，流程和上一条完全一模一样，**ContextAnnotationAutowireCandidateResolver**会解析**@Qualifier**注解，最终通过**isAutowireCandidate**方法判断**是否和dependencyDescriptor**描述的一模**一样**
- 对于加了**@Primary**注解的可以**被注入的bean**，在注入选择有它且有**很多个选择时**就会用它，可以通过**beanFactory**的**getMergeBeanDefinition**方法获得**BeanDefinition**的**isPrimary**方法判断**是否有@Primary注解**
- 对于**最保底**的情况就是根据**成员变量的名字**在找**相同名字的bean**，这就不用多说了



## 事件监听器和发布器

​	实现监听器有两种方法：

- 实现**ApplicationListener**接口**，里面有一个**范型**，可以对**事件进行筛选**，只监听范型里的事件
- 使用**@EventListener注解**在一个返回值为**void**方法**参数为**需要监听的**事件**的方法上，该方法就是监听器

​	其实**@EventListener注解的原理**就是当**扫描到**带有这个注解的方法时直接**当场实现**一个**ApplicationListener接口**的类，里面就通过**反射调用**注解所**标注的方法**，然后通过使用**ApplicationContext**的**addApplicationListener方法加入**这个**新创建的监听器**，当然实现接口的方法时得**做事件类型的判断**，使用经典的**isAssignableFrom**方法判断就行

​	可以将上述这个**扫描所有bean注册监听器**过程写在一个**SmartInitializingSingleton**接口实现的方法里，这个接口的方法会在**所有的单例类实例化以后**运行，即后处理器

​	对于**事件发布器**来说底层是调用了一个**SimpleApplicationEventMulticaster**的**广播器**来发布的，可以**设置一个线程池ThreadPoolTaskExecutor**，给这个广播器**用setTaskExecutor设置上**，这样就能**异步的发布事件**（注意用来设置这个**广播器bean的方法名**一定是**applicationEventMulticaster**）

​	实现**事件发布器**需要实现**ApplicationEventMulticaster**接口，演示代码如下

```java
@Bean
public ApplicationEventMulticaster applicationEventMulticaster
        (ConfigurableApplicationContext context, ThreadPoolExecutor executor) {
    return new ApplicationEventMulticaster() {
        // 存放所有的监听器
        private List<GenericApplicationListener> listeners = new ArrayList<>();

        // 主要就看这两个方法，其他的方法忽略不实现

        @Override
        // 收集监听器，当添加监听器时就会调用
        public void addApplicationListenerBean(String listenerBeanName) {
            // 根据bean的名字在容器里获得bean对象
            ApplicationListener listener = context.getBean(listenerBeanName, ApplicationListener.class);

            // 还得保存监听器支持的事件类型以供发布事件的时候可以筛选
            // 使用spring的工具类获取支持的事件类型（获取接口的范型类型）
            ResolvableType type = ResolvableType.forClass(listener.getClass())
                    .getInterfaces()[0].getGeneric();
            // 将原始的ApplicationListener封装为GenericApplicationListener用以保存支持的事件类型
            GenericApplicationListener genericApplicationListener = new GenericApplicationListener() {
                // 需要实现两个方法
                @Override
                // 是否支持某事件类型
                public boolean supportsEventType(ResolvableType eventType) {
                    // 右边这个类型是不是能赋值给左边这个类型
                    return type.isAssignableFrom(eventType);
                }

                @Override
                // 执行监听器的方法
                public void onApplicationEvent(ApplicationEvent event) {
                    // 这里直接执行原始listener就好了
                    listener.onApplicationEvent(event);
                }
            };
            // 封装好后再添加进所有的监听器里
            listeners.add(genericApplicationListener);
        }

        @Override
        // 发布事件
        public void multicastEvent(ApplicationEvent event, ResolvableType eventType) {
            // 遍历所有的监听器然后发布事件，实现广播
            for (GenericApplicationListener listener : listeners) {
                // 先判断是否能检查通过
                if (listener.supportsEventType(eventType)) {
                    // 再发给它（调用多线程的方法）
                    executor.execute(()->listener.onApplicationEvent(event));
                }
            }
        }

    };
}
```
