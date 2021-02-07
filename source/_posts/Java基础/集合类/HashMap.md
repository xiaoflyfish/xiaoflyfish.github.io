---
title: HashMap
categories: 
- Java基础
- 集合类
---

# 整体架构

HashMap 底层的数据结构主要是：数组 + 链表 + 红黑树。其中当链表的长度大于等于 8 时，链表会转化成红黑树，当红黑树的大小小于等于 6 时，红黑树会转化成链表，整体的数据结构如下：

<img src="https://xiaoflyfish.oss-cn-beijing.aliyuncs.com/image/20201127095949.png" style="zoom:30%;" />

# 类注释

从 HashMap 的类注释中，我们可以得到如下信息：

- 允许 null 值，不同于 `HashTable` ，是线程不安全的；
- load factor（影响因子） 默认值是 0.75， 是均衡了时间和空间损耗算出来的值，较高的值会减少空间开销（扩容减少，数组大小增长速度变慢），但增加了查找成本（`hash` 冲突增加，链表长度变长），不扩容的条件：数组容量 > 需要的数组大小 /load factor；
- 如果有很多数据需要储存到 HashMap 中，建议 `HashMap` 的容量一开始就设置成足够的大小，这样可以防止在其过程中不断的扩容，影响性能；
- HashMap 是非线程安全的，我们可以自己在外部加锁，或者通过 `Collections#synchronizedMap` 来实现线程安全，`Collections#synchronizedMap` 的实现是在每个方法上加上了 synchronized 锁；
- 在迭代过程中，如果 HashMap 的结构被修改，会快速失败。

**HashMap 源码中包含了以下几个属性：**

```java
 //初始容量为 16
 static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;
 
 //最大容量
 static final int MAXIMUM_CAPACITY = 1 << 30;
 
 //负载因子默认值
 static final float DEFAULT_LOAD_FACTOR = 0.75f;
 
 //桶上的链表长度大于等于8时，链表转化成红黑树
 static final int TREEIFY_THRESHOLD = 8;
 
 //桶上的红黑树大小小于等于6时，红黑树转化成链表
 static final int UNTREEIFY_THRESHOLD = 6;
 
 //当数组容量大于 64 时，链表才会转化成红黑树
 static final int MIN_TREEIFY_CAPACITY = 64;
 
 //记录迭代过程中 HashMap 结构是否发生变化，如果有变化，迭代时会 fail-fast
 transient int modCount;
 
 //HashMap 的实际大小，可能不准(因为当你拿到这个值的时候，可能又发生了变化)
 transient int size;
 
 //存放数据的数组
 transient Node<K,V>[] table;
 
 // 扩容的门槛，有两种情况
 // 如果初始化时，给定数组大小的话，通过 tableSizeFor 方法计算，数组大小永远接近于 2 的幂次方，比如你给定初始化大小 19，实际上初始化大小为 32，为 2 的 5 次方。
 // 如果是通过 resize 方法进行扩容，大小 = 数组容量 * 0.75
 int threshold;
 
 //链表的节点
 static class Node<K,V> implements Map.Entry<K,V> {
 
 //红黑树的节点
 static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
```

# 新增

新增 key，value 大概的步骤如下：

1. 空数组有无初始化，没有的话初始化；
2. 如果通过 key 的 hash 能够直接找到值，跳转到 6，否则到 3；
3. 如果 hash 冲突，两种解决方案：链表 or 红黑树；
4. 如果是链表，递归循环，把新元素追加到队尾；
5. 如果是红黑树，调用红黑树新增的方法；
6. 通过 2、4、5 将新元素追加成功，再根据 onlyIfAbsent 判断是否需要覆盖；
7. 判断是否需要扩容，需要扩容进行扩容，结束。

<img src="https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/053f9916a958437880f1abeff91b1205~tplv-k3u1fbpfcp-watermark.image" style="zoom:33%;" />

