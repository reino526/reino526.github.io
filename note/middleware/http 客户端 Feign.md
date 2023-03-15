---
sort: 5
---



# http客户端Feign



### 基本使用

​	以前利用RestTemplate发起远程调用的代码：

```java
String url = "http://userservice/user/" + 参数;
User user = restTemplate.getForObject(url,User.class);
```

​	存在下面的问题：

- 代码可读性差，编程体验不统一
- 参数复杂URL难以维护

​	Feign是一个声明式的http客户端，可以帮助我们优雅的实现http请求的发送

- 使用它需要引入依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

- 然后在服务的启动类上加上注解

```java
@EnableFeignClients
```

- 然后编写Feign客户端接口，例子如下

```java
@FeignClient("userservice")  // 服务名称
public interface UserClient{
    @GetMapping("/user/{id}")  // 请求方法和请求路径
    User findById(@PathVariable("id") Long id);  // 请求参数和返回值类型
}
```

- 自动注入后在需要的地方直接调用就好了

```java
User user = userCilent.findById(参数);
```



### 自定义配置

|        类型         |       作用       |                          说明                          |
| :-----------------: | :--------------: | :----------------------------------------------------: |
| feign.Logger.Level  |   修改日志级别   |     包含四种不同的级别：NONE、BASIC、HEADERS、FULL     |
| feign.codec.Decoder | 响应结果的解析器 | http远程调用的结果做解析，例如解析json字符串为java对象 |
| feign.codec.Encoder |   请求参数编码   |          将请求参数编码，便于通过http请求发送          |
|   feign.Contract    |  支持的注解格式  |                 默认是SpringMVC的注解                  |
|    feign.Retryer    |   失败重试机制   | 请求失败的重试机制，默认是没有，不过会使用Ribbon的重试 |

​	配置Feign日志有两种方式：

- 方式一：使用配置文件修改

```yaml
feign:
	cilent:
		config:
			default:  # 这里用default就是全局配置，如果是写服务名称则是针对某个微服务的配置
				loggerLevel: FULL
```

- 方式二：java代码方式

```java
// 首先声明一个bean
public class FeignClientConfiguration{
    @Bean
    public Logger.Level feignLogLevel(){
        return Logger.Level.BASIC;
    }
}
// 如果是全局配置就这样配
@EnableFeignClients(defaultConfiguration = FeignClientConfiguration.class)
// 如果是局部配置，则这样配
@FeignClient(value = "userservice", configuration = FeignClientConfiguration.class)
```



### Feign 的性能优化

​	Feign底层的客户端实现：

- URLConnection：默认实现，不支持连接池
- Apache HttpClient：支持连接池
- OKHttp：支持连接池

​	优化Feign的性能主要包括：

- 使用连接池代替默认的URLConnection
- 日志级别，最好使用basic或none

​	优化的步骤：

- 引入依赖

```xml
<denpency>
	<groupId>io.github.openfeign</groupId>
	<artifactId>feign-httpclient</artifactId>
</denpency>
```

- 配置连接池

```yml
feign:
	cilent:
		config:
			default: 
				loggerLevel: BASIC
	httpclient:
		enabled: true  # 开启feign对HttpClient的支持
		max-connections: 200  # 最大的连接数
		max-connections-per-route: 50  # 每个路径的最大连接数
```



### Feign 的两种实践方法

​	第一种：开发一个接口UserInterface，因为在UserClient和UserController中对应的方法格式时一模一样的，所以将UserClient继承UserInterface，而UserController实现UserInterface接口，这种方式就是耦合性太大

​	第二种：将UserClienet甚至包括User都放到一个api包里，当需要用到的时候就导入这个包调用里面的方法

​	而对于第二种方法由于不在同一个包里，@FeignClient无法解析，spring容器中不会有响应的UserClient实例，有两种解决方法：

- 指定UserClient所在的包

```java
@EnableFeignClients(basePackages = "com.springcloud.feign.clients")
```

- 指定UserClient字节码

```java
@EnableFeignClients(clients = {UserClient.class})
```

