---
title: LRU算法
categories: 
- 算法
---

**当我们进行 put 操作的时候：**

1.如果要 put已经存在于链表之中了，那么需要把链表中旧的数据删除，然后把新的数据插入到链表的头部

2.如果要 put的数据没有存在于链表之后，需要判断下缓存区是否已满，如果满的话，则把链表尾部的节点删除，之后把新的数据插入到链表头部，如果没有满的话，直接把数据插入链表头部即可

**对于 get 操作：**

1.如果要 get 的数据存在于链表中，则把 value 返回，并且把该节点删除，删除之后把它插入到链表的头部

2.如果要 get 的数据不存在于链表之后，则直接返回 -1 即可

单链表实现put 和 get 都需要遍历链表查找数据是否存在，所以时间复杂度为 O(n)，空间复杂度为 O(1)

# 双向链表+哈希表

可以考虑采用空间换时间的方式来加快我们的 get 操作

例如可以用一个额外哈希表（例如HashMap）来存放 key-value，这样的话，get 操作就可以在 O(1) 的时间内寻找到目标节点，并且把 value 返回了

用了哈希表之后，虽然能够在 O(1) 时间内找到目标元素，不过还需要删除该元素，并且把该元素插入到链表头部啊，删除一个元素，是需要定位到这个元素的前驱的，然后定位到这个元素的前驱，是需要 O(n) 时间复杂度的

把单链表换成双链表，这样的话，我们就可以很好着解决这个问题了

所以采用双向链表 + 哈希表这两种数据结构的组合，我们的 get 操作就可以在 O(1) 时间复杂度内完成了，由于 put 操作我们要删除的节点一般是尾部节点，所以我们可以用一个变量 tai 时刻记录尾部节点的位置，这样的话，我们的 put 操作也可以在 O(1) 时间内完成了

```java
// 链表节点的定义
class LRUNode{
    String key;
    Object value;
    LRUNode next;
    LRUNode pre;
 
    public LRUNode(String key, Object value) {
        this.key = key;
        this.value = value;
    }
}
```

```java
public class LRUCache {
    Map<String, LRUNode> map = new HashMap<>();
    LRUNode head;
    LRUNode tail;
    // 缓存最大容量
    int capacity;
 
    public LRUCache(int capacity) {
        this.capacity = capacity;
    }
 
    public void put(String key, Object value) {
        if (head == null) {
            head = new LRUNode(key, value);
            tail = head;
            map.put(key, head);
        }
        LRUNode node = map.get(key);
        if (node != null) {
            // 更新值
            node.value = value;
            // 把他从链表删除并且插入到头结点
            removeAndInsert(node);
        } else {
            LRUNode tmp = new LRUNode(key, value);
            // 如果会溢出
            if (map.size() >= capacity) {
                // 先把它从哈希表中删除
                map.remove(tail.key);
                // 删除尾部节点
                tail = tail.pre;
                tail.next = null;
            }
            map.put(key, tmp);
            // 插入
            tmp.next = head;
            head.pre = tmp;
            head = tmp;
        }
    }
 
    public Object get(String key) {
        LRUNode node = map.get(key);
        if (node != null) {
            // 把这个节点删除并插入到头结点
            removeAndInsert(node);
            return node.value;
        }
        return null;
    }
    private void removeAndInsert(LRUNode node) {
        // 特殊情况先判断，例如该节点是头结点或是尾部节点
        if (node == head) {
            return;
        } else if (node == tail) {
            tail = node.pre;
            tail.next = null;
        } else {
            node.pre.next = node.next;
            node.next.pre = node.pre;
        }
        // 插入到头结点
        node.next = head;
        node.pre = null;
        head.pre = node;
        head = node;
    }
}
```

