---
sort: 2
---



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

