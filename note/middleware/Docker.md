---
sort: 7
---



# Docker

​	

## 简介

​	由于大型项目组件较多，运行环境也较为复杂，部署项目时会碰到一些问题：

- 依赖关系负责，容易出现兼容性问题
- 开发、测试、生产环境有差异

​	Docker解决依赖的兼容问题：

- 将应用的Libs（函数库）、Deps（依赖）、配置与应用一起打包
- 将每个应用放到一个隔离容器去运行，避免互相干扰

​	Docker解决不同系统环境的问题：

- Docker将用户程序与所需要调用的系统（比如Ubuntu）函数库一起打包
- Docker运行到不同操作系统时，直接基于打包的库函数，借助于操作系统的Linux内核来运行

​	总的来说Docker是一个快速交付应用、运行应用的技术：

1. 可以将程序及其依赖、运行环境一起打包成为一个镜像，可以迁移到任意的Linux操作系统
2. 运行时利用沙箱机制形成隔离容器，各个应用互不干扰
3. 启动、移除都可以通过一行命令完成，方便快捷

​	Docker与虚拟机：虚拟机是在操作系统中模拟硬件设备，然后运行另一个操作系统，比如在Windows系统里面运行Ubuntu系统，这样就可以运行任意的Ubuntu应用了，Docker是一个系统进程，虚拟机是在操作系统中的操作系统



## 初识 Docker

​	镜像（Image）：Docker将应用程序及其所需的依赖、函数库、环境、配置等文件打包在一起，称为镜像，是只读的

​	容器（Container）：镜像中的应用程序运行后形成的进程就是容器，只是Docker会给容器做隔离，对外不可见

​	DockerHub：DockerHub是一个Docker镜像的托管平台，这种平台称作为Docker Registry，国内也有类似的如网易云镜像服务、阿里云镜像等

​	Docker是一个C/S架构的程序，由两部分组成：

- 服务端：Docker守护进程，负责处理Docker指令，管理镜像、容器等
- 客户端：通过命令或RestAPI向Docker服务端发送指令，可以在本地或远程向服务端发送指令



## Docker 基本操作	

​	镜像名称一般分两部分组成：[repository]:[tag]，在没有指定tag时，默认是latest，代表最新版本的镜像

- docker build：从文件里构建镜像
- docker pull：从Docker registry拉取镜像
- docker images：查看镜像
- docker rmi：删除镜像
- docker push：推送镜像到Docker registry
- docker save：保存镜像为一个压缩包
- docker load：加载压缩包为镜像
- docker --help：获取帮助
- docker run：从镜像创建容器并运行
- docker pause或unpause：将运行或暂停中的容器变成暂停或运行
- docker stop：将运行中的容器停止
- docker start：将停止中的容器恢复运行
- docker ps：查看所有运行的容器及状态
- docker logs：查看容器运行的日志
- docker exec：进入容器执行指令
- docker rm：删除指定容器

​	容器与数据耦合的问题：

- 不便于修改：需要进入容器内部修改，很不方便
- 数据不可复用：在容器内的修改对外是不可见的，所有修改对新建的容器是不可复用的
- 升级维护困难：数据在容器内，如果要升级必然删除旧容器，所有数据读跟着删除

​	数据卷：是一个虚拟目录，指向宿主文件系统中的某个目录

​	数据卷操作的基本语法docker volume [COMMAND]，根据不同的命令有不同的效果：

- create：创建一个volume
- inspect：显示一个或多个volume的信息
- ls：列出所有的volume
- prune：删除未使用的volume
- rm：删除一个或多个指定的volume

​	当使用docker run启动容器的时候可以加上-v挂载数据卷，挂载的目录如果不存在也会自动创建，例：

```bash
docker run --name mn -v html:/root/html -p 8080:80 nginx
```

​	挂载也可以直接挂载目录，不一定要数据卷，可以同时挂载好几个，使用几个-v参数



## 自定义镜像

​	镜像结构：

- 入口（Entrypoint）：镜像运行入口，一般是程序启动的脚本和参数
- 层（Layer）：在BaseImage基础上添加安装包、依赖、配置等，每次操作都形成新的一层
- 基础镜像（BaseImage）：应用依赖的系统函数库、环境、配置、文件等

​	自定义镜像就要用到DockerfIle，Dcokerfile就是一个文本文件，其中包含一个个的指令，用指令来说明要执行什么操作来构建镜像，每一个指令都会形成一层Layer，常见的指令：

| 指令       | 说明                                         | 示例                        |
| ---------- | -------------------------------------------- | --------------------------- |
| FROM       | 指定基础镜像                                 | FROM centos:6               |
| ENV        | 设置环境变量，可在后面指令使用               | ENV key value               |
| COPY       | 拷贝本地文件到镜像的指定目录                 | COPY ./mysql-5.7.rpm /tmp   |
| RUN        | 执行Linux的Shell命令，一般是安装过程的命令   | RUN yum install gcc         |
| EXPOSE     | 指定容器运行时监听的端口，是给镜像使用者看的 | EXPOSE 8080                 |
| ENTRYPOINT | 镜像中应用的启动命令，容器运行时调用         | ENTRYPOINT java -jar xx.jar |

​	如果是要构建一个Java项目的镜像可以基于java8:alpine这个基础镜像，可以省很多事



## DockerCompose

​	DockerCompose可以基于Compose文件帮我们快速的部署分布式应用，而无需手动一个个创建和运行容器

​	Compose文件是一个文本文件，通过指令定义集群中的每个容器如何运行，下面给一个示例

```yaml
version: "3.8"

services:
	mysql:
		image: mysql:5.7.25
		environment: MYSQL_ROOT_PASSWORD: 123
		volumes: 
			- /tmp/mysql/data:/var/lib/mysql
			- /tmp/mysql/conf/hmy.cnf:/etc/mysql/conf.d/hmf.cnf
	web:
		build: ./mysql
		ports:
			- 8090:8090
```

​	然后需要在jar包里配置文件里凡是用到localhost的地方都改成对应的服务名，比如jdbc:mysql://localhost:3306改成jdbc:mysql://mysql:3306

​	启动的话就是进入到有docker-compose.yml文件的文件夹中输入docker-compose up指令就可以启动，具体指令可用--help查看，大部分和Docker的命令差不多



## Docker 镜像仓库

​	镜像仓库有公共和私有两种形式：

- 公共仓库：例如Docker官方的Docker Hub，国内也有一些云服务提供类似于Docker Hub的公开服务，比如网易云镜像服务、DaoCloud镜像服务、阿里云镜像服务
- 除了使用公开仓库外，用户还可以在本地搭建私有Docker Registry

​	安装私有仓库也可以通过docker-compose的方式，主要是基于Docker官方提供的DockerRegistry镜像来实现的

​	私服需要做一个配置，因为私服采用的http协议默认不被Docker信任，要在Docker的一个daemon.json文件里添加

```json
"insecure-registries":["私服的ip地址:端口"]
```

​	然后通过`systemctl daemon-reload`重新加载并且使用`systemctl restart docker`重启Docker

​	在私有镜像仓库推送或拉取镜像：

- 重新tag本地镜像，名称前缀为私有仓库的地址：192.168.150.101:8080/

```bash
docker tag nginx:latest 192.168.150.101:8080/nginx:1.0
```

- 推送镜像

```bash
docker push 192.168.150.101:8080/nginx:1.0
```

- 拉取镜像

```bash
docker pull 192.168.150.101:8080/nginx:1.0
```

