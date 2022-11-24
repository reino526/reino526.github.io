---
sort: 3
---

# JAR文件



​	一个 JAR 文件既可以包含类文件，也可以包含图像和声音等其他类型的文件，JAR 文件使用了 ZIP 压缩格式



## 创建JAR文件

​	使用 jar 工具制作 JAR 文件，最常用的指令使用以下语法：

```shell
jar cvf jarFileName file1 file2
```

​	例如：

```shell
jar cvf CalulatorClasses.jar *.class icon.gif
```

​	jar 程序选项：

| 选项 | 说明                                                     |
| ---- | -------------------------------------------------------- |
| e    | 在清单文件创建jar文件入口点                              |
| f    | 将第二个命令行参数作为JAR文件名                          |
| i    | 建立索引文件                                             |
| m    | 将一个清单文件添加到JAR中                                |
| M    | 不创建清单文件                                           |
| c    | 创建一个新的JAR文件并加入文件                            |
| C    | 临时改变目录 `jar cvf jarFileName.jar -C floder *.class` |
| t    | 显示内容表                                               |
| u    | 更新一个已有的JAR文件                                    |
| v    | 生成详细的输出结果                                       |
| x    | 解压文件                                                 |
| 0    | 存储，但不进行ZIP压缩                                    |



## 清单文件

​	每个 JAR 文件包含一个清单文件，描述归档文件的特殊特性，被命名为 **MANIFEST.MF** ，位于 JAR 文件的一个特殊的 **META-INF** 子目录中，最小清单文件极其简单：

```
Manifest-Version: 1.0
```

​	复杂的清单文件被分成多个节，第一节被称为主节，作用于整个 JAR 文件，随后的指定命名实体的属性，以一个 **Name 条目**开始，例如：

```
Manifest-Version:1.0
lines describing this archive

Name: Woozle.class
lines describing this file
Name: com/mycompany/mypkg/
lines describing this package
```



## 可执行JAR文件

​	使用 jar 命令的e选项可以指定程序的入口点，例如：

```
jar cvfe Myprogram.jar com.mycompany.mypkg.MainAppClass [file1 file2...]
```

​	或者可以在清单文件中指定程序的主类，包括以下的语句：

```
Main-Class: com.mycompany.mypkg.MainAppClass
```

```tip
不需要为主类名增加扩展名`.class`，清单文件的最后一行必须以换行符结束，常见的错误就是只包含 Main-Class 行而没有行结束符的
```

