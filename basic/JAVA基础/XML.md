---
sort: 19
---

# XML



## XML 文档的结构

文档应当以一个文档头开始，例如：

```xml-dtd
<?xml version="1.0" encoding="UTF-8" ?>
```

```tip
文档头是可选的，但是强烈推荐使用文档头
```

文档头之后通常是文档类型定义 （ Document Type Definition, DTD ），例如：

```xml-dtd
<!DOCTYPE web-app PUBLIC
	"-//Sun Microsystems, Inc.//DTD Web Application 2.2//EN"
	"http://java.sun.com/j2ee/dtds/web-app_2_2.dtd">
```

```note
XML 文档中元素要么包含子元素，要么包含文本，不要混合式内容，可以简化解析过程
```

```note
属性只应该用来修改值的解释，而不是用来指定值，许多有用的文档根本不使用属性
```

XML 还有一些标记：

- 字符引用：形式是 `&#十进制值;` 或 `&#x十六进制值;` ，例如 &#233; 可以表示为 \&#233; 或 \&#xE9;
- 实体引用：形式是 `&name;` ，比如 &lt; 表示为 \&lt;
- CDATA 部分：用 `<![CDATA[` 和 `]]>` 来限定其界限，囊括那些含有 \< 、 \> 、 \& 之类字符的字符串而不必进行转义

```tip
CDATA 部分不能包含字符串 ` ]]> `
```

- 处理指令：用 `<?` 和 `?>` 来限定其界限，例如：

```xml-dtd
<?xml-stylesheet href="mystyle.css" type="text/css" ?>
```

- 注释：用 `<!-` 和 `-->` 限定界限

```tip
注释不应该含有字符串 -- 
```



## 文档类型定义

指定文档结构，可以提供一个文档类型定义 （ DTD ） 或一个 XML Schema 定义



### DTD

提供 DTD 的方式有多种，可以直接纳入到 XML 文档中：

```xml-dtd
<?xml version="1.0"?>
<!DOCTYPE config[
        <!ELEMENT config ... >
        ...
]>
<config>
	...
</config>
```

位于由 [ ... ] 限定界限的块中，这种方式不普遍，太长，可以指定一个包含 DTD 的 URL ：

```xml-dtd
<!DOCTYPE config SYSTEM "config.dtd">
<!DOCTYPE config SYSTEM "http://myserver.com/config.dtd">
```

有一个来源于 SGML 的用于识别众所周知的 DTD 的机制：

```xml-dtd
<!DOCTYPE web-app PUBLIC
	"-//Sun Microsystems, Inc.//DTD Web Application 2.2//EN">
```

如果 XML 处理器知道如何定位带有 PUBLIC 公共标识符的 DTD ，那么就不需要 URL 了

规则有 ELEMENT 、 ATTLIST 、 ENTITY 等等，具体上网搜

配置解析器就可以充分利用 DTD ，打开验证特性：

```java
factory.setValidating(true);
```

```note
验证器报告错误时，可以安装一个错误处理器，这个对象实现了 ErrorHandler 接口，通过 DocumentBuilder 类的 setErrorHandler 方法来安装错误处理器
```



### XML Schema

如果要在文档中引用 Schema 文件，需要在根元素中添加属性，例如：

```xml-dtd
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
			xsi:noNamespaceSchemaLocation="config.xsd">
	 ...
</config>
```

这个声明说明 Schema 文件 config.xsd 会被用来验证该文档

Schema 为每个元素和属性都定义了类型，一些简单类型已经被内建到了 XML Schema 内，包括：

```
xsd:string
xsd:int
xsd:boolean
```

```note
可以定义自己的简单类型，可以把类型组合成复杂类型，可以使用 ref 属性来引用在 Schema 中位于别处的定义
```

```note
xsd:sequence 结构和 DTD 中的连接符号等价， xsd:choice 结构和 | 操作符等价，指定属性可以用 xsd:attribute
```

