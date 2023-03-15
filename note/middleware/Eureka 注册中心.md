---
sort: 2
---

# Eureka 注册中心



## Eureka 简介	

​	为了解决上面提到的问题，可以使用Eureka，在Eureka中有一个服务端，而不管是消费者还是提供者都是客户端，当每一个客户端启动时都会去服务端注册服务信息，当有消费者需要消费的时候就会去找服务端要接口，而且客户端会每隔30秒去向服务端发送心跳请求报告健康状态，不健康的就会从服务端中被剔除，服务端会提供服务提供者的列表，由消费者利用负载均衡算法挑选一个



## 搭建 eureka 服务

### eureka 注册中心注册

1. 导入依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
```

2. 在配置类上加上这个注解

```java
@EnableEurekaServer
```

3. 在配置文件里加上

```yaml
server:
  port: 10086  #服务端端口
spring:
  application:
    name: eurekaserver  #服务端的名字
eureka:
  client:
    service-url:  #作为客户端配置服务端的地址
      defaultZone: http://localhost:10086/eurka  #将自己也当作客户端注册进去，以便以后多个服务端协同工作
```



### 服务注册

1. 导入依赖

```xml
<dependency>    
	<groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>```
```

2. 在配置文件加上

```yaml
server:
  port: 10087  #端口
spring:
  application:
    name: userservice  #服务的名字
eureka:
  client:
    service-url:  #作为客户端配置服务端的地址
      defaultZone: http://localhost:10086/eurka
```



### 服务发现

​	1. 在服务（客户端）中请求的URL设置为这个形式（用服务名代替ip和端口）

```java
String url = "http://userservice/user/" + 参数
```

	2. 在服务的类中（客户端）注入RestTemplate并添加负载均衡注解

```java
@Bean
@LoadBalanced
public RestTemplate restTemplate(){
    return new RestTemplate();
}
```

