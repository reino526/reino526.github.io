---
sort: 4
---



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

