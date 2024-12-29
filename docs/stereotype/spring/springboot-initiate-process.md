![SpringBoot 启动流程](http://oss.eyescode.top/eyeshunt/content/cc5165cd-ba4a-fa17-d2f6-a47fa9561171.jpg)

Springboot启动时，第一件重要事件就是初始化 SpringApplication，并主要完成如下事情。
1. 设置源。实际就是设置 Spring 容器启动时依赖的初始配置类，也就是 Springboot 中的启动类
2. 设置 WEB 应用程序类型。例如可以是 SERVLET，REACTIVE 等
3. 加载并设置 Bootstrapper，ApplicationContextInitializer 和 ApplicationListener
4. 设置应用程序主类的 Class 对象
5. 然后 Springboot 启动时还会开启事件机制，主要就是通过运行时监听器 ventPublishingRunListener 创建事件并分发给对应的 ApplicationListener

再然后会加载外部化配置，也就是得到很重要的 Environment 对象，通过 Environment 对象就可以拿到 Springboot 加载的所有外部化配置。

再然后会完成容器刷新，也就是执行 Spring 中的各种扩展点，初始化各种 bean，这部分逻辑属于是 Spring 的逻辑。除此之外，在容器刷新时，还会完成 WEB 容器的启动，例如启动 Springboot 内嵌的 Tomcat，这部分内容比较多，会在后面单独进行分析。

最后，Springboot 在整个启动流程中，会借助事件机制来发布各种事件，发布事件就是借助于上述提到的 EventPublishingRunListener，这是一个运行时监听器，是 Springboot 中提供的监听器，不要和 Spring 中的 ApplicationListener 混淆了。

------
摘自：
+ [详细分析Springboot启动流程](https://zhuanlan.zhihu.com/p/623428835)

站长略有修改