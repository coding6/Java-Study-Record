# Tomcat总体架构
我们可以将Tomcat粗略的分为两部分

* 接收来自客户端的请求
* 加载并管理servlet，具体处理request

根据这两个大的功能，Tomcat分为了连接器(Connector)和容器(container)来完成这两个事情。
![](/Asset/Tomcat架构.png)

## 连接器(Connector)
连接器是Tomcat中处理请求的模块，那么首先需要知道有哪些类型的请求，Tomcat支持多种IO模型和应用层协议
Tomcat支持的IO模型有：

* NIO
* NIO.2
* APR

Tomcat支持的应用层协议：

* HTTP/1.1
* AJP
* HTTP/2
* 
所以，Tomcat需要有多种连接器来实现这么多的IO模型和应用层协议，我们可以分为以下具体的步骤：

* 监听网络端口
* 接收网络请求
* 读取网络请求的字节流
* 根据特定协议解析数据流，生成统一的Tomcat request对象
* 将Tomcat request对象转换为标准的ServletRequest对象
* 调用Servlet容器，拿到ServletResponse
* ServletResponse转换为Tomcat Response
* Tomcat Response转换为字节流
* 字节流写回给浏览器

根据这些需求，我们可以一步步剖析Tomcat的连接器结构了，我们发现上述的需求中，**网络请求处理**，**协议解析**，**Tomcat data与Servlet data之间的转化**这三个模块属于高内聚的模块，Tomcat中抽象出Endpoint、Processor 和 Adapter三个模块来处理。所以整体的处理逻辑变为了，Endpoint接收网络请求，将字节流给Processor，Processor负责将二进制字节流转换为Tomcat Request，然后将TomcatRequest传递给Adapter，Adapter将TomcatRequest转换为标准的ServletRequest，三个组件之间通过接口进行交互。在Tomcat中Endpoint、Processor被抽象成了ProtocolHandler组件。

![](/Asset/Tomcat架构2.png)
### ProtocolHandler组件
ProtocolHandler组件是由Endpoint、Processor组成的，他的作用就是处理网络的IO请求并将二进制字节流转换为Tomcat Request。

* Endpoint
Endpoint 是通信端点，即通信监听的接口，是具体的 Socket 接收和发送处理器，是对传输层的抽象，因此 Endpoint 是用来实现 TCP/IP 协议的。

Endpoint 是一个接口，对应的抽象实现类是 AbstractEndpoint，而 AbstractEndpoint 的具体子类，比如在 NioEndpoint 和 Nio2Endpoint 中，有两个重要的子组件：Acceptor 和 SocketProcessor。

其中 Acceptor 用于监听 Socket 连接请求。SocketProcessor 用于处理接收到的 Socket 请求，它实现 Runnable 接口，在 run 方法里调用协议处理组件 Processor 进行处理。为了提高处理能力，SocketProcessor 被提交到线程池来执行

* processor
如果说 Endpoint 是用来实现 TCP/IP 协议的，那么 Processor 用来实现 HTTP 协议，Processor 接收来自 Endpoint 的 Socket，读取字节流解析成 Tomcat Request 和 Response 对象，并通过 Adapter 将其提交到容器处理，Processor 是对应用层协议的抽象。

Processor 是一个接口，定义了请求的处理等方法。它的抽象实现类 AbstractProcessor 对一些协议共有的属性进行封装，没有对方法进行实现。具体的实现有 AjpProcessor、Http11Processor 等，这些具体实现类实现了特定协议的解析方法和请求处理方式。

![](/Asset/Tomcat架构3.png)

### Adapter组件
由于协议不同，客户端发过来的请求信息也不尽相同，Tomcat 定义了自己的 Request 类来“存放”这些请求信息。ProtocolHandler 接口负责解析请求并生成 Tomcat Request 类。

但是这个 Request 对象不是标准的 ServletRequest，也就意味着，不能用 Tomcat Request 作为参数来调用容器。Tomcat 设计者的解决方案是引入 CoyoteAdapter，这是适配器模式的经典运用，连接器调用 CoyoteAdapter 的 sevice 方法，传入的是 Tomcat Request 对象，CoyoteAdapter 负责将 Tomcat Request 转成 ServletRequest，再调用容器的 service 方法。

## 怎么设计实现一次性启停Tomca