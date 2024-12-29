# 问题排查

**查询哪个进程占用CPU**

可以使用 `Top` 或者 `top | grep 用户名`。比如这里我们可以使用 `top` 查询有哪些进程比较占用 CPU，如下图，可以发现是进程 15913：

![top命令](http://oss.eyescode.top/eyeshunt/content/8c2b7604-c68d-2f5e-0253-4c0dd439c162.png)

**进程哪个线程占用CPU**

接着我们查看上述进程内是哪些线程在捣乱，使用命令`top -H -p PID`，加上 -H 选项可以该进程的相关线程信息。

在这里我们使用`top -H -p 15913`，结果如下图，从下图种可知最耗 CPU 的两个线程 PID 分别是 15924 和 15925，对应的 16 进制为 0x3e34 和 0x3e35：

![排查线程](http://oss.eyescode.top/eyeshunt/content/931ef8df-220b-fcac-cc6b-82a43f63e3a4.png)

**查询线程的堆栈信息**

jstack 命令可以得到线程堆栈信息，根据这些线程堆栈信息，可以去检查 Java 程序出现的问题，如检测死锁。

在这里我们要分两步：
1. 将 tid 转换为 16 进制的数字：printf “%x\n” tid
2. 查询线程信息：jstack 15913 | grep -A 10 0x3e34

由于站长没有执行这个命令，所以在网上找了个类似的图：

![堆栈信息](http://oss.eyescode.top/eyeshunt/content/977ca4d2-ec80-e632-a5ab-243bd4dca419.jpg)

看到具体代码后，就可以根据情况解决了。

# 紧急解决方案

导致 CPU 高占用的因素很多，具体情况具体分析，以下仅供参考：
+ kill问题进程，保全其他程序
+ 紧急扩容
+ 重启服务器
+ 中断造成 CPU 高占用的任务
+ 代码回滚

------
摘自：
+ [CPU飙高问题排查](https://zhuanlan.zhihu.com/p/322597955)
+ [CPU占用100%排查过程](https://cloud.tencent.com/developer/article/1400432)

站长略有修改