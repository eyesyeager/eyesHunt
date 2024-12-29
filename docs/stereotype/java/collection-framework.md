Java 集合类主要由两个根接口 Collection 和 Map 派生出来的，Collection 派生出了三个子接口：List、Set、Queue（Java5 新增的队列），因此 Java 集合大致也可分成 List、Set、Queue、Map 四种接口体系。

注意：Collection 是一个接口，Collections 是一个工具类，Map 不是 Collection 的子接口。

Java集合框架图如下：

![Java集合框架](http://oss.eyescode.top/eyeshunt/content/26aac0d6-33cb-5b7b-1737-fd3f48f0fc67.png)

说明如下：
+ List 代表了有序可重复集合，可直接根据元素的索引来访问
+ Set 代表无序不可重复集合，只能根据元素本身来访问
+ Queue 是队列集合
+ Map 代表的是存储 key-value 对的集合，可根据元素的 key 来访问 value。

上图中淡绿色背景覆盖的是集合体系中常用的实现类，分别是 ArrayList、LinkedList、ArrayQueue、HashSet、TreeSet、HashMap、TreeMap 等实现类。

------
原作者：库森

原文链接：[常见的集合有哪些？](https://www.javalearn.cn/#/doc/Java%E9%9B%86%E5%90%88/%E9%9D%A2%E8%AF%95%E9%A2%98?id=_1-%e5%b8%b8%e8%a7%81%e7%9a%84%e9%9b%86%e5%90%88%e6%9c%89%e5%93%aa%e4%ba%9b%ef%bc%9f)

站长略有修改