```java
public V put(K key, V value) {
    // 对 key 进行哈希操作
    return putVal(hash(key), key, value, false, true);
}
// 入参 hash：通过 hash 算法计算出来的值。
// 入参 onlyIfAbsent：false 表示即使 key 已经存在了，仍然会用新值覆盖原来的值，默认为 false
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    // n 表示数组的长度，i 为数组索引下标，p 为 i 下标位置的 Node 值
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    //如果数组为空，使用 resize 方法初始化
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    // 如果当前索引位置是空的，直接生成新的节点在当前索引位置上
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    // 如果当前索引位置有值的处理方法，即我们常说的如何解决 hash 冲突
    else {
        // e 当前节点的临时变量
        Node<K,V> e; K k;
        // 如果 key 的 hash 和值都相等，直接把当前下标位置的 Node 值赋值给临时变量
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        // 如果是红黑树，使用红黑树的方式新增
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        // 是个链表，把新节点放到链表的尾端
        else {
            // 自旋
            for (int binCount = 0; ; ++binCount) {
                // e = p.next 表示从头开始，遍历链表
                // p.next == null 表明 p 是链表的尾节点
                if ((e = p.next) == null) {
                    // 把新节点放到链表的尾部 
                    p.next = newNode(hash, key, value, null);
                    // 当链表的长度大于等于 8 时，链表转红黑树
                    if (binCount >= TREEIFY_THRESHOLD - 1)
                        treeifyBin(tab, hash);
                    break;
                }
                // 链表遍历过程中，发现有元素和新增的元素相等，结束循环
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                //更改循环的当前元素，使 p 在遍历过程中，一直往后移动。
                p = e;
            }
        }
        // 说明新节点的新增位置已经找到了
        if (e != null) {
            V oldValue = e.value;
            // 当 onlyIfAbsent 为 false 时，才会覆盖值 
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            // 返回老值
            return oldValue;
        }
    }
    // 记录 HashMap 的数据结构发生了变化
    ++modCount;
    //如果 HashMap 的实际大小大于扩容的门槛，开始扩容
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

**为什么是 8，这个答案在源码中注释有说，中文翻译过来大概的意思是：**

链表查询的时间复杂度是 `O (n)`，红黑树的查询复杂度是 `O (log (n))`。在链表数据不多的时候，使用链表进行遍历也比较快，只有当链表数据比较多的时候，才会转化成红黑树，但红黑树需要的占用空间是链表的 2 倍，考虑到转化时间和空间损耗，所以我们需要定义出转化的边界值。

在考虑设计 8 这个值的时候，我们参考了泊松分布概率函数，由泊松分布中得出结论，链表各个长度的命中概率为：

```
* 0:    0.60653066
* 1:    0.30326533
* 2:    0.07581633
* 3:    0.01263606
* 4:    0.00157952
* 5:    0.00015795
* 6:    0.00001316
* 7:    0.00000094
* 8:    0.00000006
```

意思是，当链表的长度是 8 的时候，出现的概率是 0.00000006，不到千万分之一，所以说正常情况下，链表的长度不可能到达 8 ，而一旦到达 8 时，肯定是 hash 算法出了问题，所以在这种情况下，为了让` HashMap `仍然有较高的查询性能，所以让链表转化成红黑树，我们正常写代码，使用 HashMap 时，几乎不会碰到链表转化成红黑树的情况，毕竟概念只有千万分之一。

**红黑树新增节点过程**

1. 首先判断新增的节点在红黑树上是不是已经存在，判断手段有如下两种：

   1.1. 如果节点没有实现 `Comparable` 接口，使用 equals 进行判断；

   1.2. 如果节点自己实现了 Comparable 接口，使用 `compareTo` 进行判断。

2. 新增的节点如果已经在红黑树上，直接返回；不在的话，判断新增节点是在当前节点的左边还是右边，左边值小，右边值大；

3. 自旋递归 1 和 2 步，直到当前节点的左边或者右边的节点为空时，停止自旋，当前节点即为我们新增节点的父节点；

4. 把新增节点放到当前节点的左边或右边为空的地方，并于当前节点建立父子节点关系；

5. 进行着色和旋转，结束。

```java
//入参 h：key 的hash值
final TreeNode<K,V> putTreeVal(HashMap<K,V> map, Node<K,V>[] tab,
                               int h, K k, V v) {
    Class<?> kc = null;
    boolean searched = false;
    //找到根节点
    TreeNode<K,V> root = (parent != null) ? root() : this;
    //自旋
    for (TreeNode<K,V> p = root;;) {
        int dir, ph; K pk;
        // p hash 值大于 h，说明 p 在 h 的右边
        if ((ph = p.hash) > h)
            dir = -1;
        // p hash 值小于 h，说明 p 在 h 的左边
        else if (ph < h)
            dir = 1;
        //要放进去key在当前树中已经存在了(equals来判断)
        else if ((pk = p.key) == k || (k != null && k.equals(pk)))
            return p;
        //自己实现的Comparable的话，不能用hashcode比较了，需要用compareTo
        else if ((kc == null &&
                  //得到key的Class类型，如果key没有实现Comparable就是null
                  (kc = comparableClassFor(k)) == null) ||
                  //当前节点pk和入参k不等
                 (dir = compareComparables(kc, k, pk)) == 0) {
            if (!searched) {
                TreeNode<K,V> q, ch;
                searched = true;
                if (((ch = p.left) != null &&
                     (q = ch.find(h, k, kc)) != null) ||
                    ((ch = p.right) != null &&
                     (q = ch.find(h, k, kc)) != null))
                    return q;
            }
            dir = tieBreakOrder(k, pk);
        }
 
        TreeNode<K,V> xp = p;
        //找到和当前hashcode值相近的节点(当前节点的左右子节点其中一个为空即可)
        if ((p = (dir <= 0) ? p.left : p.right) == null) {
            Node<K,V> xpn = xp.next;
            //生成新的节点
            TreeNode<K,V> x = map.newTreeNode(h, k, v, xpn);
            //把新节点放在当前子节点为空的位置上
            if (dir <= 0)
                xp.left = x;
            else
                xp.right = x;
            //当前节点和新节点建立父子，前后关系
            xp.next = x;
            x.parent = x.prev = xp;
            if (xpn != null)
                ((TreeNode<K,V>)xpn).prev = x;
            //balanceInsertion 对红黑树进行着色或旋转，以达到更多的查找效率，着色或旋转的几种场景如下
            //着色：新节点总是为红色；如果新节点的父亲是黑色，则不需要重新着色；如果父亲是红色，那么必须通过重新着色或者旋转的方法，再次达到红黑树的5个约束条件
            //旋转： 父亲是红色，叔叔是黑色时，进行旋转
            //如果当前节点是父亲的右节点，则进行左旋
            //如果当前节点是父亲的左节点，则进行右旋
          
            //moveRootToFront 方法是把算出来的root放到根节点上
            moveRootToFront(tab, balanceInsertion(root, x));
            return null;
        }
    }
}
```

# 查找

HashMap 的查找主要分为以下三步：

- 根据 hash 算法定位数组的索引位置，`equals` 判断当前节点是否是我们需要寻找的 key，是的话直接返回，不是的话往下。
- 判断当前节点有无 `next `节点，有的话判断是链表类型，还是红黑树类型。
- 分别走链表和红黑树不同类型的查找方法。

```java
public V get(Object key) {
    Node<K,V> e;
    // 对 key 进行哈希操作
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}
final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    // 非空判断
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        // 判断第一个元素是否是要查询的元素
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        // 下一个节点非空判断
        if ((e = first.next) != null) {
            // 如果第一节点是树结构，则使用 getTreeNode 直接获取相应的数据
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            do { // 非树结构，循环节点判断
                // hash 相等并且 key 相同，则返回此节点
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```

```java
// 采用自旋方式从链表中查找 key，e 初始为为链表的头节点
do {
    // 如果当前节点 hash 等于 key 的 hash，并且 equals 相等，当前节点就是我们要找的节点
    // 当 hash 冲突时，同一个 hash 值上是一个链表的时候，我们是通过 equals 方法来比较 key 是否相等的
    if (e.hash == hash &&
        ((k = e.key) == key || (key != null && key.equals(k))))
        return e;
    // 否则，把当前节点的下一个节点拿出来继续寻找
} while ((e = e.next) != null);
```

**红黑树查找：**

1. 从根节点递归查找；
2. 根据 `hashcode`，比较查找节点，左边节点，右边节点之间的大小，根本红黑树左小右大的特性进行判断；
3. 判断查找节点在第 2 步有无定位节点位置，有的话返回，没有的话重复 2，3 两步；
4. 一直自旋到定位到节点位置为止。

如果红黑树比较平衡的话，每次查找的次数就是树的深度。

# 扩容

**什么时候进行扩容**

当hashmap中的元素个数超过数组大小loadFactor时，就会进行数组扩容，loadFactor的默认值为0.75，也就是说，默认情况下，数组大小为16，那么当hashmap中元素个数超过`16*0.75=12`的时候，就把数组的大小扩展为`2*16=32`，即扩大一倍，然后重新计算每个元素在数组中的位置，而这是一个非常消耗性能的操作，所以如果我们已经预知hashmap中元素的个数，那么预设元素的个数能够有效的提高hashmap的性能。

**为什么扩容是2的次幂**

HashMap的初始容量是2的n次幂，扩容也是2倍的形式进行扩容，是因为容量是2的n次幂，可以使得添加的元素均匀分布在HashMap中的数组上，减少hash碰撞，避免形成链表的结构，使得查询效率降低

容积是2的n次方，是为了保证在做`(length-1)`的时候，每一位都能`&`与运算

**扩容步骤**

当这个数组满了以后，就会自动扩容，变成一个更大的数组，可以在里面放更多的元素。

1.在resize()方法中，定义了`oldCap`参数，记录了原table的长度，定义了`newCap`参数，记录新table长度，newCap是oldCap长度的2倍。

2.循环原table，把原`table`中的每个链表中的每个元素放入新table。

假设table原长度是16，扩容后长度32，那么一个`hash`值在扩容前后的table下标是这么计算的：

判断条件`(e.hash & oldCap) == 0` 计算出来的值，如果等于0，就放入低位节点，低位节点在新数组的位置跟原数组一样。 等于1，就放入高位节点，高位放在新数组的位置等于原来位置在加上原数组的长度

```java
final Node<K,V>[] resize() {
    // 扩容前的数组
    Node<K,V>[] oldTab = table;
    // 扩容前的数组的大小和阈值
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    // 预定义新数组的大小和阈值
    int newCap, newThr = 0;
    if (oldCap > 0) {
        // 超过最大值就不再扩容了
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        // 扩大容量为当前容量的两倍，但不能超过 MAXIMUM_CAPACITY
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    // 当前数组没有数据，使用初始化的值
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    else {               // zero initial threshold signifies using defaults
        // 如果初始化的值为 0，则使用默认的初始化容量
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    // 如果新的容量等于 0
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr; 
    @SuppressWarnings({"rawtypes","unchecked"})
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    // 开始扩容，将新的容量赋值给 table
    table = newTab;
    // 原数据不为空，将原数据复制到新 table 中
    if (oldTab != null) {
        // 根据容量循环数组，复制非空元素到新 table
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                // 如果链表只有一个，则进行直接赋值
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)
                    // 红黑树相关的操作
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order
                    // 链表复制，JDK 1.8 扩容优化部分
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        // 原索引
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        // 原索引 + oldCap
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    // 将原索引放到哈希桶中
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    // 将原索引 + oldCap 放到哈希桶中
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```

# 其他问题

**JDK1.8中对hash算法优化**

JDK1.7中通过`key.hashcode()`得到关键字的hash值

hash算法的优化：

```java
//JDK1.8以后的HashMap部分源码
static final int hash(Object key){
 int h;
 return (key == null)?0(h=key.hashCode())^(h>>>16);
}
```

将32位的hash值右移16位，结果是将高16位推到了低16位上，高16位用0来补齐，优化的实质是将高16位与低16位进行异或运算，对于两个hash值的低16位相等，高16位不等的情况，尽量减少了hash冲突

**寻址算法的优化：**

寻址算法就是对长度为n的数组取模，得到在数组中的位置。根据数学规律，对n取模，就是和n-1进行与运算。与运算的效率远远高于求模运算，所以采用与运算。

当数组长度是2的幂次时，hash值对数组长度取模的效果和`hash&(n-1)`是一样的

**加载因子为什么是 0.75**

加载因子也叫扩容因子或负载因子，用来判断什么时候进行扩容的，假如加载因子是 0.5，`HashMap` 的初始化容量是 16，那么当 HashMap 中有 `16*0.5=8` 个元素时，HashMap 就会进行扩容。

这其实是出于容量和性能之间平衡的结果：

- 当加载因子设置比较大的时候，扩容的门槛就被提高了，扩容发生的频率比较低，占用的空间会比较小，但此时发生 Hash 冲突的几率就会提升，因此需要更复杂的数据结构来存储元素，这样对元素的操作时间就会增加，运行效率也会因此降低；
- 而当加载因子值比较小的时候，扩容的门槛会比较低，因此会占用更多的空间，此时元素的存储就比较稀疏，发生哈希冲突的可能性就比较小，因此操作性能会比较高。

所以综合了以上情况就取了一个 0.5 到 1.0 的平均数 0.75 作为加载因子。

**为什么线程不安全**

HashMap 在并发时可能出现的问题主要是两方面：

- 如果多个线程同时使用 put 方法添加元素，而且假设正好存在两个 put 的 key 发生了碰撞（根据 hash 值计算的 bucket 一样），那么根据 HashMap 的实现，这两个 key 会添加到数组的同一个位置，这样最终就会发生其中一个线程 put 的数据被覆盖
- 如果多个线程同时检测到元素个数超过`数组大小 * loadFactor`，这样就会发生多个线程同时对 Node 数组进行扩容，都在重新计算元素位置以及复制数据，但是最终只有一个线程扩容后的数组会赋给 table，也就是说其他线程的都会丢失，并且各自线程 put 的数据也丢失

**一般用什么作为HashMap的key**

一般用Integer、String这种不可变类当`HashMap`当key，而且String最为常用。

- 因为字符串是不可变的，所以在它创建的时候hashcode就被缓存了，不需要重新计算。这就使得字符串很适合作为Map中的键，字符串的处理速度要快过其它的键对象。这就是HashMap中的键往往都使用字符串。
- 因为获取对象的时候要用到`equals()`和hashCode()方法，那么键对象正确的重写这两个方法是非常重要的,这些类已经很规范的覆写了hashCode()以及equals()方法。

**用可变类当HashMap的key有什么问题**

hashcode可能发生改变，导致put进去的值，无法get出，如下所示

```java
HashMap<List<String>, Object> changeMap = new HashMap<>();
List<String> list = new ArrayList<>();
list.add("hello");
Object objectValue = new Object();
changeMap.put(list, objectValue);
System.out.println(changeMap.get(list));
list.add("hello world");//hashcode发生了改变
System.out.println(changeMap.get(list));
```

输出值如下   

```
java.lang.Object@74a14482
null
```

**实现一个自定义的class作为HashMap的key该如何实现**

记住下面四个原则

* 两个对象相等，hashcode一定相等
* 两个对象不等，hashcode不一定不等
* hashcode相等，两个对象不一定相等
* hashcode不等，两个对象一定不等

**记住如何写一个不可变类**

* 类添加final修饰符，保证类不被继承。
  如果类可以被继承会破坏类的不可变性机制，只要继承类覆盖父类的方法并且继承类可以改变成员变量值，那么一旦子类以父类的形式出现时，不能保证当前类是否可变。

* 保证所有成员变量必须私有，并且加上`final`修饰
  通过这种方式保证成员变量不可改变。但只做到这一步还不够，因为如果是对象成员变量有可能再外部改变其值。

* 不提供改变成员变量的方法，包括setter
  避免通过其他接口改变成员变量的值，破坏不可变特性。

* 通过构造器初始化所有成员，进行深拷贝

* ```java
  public final class MyImmutableDemo {  
      private final int[] myArray;  
      public MyImmutableDemo(int[] array) {  
          this.myArray = array.clone();   
      }   
  }
  ```

* 在getter方法中，不要直接返回对象本身，而是克隆对象，并返回对象的拷贝
  这种做法也是防止对象外泄，防止通过getter获得内部可变成员对象后对成员变量直接操作，导致成员变量发生改变。

**和HashTable的区别**

1.Hashtable 是不允许键或值为 null 的，HashMap 的键值则都可以为 null，在HashMap 中不能用get()方法来判断HashMap中是否存在某个键，而应该用containsKey()方法来判断

2.Hashtable 继承了 Dictionary类，而 HashMap 继承的是 AbstractMap 类，`Dictionary`的子类同时也实现了`Map`接口

3.HashMap 的初始容量为：16，Hashtable 初始容量为：11，两者的负载因子默认都是：0.75，当现有容量大于`总容量 * 负载因子`时，HashMap 扩容规则为当前容量翻倍，Hashtable 扩容规则为当前容量翻倍 + 1，`int newCapacity = (oldCapacity << 1) + 1;`

5.`HashTable`是直接使用key的hashCode(`key.hashCode()`)作为hash值，不像`HashMap`内部使用`static final int hash(Object key)`扰动函数对key的hashCode进行扰动后作为hash值

6.`HashTable`取哈希桶下标是直接用模运算`%`（因为其默认容量也不是2的n次方。所以也无法用位运算替代模运算）

7.Hashtable是线程安全的，方法是Synchronized的

**尾插法好处**

尾插法主要是为了安全，防止环化，因为resize的赋值方式，也就是使用了单链表的头插入方式，同一位置上新元素总会被放在链表的头部位置，在旧数组中同一条Entry链上的元素，通过重新计算索引位置后，有可能被放到了新数组的不同位置上

使用头插会改变链表的上的顺序，但是如果使用尾插，在扩容时会保持链表元素原本的顺序，就不会出现链表成环的问题了