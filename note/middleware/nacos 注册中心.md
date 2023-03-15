---
sort: 4
---

# nacos 注册中心

​	阿里巴巴开发的服务注册和服务发现中心，除了这两个功能以外还有很多功能，对于拉取服务来说，服务消费者会定时拉取服务到服务的缓存里加快响应速度（Eureka也会这样）

​	与Eukera的区别：

- Nacos支持服务端主动检测提供者状态：临时实例采用心跳模式，非临时实例采用主动监测模式
- 临时实例心跳不正常会被剔除，非临时实例则不会被剔除
- Nacos支持服务列表变更的消息推送模式，服务列表更新即时（不然可能会仍然从缓存中调用已经挂掉的服务）
- Nacos集群默认采用AP方式，当集群中存在非临时实例时，采用CP模式，Eureka采用AP方式

​	设置为非临时实例需要在配置文件加上spring.cloud.nacos.discovery.ephemeral: false



### 	服务注册到 nacos

1. 安装后可在bin文件夹中使用cmd并输入`startup.cmd - m standalone`指令启动nacos

2. 在父工程添加管理依赖

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-alibaba-dependencies</artifactId>
    <version>2.2.5.RELEASE</version>
    <type>pom</type>
    <scope>import</scope>
</dependency>
```

3. 添加nacos的客户端依赖

```xml
<dependency>
    <groupId>com.alilbaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>
```

4. 修改配置文件

```yaml
spring:
  cloud:
    nacos:
      server-addr: localhost:8848  #nacos服务端地址，可以在conf/application.properties中配置
```



### 服务多级存储模型

- 一级是服务，例如userservice
- 二级是集群，例如杭州或上海
- 三级是实例，例如杭州机房的某台部署了userservice的服务器

​	设置实例的集群属性在配置文件中的spring.cloud.nacos.discovery.cluster-name属性中设置集群名，而要想访问服务时优先访问相同集群的就需要修改负载均衡策略为com.alibaba.cloud.nacos.ribbon.NacosRule（只能从配置文件里改），如果本地有多个实例就会随机选择一个，如果本地没有就会跨境访问，跨境访问的时候会发出警告

​	Nacos还能够通过权重配置来控制访问效率，权重越大则访问频率越高



### 环境隔离

​	Nacos中服务存储和数据存储的最外层都是一个名为namespace的东西，用来做最外层隔离

​	新建一个命名空间可以在nacos的控制页面里新建，将得到的namespace id设置到服务的配置文件里就可以将这个服务加入这个命名空间，在spring.cloud.nacos.discovery.namespace里设置id，不同namespace下的服务不可见



## Nacos配置管理



### 统一配置管理

​	在nacos的控制页可以添加配置，Data ID的一般格式为【服务名称】-【profile】.【后缀名】，如userservice-dev.yaml，目前文件格式只支持yaml和properties

​	由于需要在读取application.yml配置文件之前就要读取在nacos里的配置文件，所以应该将nacos地址配置到bootstrap.yml文件中，这个配置文件是在读取application.yml之前就会读取，另外读取nacos里的配置需要导入下面这个依赖

```xml
<dependency>
    <groupId>com.alilbaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
</dependency>
```

​	对于bootstrap.yml应该这样写

```yml
spring:
	application:
		name: userservice  #服务名称
	profiles:
		active: dev  #开发环境
	cloud:
		nacos:
			server-addr: localhost:8848  #nacos地址
			cofig:
				file-extension: yaml  #文件后缀名
```

​	读取配置中的属性就是和读取application.yml配置文件一样的



### 配置热更新

​	热更新就是在nacos控制页更改配置文件后就能立马在正在运行的服务上成功实现，两种方法：

- 第一种是在需要读取nacos配置文件的类上加@RefreshScope注解
- 第二种就是使用@ConfigurationProperties注解绑定前缀的配置到一个配置类上



### 多环境配置共享

​	微服务启动时会从nacos读取多个配置文件：

- 【服务名称】-【profile】.【后缀名】，例如userservice-dev.yaml
- 【服务名称】.【后缀名】，例如userservice.yaml

​	无论profile如何变化，【服务名称】.【后缀名】这个文件一定会加载，因此多环境共享配置可以写入这个文件

​	配置优先级：【服务名称】-【profile】 > 【服务名称】>  本地配置



### nacos集群搭建

- 首先在每个nacos的conf目录中将cluster.conf.example文件重命名为cluster.conf
- 然后每个都添加集群中所有nacos的地址
- 然后要到application.properties文件中将每个nacos的端口号改好，使用的数据库技术也配置好
- 然后分别启动nacos，这次不需要加参数直接启动就好了（集群启动）
- 然后需要使用nginx进行负载均衡和请求分发（反向代理），所以得配置其conf目录下的nginx.conf文件，配置示例如下

```nginx
upstream nacos-cluster{
    server 127.0.0.1:8845;
    server 127.0.0.1:8846;
    server 127.0.0.1:8847;
}

server{
    listen 80;
    server_name localhost;
    
    location /nacos{
        proxy_pass http://nacos-cluster;
    }
}
```

- 然后启动nginx就行了