解析带有 Schema 的 XML 文件和解析带有 DTD 的文件类似，但有 2 点差别：

- 必须打开对命名空间的支持，即使在 XML 文件里可能不会用到它

```java
factory.setNamespaceAware(true);
```

- 必须要写下面固定的代码：

```java
final String JAXP_SCHEMA_LANGUAGE = "http://java.sun.com/xml/jaxp/properties/schemaLanguage";
final String W3C_XML_SCHEMA = "http://www.w3.org/2001/XMLSchema";
factory.setAttribute(JAXP_SCHEMA_LANGUAGE, W3C_XML_SCHEMA);
```

```tip
子元素会继承父元素的命名空间，但是属性不会继承，只能显示提供前缀，带前缀的叫限定名
```





## 解析 XML 文档

Java 库提供了两种 XML 解析器：

- 文档对象模型 （ Document Object Model， **DOM** ） 解析器这样的**树型**解析器，将读入的 XML 文档转换为树结构
- XML 简单 API （ Simple API for XML， **SAX** ） 解析器这样的**流机制**解析器，在读入 XML 文档时生成相应的事件



### DOM

要读入一个 XML 文档，首先需要一个 DocumentBuilder 对象，可以从 DocumentBuilderFactory 中得到这个对象，例如：

```java
DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
DocumentBuilder builder = factory.newDocumentBuilder();
```

然后可以从文件中读入某个文档：

```java
File f = ... ;
Document doc = builder.parse(f);
```

```tip
或者可以用一个 URL 对象 或者一个输入流，如果是输入流作为源，那么文档里的相对路径将无法定位
```

可以调用 getDocumentElement 方法获得文档的根元素：

```java
Element root = doc.getDocumentElement();
```

其它方法可以去查看源码了解



### SAX

得到 SAX 解析器：

```java
SAXParserFactory facory = SAXParserFactory.newInstance();
SAXParser parser = factory.newSAXParser();
```

然后就可以开始解析：

```java
parser.parse(source.handler);
```

其中 source 可以是一个文件、一个 URL 字符串或者是一个输入流， handler 属于 DefaultHandler 的一个子类，通过覆盖里面的方法实现自己需要的操作



## XPath 定位信息

通过遍历 DOM 树的众多节点来进行查找会显得有些麻烦，可以通过 XPath 表达式容易的获得，比如 `/html/head/title/text()`

一些语法：

-  `/html/body/form[1]` 表示第一个 form

-  `/html/body/form[1]/@action` 表示第一个 form 的 action 属性

-  `count(/html/body/form)` 返回 body 根元素的 form 子元素的数量

要计算 XPath 表达式，需要从 XPathFactory 创建一个 XPath 对象：

```java
XPathFactory xpfactory = XPathfactory.newInstance();
path = xpfactory.newXPath();
```

然后调用 evaluate 方法来计算 XPath 表达式：

```java
String username = path.evaluate("/html/head/title/text(), doc");
```

```tip
不一定要从根节点开始搜索，可以在第二个参数传入一个节点，就从这个节点开始搜索
```

如果 XPath 表达式产生了一组节点，调用：

```java
XPathNodes result = path.evaluateExpression("/html/body/form", doc, XPathNodes.class);
```

```tip
不知道 XPath 的计算结果是什么，就将返回值类型改为 `XPathEvaluationResult<?>` ，调用其 type 方法返回 XPathEvaluationResult.XPathResultType 里的枚举常量，调用 value 方法返回结果值
```



## 使用 StAX 写出 XML 文档

需要从某个 OutputStream 中构建一个 XMLStreamWriter ，像下面这样：

```java
XMLOutputFactory factory = XMLOutputFactory.newInstance();
XMLStreamWriter writer = factory.createXMLStreamWriter(out);
```

然后调用 XMLStreamWriter 的方法输出就好

```tip
需要人为的关闭 XMLStreamWriter ，因为它没有扩展 AutoCloseable 接口
```

