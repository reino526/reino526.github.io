---
sort: 3
---



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

-  Parser**把**String转为其它类型**

-  Formatter**综合**Printer和Parser**功能

-  Converter**把**类型S转为类型T**

-  Printer、Parser、Converter经过**适配转换为GenericConverter**放入**Converters集合**


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