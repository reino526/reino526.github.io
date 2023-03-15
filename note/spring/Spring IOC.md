---
sort: 1
---



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





