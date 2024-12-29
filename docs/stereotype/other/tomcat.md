# tomcat 架构

![架构图](http://oss.eyescode.top/eyeshunt/content/7cb7f95a-31f6-b571-bad6-71642ed4e3ef.png)

+ Server：服务器。Tomcat 就是一个 Server 服务器
+ Service：在服务器中可以有多个 Service，只不过在我们常用的这套 Catalina 容器的 Tomcat 中只包含一个 Service，在 Service 中包含连接器和容器。一个完整的 Service 才能完成对请求的接收和处理
+ 连接器：Coyote 是连接器具体的实现。用于与新来的请求建立连接并解析数据。因为 Tomcat 支持的 IO 模型有 NIO、NIO2、APR，而支持的应用层协议有 HTTP1.1、HTTP2、AJP。所以针对不同的 IO 模型和应用层协议请求，在一个 Service 中可以有多个连接器来适用不同的协议的IO请求
  + EndPoint：Coyote 通信端点，即通信监听的接口，是具体 Socket 接收和发送处理器，是用来实现 TCP/IP 传输协议的
  + Acceptor：用于接收请求的 socket
  + Executor：线程池，在接收到请求的 socket 后会从线程池中分配一条来执行后面的操作
  + Processor ：Coyote 协议处理接口，是用来实现 HTTP 应用层协议的，接收 EndPoint 、容器传来的 Socket 字节流，解析成 request 或 response 对象
  + ProtocolHandler：Coyote 协议接口，通过 EndPoint 和 Processor，实现针对具体协议的处理能力
  + Adapter：容器只负责处理数据，对于请求协议不同的数据，容器会无法处理，所以在 ProtocolHandler 处理生成的 request 对象后，还需要将其转成 Tomcat 定义好的统一格式的 ServletRequest 对象，Adapter 就是用来进行这样的操作的
+ 容器： Tomcat 的核心组件， 用于处理请求并返回数据。Catalina 是其具体的实现
  + Engine：表示整个 Catalina 的 Servlet 引擎，用来管理多个虚拟站点，一个 Service 最多只能有一个 Engine。但是一个 Engine 可以包含多个 Host
  + Host：表示一个主机地址，或者说一个站点，一个 Host 下有可以配置多个 Context
  + Context：表示一个 web 应用，一个 Web 应用可以包含多个 Wrapper
  + Wrapper：表示一个 Servlet，是容器中的最底层组件

------
摘自：
+ [Tomcat基础知识总结](https://topjavaer.cn/web/tomcat.html)

站长略有修改