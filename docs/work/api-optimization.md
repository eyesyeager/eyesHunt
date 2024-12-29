# 定位耗时节点

定位性能瓶颈有两个思路，一个是通过工具去监控，一个是通过经验去猜想。

就工具而言，推荐使用 [arthas](https://arthas.aliyun.com/doc)。具体安装适用这里就不多介绍了，可以查看官方文档，如果是在阿里工作的话，arthas 可以说是必备技能。

arthas 的 trace 命令可以查看当前方法内，各个子函数的耗时情况，如下：

```
trace com.xxx.service.impl.AServiceImpl refresh
```

其中耗时最多的子函数会被标红色：

```
Affect(class-cnt:2 , method-cnt:2) cost in 525 ms.
`---ts=2020-0X-0Y 13:33:18;thread_name=DubboServerHandler-127.0.0.1:20880-thread-36;id=24e;is_daemon=true;priority=5;TCCL=com.mmm.WWWClassLoader@4362d7df
    `---[1761.834357ms] com.xxx.service.impl.AServiceImpl$$EnhancerBySpringCGLIB$$e3cd7543:refresh()
        +---[0.017066ms] com.xxx.service.impl.AServiceImpl$$EnhancerBySpringCGLIB$$e3cd7543:$jacocoInit()
        `---[1761.00347ms] org.springframework.cglib.proxy.MethodInterceptor:intercept()
            `---[1757.647111ms] com.xxx.service.impl.AdServiceImpl:refresh()
                +---[0.006629ms] com.xxx.biz.yyy.service.impl.AServiceImpl:$jacocoInit()
                +---[0.004073ms] java.util.Collections:singletonList()
                +---[1709.203302ms] com.yyy.service.impl.AServiceImpl:refreshSomeThings()
                `---[48.135719ms] com.yzzzz.service.impl.AServiceImpl:createSurvey()
```

继续 trace 耗时最多的子函数，最后就可以定位到问题函数上。

当然对于大公司来说，项目比较复杂，RPC 调用比较多，这时候一般需要用到链路跟踪系统，快速定位耗时服务，然后再进入服务用 arthas 一步一步排查。

# 常见优化方案

![优化方案](http://oss.eyescode.top/eyeshunt/content/9d3eb398-c2c3-947f-729a-4b75f51b936a.png)

------
摘自：
+ [如何快速定位接口响应慢问题](https://blog.csdn.net/zj15527620802/article/details/115182528)
+ [用了这18种方案，接口性能提高了100倍！](https://juejin.cn/post/7167153109158854687)

站长略有修改