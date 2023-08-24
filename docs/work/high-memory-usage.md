# 问题排查

**查询哪个进程占用内存过高**

可以通过使用 linux 中的 top 命令查询服务器中占用内存的状态。默认是 CPU 排序，我们可以调整为内存排序，整理不同的服务按键不一样，这里是在窗口点击“m”键：

![top](http://hunt-cdn.eyescode.top/content/d6b516d9-20dd-41a7-4951-1a07e084c58e.png)

**找到高内存的类**

使用`jmap -histo`查询当前占用内存较大的类 这里使用下面命令查看占用内存高的前20个类，这里就定位到问题类了，从下图中可以看到相应业务中占用内存比较高的类：

```shell
jmap -histo <pid> | head -20
```

![hmap](http://hunt-cdn.eyescode.top/content/1ef8874f-3fba-79be-f898-9e06e3b16f87.png)

# 紧急解决方案

导致内存高占用的因素很多，具体情况具体分析，以下仅供参考：
+ kill问题进程，保全其他程序
+ 紧急扩容
+ 重启服务器
+ 中断造成内存高占用的任务
+ 代码回滚

------
摘抄：
+ [JAVA 内存使用过高排查方案](https://juejin.cn/post/7184994194987941944)

站长略有修改