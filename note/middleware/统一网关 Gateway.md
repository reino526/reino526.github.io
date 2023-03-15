---
sort: 6
---



# 统一网关 Gateway

​	网关功能：

- 身份认证和权限校验
- 服务路由、负载均衡
- 请求限流

​	在SpringCloud中网关的实现包括两种：

- gateway
- zuul

​	Zuul是基于Servlet的实现，属于阻塞式编程，而SpringCloudGateway则是基于Spring5中提供的WebFlux，属于响应式编程的实现，具备更好的性能



## 基本使用

- 引入依赖

```xml
<dependency>
	<groupId>com.alibaba.cloud</groupId>
	<artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
```

- 编写路由配置及nacos地址

```yml
server:
	port: 10010  # 网关端口
spring:
	application:
		name: gateway  # 服务名称
	cloud:
		nacos:
			server-addr: localhost:8848  # nacos地址
		gateway:
			routes:  # 网关路由配置
				- id: user-service  # 路由id，自定义，只要唯一即可
				  uri: lb://userservice  # 路由的目标地址 lb就是负载均衡，后面跟服务名称（也可以写ip，但是写死了不推荐）
				  predicates:  # 路由断言，也就是判断请求是否符合路由规则的条件
				  	- Path=/user/**  # 这个是按照路径匹配，只要以/user/开头就符合要求
```

然后以后只要访问网关的端口就行了，根据后面的路径选择进入哪个服务



## 路由断言工厂

- 在配置文件中写的断言规则只是字符串，这些字符串会被Predicate Factory读取并处理，转变为路由判断的条件
- 例如Path=/user/**是按照路径匹配的，这个规则是由org.springframework.cloud.gateway.handler.predicate.PathRoutePredicateFactory类来处理的
- 像这样的断言工厂在SpringCloudGateway还有十几个

​	Spring提供了11种基本的Predicate工厂：

|    名称    |              说明              |                          示例                          |
| :--------: | :----------------------------: | :----------------------------------------------------: |
|   After    |     是某个时间点之后的请求     | - After=2037-01-20T17:42:47.789-07:00[America/Denver]  |
|   Before   |     是某个时间点之前的请求     | - Before=2037-01-20T17:42:47.789-07:00[America/Denver] |
|  Between   |    是某两个时间点之间的请求    |              - Between=【时间】,【时间】               |
|   Cookie   |     请求必须包含某些cookie     |                - Cookie=chocolate,ch.p                 |
|   Header   |     请求必须包含某些header     |               - Header=X-Request-Id,\d+                |
|    Host    | 请求必须是访问某个host（域名） |      - Host=**.somehost.org,\*\*.anotherhost.org       |
|   Method   |     请求方式必须是指定方式     |                   - Method=GET,POST                    |
|    Path    |    请求路径必须符合指定规则    |             - Path=/red/{segment},/blue/**             |
|   Query    |    请求参数必须包含指定参数    |                   - Query=name,Jack                    |
| RemoteAddr |   请求者的ip必须是指定范围内   |              - RemoteAddr=192.168.1.1/24               |
|   Weight   |            权重处理            |                                                        |



## 路由过滤器

​	GateFilter是网关中提供的一种过滤器，可以对进入网关的请求和微服务返回的响应做处理，Spring提供了31种不同的路由过滤器工厂：

|         名称         |            说明            |
| :------------------: | :------------------------: |
|   AddRequestHeader   |  给当前请求添加一个请求头  |
| RemoveRequestHeader  |   移除请求中的一个请求头   |
|  AddResponseHeader   | 给响应结果中添加一个响应头 |
| RemoveResponseHeader | 从响应结果中移除一个响应头 |
|  RequestRateLimiter  |       限制请求的流量       |

​	配置过滤和配置断言一样，在配置文件里可以设置对每一条路由的：


```yaml
spring:
	cloud:
		gateway:
			routes:
				- id: user-service
				  uri: lb://userservice
				  predicates:
				  	- Path=/user/**
				  filters:  # 配置过滤器
				  	- AddRequestHeader=Truth,The Truth  # 添加请求头，键名为Truth键值为The Truth
```

​	设置全局的要这样：

```yaml
spring:
	cloud:
		gateway:
			default-filters:  # 默认过滤器，对所有的路由都生效
				- AddRequestHeader=Truth,The Truth  # 添加请求头，键名为Truth键值为The Truth
```

​	当然局部的优先级要高于全局的



## 全局过滤器

​	全局过滤器的作用也是处理一切进入网关的请求和微服务响应，与GatewayFilter的作用一样，区别在于GatewayFilter通过配置定义，处理逻辑是固定的，而GlobalFilter的逻辑需要自己写代码实现，定义方式是实现GlobalFilter接口

```java
public interface GlobalFilter{
    // 处理当前请求
    // exchange参数是请求上下文，可以获取request、response等信息
    // chain参数是过滤器链，用来把请求委托给下一个过滤器
    Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain);
}
```

​	下面实现一个通过判断请求头里authorization参数的值是否等于admin来放行的例子：

```java
@Component	// 注入Spring容器
@Order(-1)	// 定义过滤器的顺序，或者可以重写接口的getOrder方法
public class AuthorizeFilter implements GlobalFilter{
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain){
        ServerHttpRequest request = exchange.getRequest();
        MultiValueMap<String, String> params = request.getQueryParmas();
        String auth = parmas.getFirst("authorization");
        if("admin".equals(auth)){
			// 放行
            return chain.filter(exchange);
        }
		// 否则设置状态码并拦截
        exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
        return exchange.getResponse().setComplete();
    }
}
```



## 过滤器执行顺序

​	请求路由后，会将当前路由过滤器和DefaultFilter、GlobalFilter合并到一个过滤器链中，排序后依次执行每个过滤器，对于GlobalFilter会使用适配器将其转为GatewayFileter，最终都是GatewayFileter

​	路由过滤器和DefaultFilter的oreder值由在配置文件中声明的顺序决定，越先声明的oreder值越小，对于order值相同时，会按照DefaultFilter、路由过滤器、GlobalFilter的顺序执行



## 跨域问题

​	跨域问题是指浏览器禁止请求的发起者与服务端发生跨域ajax请求，请求被浏览拦截的问题，网关处理跨域采用的同样是CORS方案（浏览器询问服务器是否允许跨域），并且只需要简单配置即可实现：

```yaml
spring:
	cloud:
		gateway:
			globalcors:  # 全局的跨域处理
				add-to-simple-url-handler-mapping: true  # 解决options请求被拦截的问题
			corsConfigurations:
				'[/**]':
					allowedOrighins:  # 允许哪些网站的跨域请求
                    	- "http://localhost:8090"
                    	- "http://www.leyou.com"
                    allowedMethods:  # 允许的跨域ajax的请求方式
                    	- "GET"
                    	- "POST"
                    allowedHeaders: "*"  # 允许在请求头中携带的头信息
                    allowCredentials: true  # 是否允许携带cookie
                    maxAge: 360000  # 这次跨域检测的有效期
```

