# 算法介绍

LRU（Least Recently Used，最近最久未使用）算法是⼀种缓存淘汰策略，它是大部分操作系统为最大化页面命中率而广泛采用的一种页面置换算法。该算法的思路是，发生缺页中断时，将最近一段时间内最久未使用的页面置换出去。 从程序运行的原理来看，最近最久未使用算法是比较接近理想的一种页面置换算法，这种算法既充分利用了内存中页面调用的历史信息，又正确反映了程序的局部问题。

假设当前序列为 4 3 4 2 3 1 4 2，物理块有 3 个，则：
1. 4 调入内存 4
2. 3 调入内存 3 4
3. 4 调入内存 4 3
4. 2 调入内存 2 4 3
5. 3 调入内存 3 2 4
6. 1 调入内存 1 3 2（因为最少使用的是 4，所以丢弃 4）
7. 4 调入内存 4 1 3（原理同上）
8. 2 调入内存 2 4 1

规律就是，如果新存入或者访问一个值，则将这个值放在队列开头。如果存储容量超过上限 cap，那么删除队尾元素，再存入新的值。

# 数据结构

⾸先要接收⼀个参数作为缓存的最⼤容量， 然后实现两个 API， ⼀个是 `put(key, val)` ⽅法存⼊键值对，`get(key)` ⽅法获取 key 对应的 val， 如果 key 不存在则返回 null，get 和 put ⽅法必须都是 O(1) 的时间复杂度。

因此 LRU cache 的数据结构的必要的条件：查找快，插⼊快，删除快，有顺序之分。那么什么数据结构同时符合上述条件呢？哈希表查找快，但是数据⽆固定顺序； 链表有顺序之分，插⼊删除快，但是查找慢。所以结合⼀下，形成⼀种新的数据结构：哈希链表。如下图所示：

![哈希链表](http://oss.eyescode.top/eyeshunt/content/6e38cd32-7ec3-8ca5-9739-83af7780f0ae.png)

# 算法实现

```java
public class LRUCache {
    class DLinkedNode {
        int key;
        int value;
        DLinkedNode prev;
        DLinkedNode next;
        public DLinkedNode() {}
        public DLinkedNode(int _key, int _value) {key = _key; value = _value;}
    }
 
    private Map<Integer, DLinkedNode> cache = new HashMap<Integer, DLinkedNode>();
    private int size;
    private int capacity;
    private DLinkedNode head, tail;
 
    public LRUCache(int capacity) {
        this.size = 0;
        this.capacity = capacity;
        // 使用伪头部和伪尾部节点
        head = new DLinkedNode();
        tail = new DLinkedNode();
        head.next = tail;
        tail.prev = head;
    }
 
    public int get(int key) {
        DLinkedNode node = cache.get(key);
        if (node == null) {
            return -1;
        }
        // 如果 key 存在，先通过哈希表定位，再移到头部
        moveToHead(node);
        return node.value;
    }
 
    public void put(int key, int value) {
        DLinkedNode node = cache.get(key);
        if (node == null) {
            // 如果 key 不存在，创建一个新的节点
            DLinkedNode newNode = new DLinkedNode(key, value);
            // 添加进哈希表
            cache.put(key, newNode);
            // 添加至双向链表的头部
            addToHead(newNode);
            ++size;
            if (size > capacity) {
                // 如果超出容量，删除双向链表的尾部节点
                DLinkedNode tail = removeTail();
                // 删除哈希表中对应的项
                cache.remove(tail.key);
                --size;
            }
        }
        else {
            // 如果 key 存在，先通过哈希表定位，再修改 value，并移到头部
            node.value = value;
            moveToHead(node);
        }
    }
 
    private void addToHead(DLinkedNode node) {
        node.prev = head;
        node.next = head.next;
        head.next.prev = node;
        head.next = node;
    }
 
    private void removeNode(DLinkedNode node) {
        node.prev.next = node.next;
        node.next.prev = node.prev;
    }
 
    private void moveToHead(DLinkedNode node) {
        removeNode(node);
        addToHead(node);
    }
 
    private DLinkedNode removeTail() {
        DLinkedNode res = tail.prev;
        removeNode(res);
        return res;
    }
}
```

------
摘自：
+ [LRU算法（JAVA实现）](https://blog.csdn.net/qq_42671928/article/details/124698082)

站长略有修改