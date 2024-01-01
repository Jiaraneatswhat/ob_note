- 集合的继承关系

![[collection.svg]]

![[map.svg]]

# 1 ArrayList
## 1.1 fields
```java
private static final int DEFAULT_CAPACITY = 10;

// 空参构造器使用
private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

// 传入的 capacity 为 0 时使用
private static final Object[] EMPTY_ELEMENTDATA = {};

// 用于存储数据的 Object 数组
transient Object[] elementData;

private int size;
```
## 1.2 constructors
```java
public ArrayList() {
	// 创建空的 Object 数组
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;  
}

public ArrayList(int initialCapacity) { 
	// 创建指定长度的 Object 数组
    if (initialCapacity > 0) {  
        this.elementData = new Object[initialCapacity];  
    } else if (initialCapacity == 0) {  
        this.elementData = EMPTY_ELEMENTDATA;  
    } 
}
```
## 1.3 扩容
```java
public boolean add(E e) {
    ensureCapacityInternal(size + 1);  // Increments modCount!!  
    elementData[size++] = e;  
    return true;  
}

private void ensureCapacityInternal(int minCapacity) {  
    ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));  
}


private static int calculateCapacity(Object[] elementData, int minCapacity) {
	// 没有元素，即第一次插入时，取默认容量 10
	// 如果此时已有 10 个元素， 返回 11
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {  
        return Math.max(DEFAULT_CAPACITY, minCapacity);  
    }  
    return minCapacity;  
}

private void ensureExplicitCapacity(int minCapacity) {  
    modCount++;  
    // overflow-conscious code  
    if (minCapacity - elementData.length > 0)  
        grow(minCapacity);  
}

// 第一次插入时创建长度为 10 的数组
private void grow(int minCapacity) {  
    // overflow-conscious code  
    int oldCapacity = elementData.length; // 10
    int newCapacity = oldCapacity + (oldCapacity >> 1); // 扩容为 1.5 倍  
    if (newCapacity - minCapacity < 0)  
        newCapacity = minCapacity;  
    if (newCapacity - MAX_ARRAY_SIZE > 0)  
        newCapacity = hugeCapacity(minCapacity);  
    // minCapacity is usually close to size, so this is a win:  
    elementData = Arrays.copyOf(elementData, newCapacity); 
}
```
# 2 HashMap
## 2.1 fields
```java
// 初始容量
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;
// 默认装载因子
static final float DEFAULT_LOAD_FACTOR = 0.75f;
// treeify 阈值
static final int TREEIFY_THRESHOLD = 8;
// untreeify 阈值
static final int UNTREEIFY_THRESHOLD = 6;
// 桶的数量大于 64 时会转换
static final int MIN_TREEIFY_CAPACITY = 64;
```
## 2.2 InnerClass
### 2.2.1 Node
- `Node` 相当于一个桶，`Node` 数组相当于一个 `Hashtable`
```java
// 实现了 Map 接口中定义的 Entry 接口
static class Node<K,V> implements Map.Entry<K,V> {  
    final int hash;  
    final K key;  
    V value;  
    Node<K,V> next;  
  
    Node(int hash, K key, V value, Node<K,V> next) {  
        this.hash = hash;  
        this.key = key;  
        this.value = value;  
        this.next = next;  
    }
}
```
### 2.2.2 TreeNode
```java
// 继承了 LinkedHashMap 的 Entry
// LinkedHashMap 的 Entry 又继承了 HashMap 的 Entry
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {  
    TreeNode<K,V> parent;  // red-black tree links  
    TreeNode<K,V> left;  
    TreeNode<K,V> right;  
    TreeNode<K,V> prev;    // needed to unlink next upon deletion  
    boolean red;  
    TreeNode(int hash, K key, V val, Node<K,V> next) {  
        super(hash, key, val, next);  
    }
}
```
![[hashMap.svg]] 

## 2.3 constructors
```java
public HashMap() {  
    this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted  
}

public HashMap(int initialCapacity, float loadFactor) {  
    this.loadFactor = loadFactor;  
    // 计算出离初始 Capacity 最近的 2 次幂
    this.threshold = tableSizeFor(initialCapacity);  
}
```
## 2.4 插入元素
### 2.4.1 putVal()
```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,  
               boolean evict) {  
    Node<K,V>[] tab; Node<K,V> p; int n, i; 
    // 初始化 table 为 null，见 4.2
    if ((tab = table) == null || (n = tab.length) == 0)  
        n = (tab = resize()).length;
    // 如果 (n - 1) & hash 处有没有桶，即 table 中对应 key 的桶为 null
    if ((p = tab[i = (n - 1) & hash]) == null)  
        tab[i] = newNode(hash, key, value, null);  
    else {  
        Node<K,V> e; K k;  
        // hash 值相同且 key 相同，将 i 处的节点 p 赋给新节点 e
        if (p.hash == hash &&  
            ((k = p.key) == key || (key != null && key.equals(k))))  
            e = p;  
        else if (p instanceof TreeNode)  
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);  
        else {
            for (int binCount = 0; ; ++binCount) {  
	            // 如果 p 是链表  
                if ((e = p.next) == null) { 
	                // 插入链表 
                    p.next = newNode(hash, key, value, null);  
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st  
                        treeifyBin(tab, hash);  
                    break;  
                }  
                // k 存在则覆盖 v
                if (e.hash == hash &&  
                    ((k = e.key) == key || (key != null && key.equals(k))))  
                    break;  
                p = e;  
            }  
        }  
        // 替换 value 的值
        if (e != null) { // existing mapping for key  
            V oldValue = e.value;  
            if (!onlyIfAbsent || oldValue == null)  
                e.value = value;  
            afterNodeAccess(e);  
            return oldValue;  
        }  
    }  
    ++modCount;  
    // 判断是否需要扩容
    if (++size > threshold)  
        resize();  
    afterNodeInsertion(evict);  
    return null;  
}
```
### 2.4.2 首次 resize()
- 第一次扩容，生成一个容量为 16，阈值为 12 的 `HashMap`
```java
// 初始化扩容
final Node<K,V>[] resize() {  
    Node<K,V>[] oldTab = table; // null  
    int oldCap = (oldTab == null) ? 0 : oldTab.length; // 0
    int oldThr = threshold; // 0
    int newCap, newThr = 0;  
    else {               
        // 默认 capacaty 16
        newCap = DEFAULT_INITIAL_CAPACITY;  
        // 默认 threshold 16 * 0.75 = 12
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);  
    }  
    threshold = newThr;  
    @SuppressWarnings({"rawtypes","unchecked"})  
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];  
    table = newTab;  
    return newTab;  
}
```
### 2.4.3 非首次 resize()

![[hashmap_resize.svg]]

- `table` 中的 `bin` 个数达到 `threshold` 时，进行第二次扩容
	- jdk1.7 采用尾插法
	- jdk1.8 采用头插法
```java
final Node<K,V>[] resize() {  
    Node<K,V>[] oldTab = table;  
    int oldCap = (oldTab == null) ? 0 : oldTab.length; // oldCap: 16
    int oldThr = threshold; // oldThr: 12
    int newCap, newThr = 0;  
    if (oldCap > 0) {  
        if (oldCap >= MAXIMUM_CAPACITY) {...}  
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&  
                 oldCap >= DEFAULT_INITIAL_CAPACITY) 
            // oldThr * 2 
            newThr = oldThr << 1; // newThr = 24
    }  
    threshold = newThr;  
    // 将 bin 移动到新的 table 中  
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];  
    table = newTab;  
    if (oldTab != null) {  
        for (int j = 0; j < oldCap; ++j) {  
            Node<K,V> e;  
            if ((e = oldTab[j]) != null) {  
                oldTab[j] = null; // help gc  
                if (e.next == null) // 只有一个元素
                    newTab[e.hash & (newCap - 1)] = e;  
                else if (e instanceof TreeNode)  
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);  
                else { // preserve order  
                    Node<K,V> loHead = null, loTail = null;  
                    Node<K,V> hiHead = null, hiTail = null;  
                    Node<K,V> next;  
                    do {  
                        next = e.next;  
                        // 位置不变
                        // 每次遍历时将新节点添加在尾部
                        if ((e.hash & oldCap) == 0) {
		                    // 第一次遍历 loTail 为 null  
                            if (loTail == null)  
	                            // 将 e 设置为头节点
                                loHead = e;  
                            else 
		                        // 否则添加到 tail 后并设置为 tail 
                                loTail.next = e;  
                            loTail = e;  
                        }  
                        else {
	                        // 同上添加到 hi 链表中  
                            if (hiTail == null)  
                                hiHead = e;  
                            else  
                                hiTail.next = e;  
                            hiTail = e;  
                        }  
                    } while ((e = next) != null);  
                    if (loTail != null) {  
                        loTail.next = null;  
                        newTab[j] = loHead;  
                    }  
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
- `hash` 计算新索引位置
```java
     a.hash       0 0 0 0 0 1 0 1
      n - 1       0 0 0 0 1 1 1 1
----------------------------------
(n - 1) & a.hash   0 0 0 0 0 1 0 1

     b.hash       0 0 0 1 0 1 0 1
      n - 1       0 0 0 0 1 1 1 1
----------------------------------
(n - 1) & b.hash   0 0 0 0 0 1 0 1

扩容后 n 的大小变为 32 相当于多了一个 1

     a.hash       0 0 0 0 0 1 0 1
      n - 1       0 0 0 1 1 1 1 1
----------------------------------
(n - 1) & a.hash   0 0 0 0 0 1 0 1

     b.hash       0 0 0 1 0 1 0 1
      n - 1       0 0 0 1 1 1 1 1
----------------------------------
(n - 1) & b.hash   0 0 0 1 0 1 0 1

新位置比旧位置大 16，即 oldCap
e.hash & oldCap == 0 说明前后索引位置一样
e.hash & oldCap != 0 说明需要移动 oldCap 位置
```
### 2.4.4 rbTree 扩容
#### 2.4.4.1 生成 rbNode 
```java
// putVal
for (int binCount = 0; ; ++binCount) {  
    if ((e = p.next) == null) {  
        p.next = newNode(hash, key, value, null);  
        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st  
            treeifyBin(tab, hash);  
        break;  
    }   
}

// 链表上的元素个数到达 8 个时，转换为 rbNode
final void treeifyBin(Node<K,V>[] tab, int hash) {  
    int n, index; Node<K,V> e;
    // 数组大小未达到 64 时，不转树，扩容  
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)  
        resize();  
    else if ((e = tab[index = (n - 1) & hash]) != null) { 
	    // 存在节点 
        TreeNode<K,V> hd = null, tl = null;  
        do {  
		    // 转换为 treeNode
            TreeNode<K,V> p = replacementTreeNode(e, null);  
            if (tl == null)
                hd = p;  
            else {  
                p.prev = tl;  
                tl.next = p;  
            }  
            tl = p;  
        } while ((e = e.next) != null);  
        if ((tab[index] = hd) != null)  
	        // 树化每个 bin 后，树化 tab
            hd.treeify(tab);  
    }  
}

TreeNode<K,V> replacementTreeNode(Node<K,V> p, Node<K,V> next) {  
    return new TreeNode<>(p.hash, p.key, p.value, next);  
}
```
#### 2.4.4.2 构建 rbTree
```java
final void treeify(Node<K,V>[] tab) {  
    TreeNode<K,V> root = null;  
    // this = head
    for (TreeNode<K,V> x = this, next; x != null; x = next) {  
        next = (TreeNode<K,V>)x.next;  
        x.left = x.right = null;  
        if (root == null) {  
            x.parent = null;  
            // 根节点为黑色
            x.red = false;  
            root = x;  
        }  
        else {  
            K k = x.key;  
            int h = x.hash;  
            Class<?> kc = null; 
            // 和 root 比较 hash 
            for (TreeNode<K,V> p = root;;) {  
                int dir, ph;  
                K pk = p.key;  
                if ((ph = p.hash) > h)  
                    dir = -1;  
                else if (ph < h)  
                    dir = 1; 
                // 不能比较的或比较结果相同的通过 native 方法比较大小 
                else if ((kc == null &&  
                          (kc = comparableClassFor(k)) == null) ||  
                         (dir = compareComparables(kc, k, pk)) == 0)  
                    dir = tieBreakOrder(k, pk);  
                TreeNode<K,V> xp = p; 
                // 比 root 小的作为左节点，否则是右节点 
                if ((p = (dir <= 0) ? p.left : p.right) == null) {  
                    x.parent = xp;  
                    if (dir <= 0)  
                        xp.left = x;  
                    else  
                        xp.right = x;  
                    // 重平衡
                    root = balanceInsertion(root, x);  
                    break;  
                }  
            }  
        }  
    }  
    // 确保 root 节点在 bin 的头部
    moveRootToFront(tab, root);  
}
```
#### 2.4.4.3 插入重平衡
```java
/*
 * @param x: hd
 * @param root
 * @return root
 */
static <K,V> TreeNode<K,V> balanceInsertion(TreeNode<K,V> root,  
                                            TreeNode<K,V> x) {  
    // 插入红色节点
    x.red = true;  
    /*
            xpp
           /    \
       xp(xppl)  xppr
         /
        x
    */
    for (TreeNode<K,V> xp, xpp, xppl, xppr;;) {  
	    // 只有一个节点，转黑作为 root
        if ((xp = x.parent) == null) {  
            x.red = false;  
            return x;  
        }  
        // 父节点是黑色不用处理
        else if (!xp.red || (xpp = xp.parent) == null)  
            return root;  
        if (xp == (xppl = xpp.left)) {  
            if ((xppr = xpp.right) != null && xppr.red) {  
                xppr.red = false;  
                xp.red = false;  
                xpp.red = true;  
                x = xpp;  
            }  
            else {  
                if (x == xp.right) {  
                    root = rotateLeft(root, x = xp);  
                    xpp = (xp = x.parent) == null ? null : xp.parent;  
                }  
                if (xp != null) {  
                    xp.red = false;  
                    if (xpp != null) {  
                        xpp.red = true;  
                        root = rotateRight(root, xpp);  
                    }  
                }  
            }  
        }  
        else {  
            if (xppl != null && xppl.red) {  
                xppl.red = false;  
                xp.red = false;  
                xpp.red = true;  
                x = xpp;  
            }  
            else {  
                if (x == xp.left) {  
                    root = rotateRight(root, x = xp);  
                    xpp = (xp = x.parent) == null ? null : xp.parent;  
                }  
                if (xp != null) {  
                    xp.red = false;  
                    if (xpp != null) {  
                        xpp.red = true;  
                        root = rotateLeft(root, xpp);  
                    }  
                }  
            }  
        }  
    }  
}
```
#### 2.4.4.4 扩容
```java
if (oldTab != null) {  
    for (int j = 0; j < oldCap; ++j) {  
        Node<K,V> e;  
        if ((e = oldTab[j]) != null) {  
            oldTab[j] = null;  
            else if (e instanceof TreeNode)  
                ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
	    }
	}
}

// 分为两个链表，达到反树化阈值时变为链表
final void split(HashMap<K,V> map, Node<K,V>[] tab, int index, int bit) {  
    TreeNode<K,V> b = this;  
    // Relink into lo and hi lists, preserving order  
    TreeNode<K,V> loHead = null, loTail = null;  
    TreeNode<K,V> hiHead = null, hiTail = null;  
    int lc = 0, hc = 0;  
    for (TreeNode<K,V> e = b, next; e != null; e = next) {  
        next = (TreeNode<K,V>)e.next;  
        e.next = null;  
        if ((e.hash & bit) == 0) {  
            if ((e.prev = loTail) == null)  
                loHead = e;  
            else  
                loTail.next = e;  
            loTail = e;  
            ++lc;  
        }  
        else {  
            if ((e.prev = hiTail) == null)  
                hiHead = e;  
            else  
                hiTail.next = e;  
            hiTail = e;  
            ++hc;  
        }  
    }  
  
    if (loHead != null) {  
        if (lc <= UNTREEIFY_THRESHOLD)  
            tab[index] = loHead.untreeify(map);  
        else {  
            tab[index] = loHead;  
            if (hiHead != null) // (else is already treeified)  
		        // 再树化一次，整理节点顺序
                loHead.treeify(tab);  
        }  
    }  
    if (hiHead != null) {  
        if (hc <= UNTREEIFY_THRESHOLD)  
            tab[index + bit] = hiHead.untreeify(map);  
        else {  
            tab[index + bit] = hiHead;  
            if (loHead != null)  
                hiHead.treeify(tab);  
        }  
    }  
}
```
- `HashMap` 在 `resize()` 时，如果多个线程进行 `put()` 时，可能出现环形列表的情况，容易出现环形链表的情况，因此线程不安全
# 3 ConcurrentHashMap
## 3.1 fields
```java
private static final int MAXIMUM_CAPACITY = 1 << 30;

static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

private static final int DEFAULT_CONCURRENCY_LEVEL = 16;

// 与 HashMap 相同的属性
static final int TREEIFY_THRESHOLD = 8;
static final int UNTREEIFY_THRESHOLD = 6;
static final int MIN_TREEIFY_CAPACITY = 64;
private static final int DEFAULT_CAPACITY = 16;
private static final float LOAD_FACTOR = 0.75f;

// 存储数据的 Node 数组
transient volatile Node<K,V>[] table;
// 扩容时使用
private transient volatile Node<K,V>[] nextTable;
// 用于初始化和扩容
private transient volatile int sizeCtl;
// 提供一些 CAS 方法
// 通过 Unsafe 实现 CAS 汇编指令解决线程安全问题
private static final sun.misc.Unsafe U;
// 用于记录元素个数
// 如果用一般的属性 size 记录元数个数，在保证线程安全时需要通过加锁或自旋
private transient volatile CounterCell[] counterCells;
```
## 3.2 InnerClass
### 3.2.1 TreeBin
```java
// 在 Node 和 TreeNode 的基础上增加了 TreeBin
// TreeBin 表示 TreeNode 的列表以及 root
static final class TreeBin<K,V> extends Node<K,V> {  
    TreeNode<K,V> root;  
    volatile TreeNode<K,V> first;  
    volatile Thread waiter;  
    volatile int lockState;  
    // values for lockState  
    static final int WRITER = 1; // set while holding write lock  
    static final int WAITER = 2; // set when waiting for write lock  
    static final int READER = 4; // increment value for setting read lock
}
```
### 3.2.2 ForwardingNode
```java
// 扩容时使用的特殊节点，k, v, hash 均为 null, 用一个 nextTable 指针引用新的 table 数组
static final class ForwardingNode<K,V> extends Node<K,V> {  
    final Node<K,V>[] nextTable;  
    ForwardingNode(Node<K,V>[] tab) {  
        super(MOVED, null, null, null);  
        this.nextTable = tab;  
    }
}
```
## 3.3 CAS 操作
```java
// 从 tab 中通过索引获取 Node 
static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {  
    return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);  
}

// 通过 cas 操作更改索引位置的元素
static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,  
                                    Node<K,V> c, Node<K,V> v) {  
    return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);  
}

static final <K,V> void setTabAt(Node<K,V>[] tab, int i, Node<K,V> v) {  
    U.putObjectVolatile(tab, ((long)i << ASHIFT) + ABASE, v);  
}
```
## 3.4 constructor
```java
public ConcurrentHashMap() {}

public ConcurrentHashMap(int initialCapacity) {  
    int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?  
               MAXIMUM_CAPACITY :  
               tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));  
    this.sizeCtl = cap;  
}

public ConcurrentHashMap(int initialCapacity,  
                         float loadFactor, int concurrencyLevel) {   
    if (initialCapacity < concurrencyLevel)   // Use at least as many bins  
        initialCapacity = concurrencyLevel;   // as estimated threads  
    long size = (long)(1.0 + (long)initialCapacity / loadFactor);  
    int cap = (size >= (long)MAXIMUM_CAPACITY) ?  
        MAXIMUM_CAPACITY : tableSizeFor((int)size);  
    this.sizeCtl = cap;  
}
```
## 3.5 插入元素
### 3.5.1 putVal()
```java
final V putVal(K key, V value, boolean onlyIfAbsent) {  
    if (key == null || value == null) throw new NullPointerException();  
    // 计算 hash 值
    int hash = spread(key.hashCode());  
    int binCount = 0;  
    for (Node<K,V>[] tab = table;;) {  
        Node<K,V> f; int n, i, fh;  
        if (tab == null || (n = tab.length) == 0)  
		    // 第一次插入初始化 tab
            tab = initTable();  
        // 没有元素
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {  
            if (casTabAt(tab, i, null,  
	                new Node<K,V>(hash, key, value, null)))  
                break;                   // no lock when adding to empty bin  
        }  
        // MOVED = -1
        // 如果计算出 hash 为 - 1，说明该节点是 forwardingNode，有扩容操作正在进行
        else if ((fh = f.hash) == MOVED)  
            tab = helpTransfer(tab, f);  
        else {  
            V oldVal = null;  
            synchronized (f) {  
                if (tabAt(tab, i) == f) {  
                    if (fh >= 0) {  
                        binCount = 1;  
                        for (Node<K,V> e = f;; ++binCount) {  
                            K ek;  
                            // 替换 value
                            if (e.hash == hash &&  
                                ((ek = e.key) == key ||  
                                 (ek != null && key.equals(ek)))) {  
                                oldVal = e.val;  
                                if (!onlyIfAbsent)  
                                    e.val = value;  
                                break;  
                            }  
                            // 插入链表
                            Node<K,V> pred = e;  
                            if ((e = e.next) == null) {  
                                pred.next = new Node<K,V>(hash, key,  
                                                          value, null);  
                                break;  
                            }  
                        }  
                    }  
                    // 插入到红黑树
                    else if (f instanceof TreeBin) {  
                        Node<K,V> p;  
                        binCount = 2;  
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,  
                             value)) != null) {  
                            oldVal = p.val;  
                            if (!onlyIfAbsent)  
                                p.val = value;  
                        }  
                    }  
                }  
            }  
            if (binCount != 0) {  
                if (binCount >= TREEIFY_THRESHOLD)  
                    treeifyBin(tab, i);  
                if (oldVal != null)  
                    return oldVal;  
                break;  
            }  
        }  
    }  
    // 检查扩容
    addCount(1L, binCount);  
    return null;  
}
```
### 3.5.2 initTable()
```java
private final Node<K,V>[] initTable() {  
    Node<K,V>[] tab; int sc;  
    while ((tab = table) == null || tab.length == 0) {  
	    // sizeCtl = -1 时表示有线程正在初始化，让出时间片
        if ((sc = sizeCtl) < 0)  
            Thread.yield(); // lost initialization race; just spin  
        // 将 sizeCtl 设置为 -1
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {  
            try {  
                if ((tab = table) == null || tab.length == 0) {
	                // 初始化数组大小为 16  
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;  
                    @SuppressWarnings("unchecked")  
                    // 创建 Node 数组
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];  
                    table = tab = nt;  
                    sc = n - (n >>> 2);  
                }  
            } finally {  
	            // 更新 sizeCtl
                sizeCtl = sc;  
            }  
            break;  
        }  
    }  
    return tab;  
}
```
### 3.5.3 helpTransfer()
```java
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {  
    Node<K,V>[] nextTab; int sc;  
    if (tab != null && (f instanceof ForwardingNode) &&  
        (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {  
        int rs = resizeStamp(tab.length) << RESIZE_STAMP_SHIFT;
        // 三个条件均表示正在扩容中
        while (nextTab == nextTable && table == tab &&  
               (sc = sizeCtl) < 0) {  
            // 不需要协助
            if (sc == rs + MAX_RESIZERS || sc == rs + 1 ||  
                transferIndex <= 0)  
                break;  
            if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {  
                transfer(tab, nextTab);  
                break;  
            }  
        }  
        return nextTab;  
    }  
    return table;  
}
```
## 3.6 扩容
```java
private final void addCount(long x, int check) {  
    CounterCell[] as; long b, s;  
    if (check >= 0) {  
        Node<K,V>[] tab, nt; int n, sc;  
        // 集合元素个数达到 sizeCtl，长度没有达到 MAXIMUM_CAPACITY
        while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&  
               (n = tab.length) < MAXIMUM_CAPACITY) {  
            int rs = resizeStamp(n) << RESIZE_STAMP_SHIFT; 
            // 正在扩容中 
            if (sc < 0) {  
	            // 判断扩容是否结束或扩容并发线程数是否到最大值
                if (sc == rs + MAX_RESIZERS || sc == rs + 1 ||  
                    (nt = nextTable) == null || transferIndex <= 0)  
                    break;  
                // 扩容还未结束，并且允许扩容线程加入
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))  
                    transfer(tab, nt);  
            }  
            // 扩容
            else if (U.compareAndSwapInt(this, SIZECTL, sc, rs + 2))  
                transfer(tab, null);  
            s = sumCount();  
        }  
    }  
}

private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {  
    int n = tab.length, stride; 
    // 判断 CPU 的个数 
    // 计算结果小于 16 时，一条线程处理 16 个桶
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)  
        stride = MIN_TRANSFER_STRIDE; // subdivide range  
    if (nextTab == null) {            // initiating  
        try {  
            // 初始化一个新 table nt, 长度为之前的 2 倍
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];  
            nextTab = nt;  
        } 
        nextTable = nextTab;  
        // 将 transferIndex 指向最右边的 bin
        transferIndex = n;  
    }  
    int nextn = nextTab.length;  
    // 新建一个 ForwardingNode，1: 表示正在扩容, 2: 遇到查询操作时会转发
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);  
    // 表示是否处理完当前的桶
    boolean advance = true;  
    // 表示扩容是否结束
    boolean finishing = false; // to ensure sweep before committing nextTab  
    for (int i = 0, bound = 0;;) {  
        Node<K,V> f; int fh;  
        while (advance) {  
            int nextIndex, nextBound;  
            // 每处理完一个 bin 后 --i
            if (--i >= bound || finishing)  
                advance = false;
            // transferIndex <=0 说明数组的 bin 已经被线程分配完毕
            else if ((nextIndex = transferIndex) <= 0) {  
                i = -1;  
                advance = false;  
            }  
            // 首次进入 for 循环时进入，设置 bound 和 i 的值
            else if (U.compareAndSwapInt  
                     (this, TRANSFERINDEX, nextIndex,  
                      nextBound = (nextIndex > stride ?  
                                   nextIndex - stride : 0))) {  
                bound = nextBound;  
                i = nextIndex - 1;  
                advance = false;  
            }  
        }  
        // 扩容结束，将 nextTable 设置为 null
        if (i < 0 || i >= n || i + n >= nextn) {  
            int sc;  
            if (finishing) {  
                nextTable = null;  
                table = nextTab;  
                sizeCtl = (n << 1) - (n >>> 1);  
                return;  
            }  
            // 每当一条线程扩容更新一次 sizeCtl，-1
            if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) { 
	            // 最后一条扩容线程 
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)  
                    return;  
                finishing = advance = true;  
                i = n; // recheck before commit  
            }  
        }  
        // 未结束时，遇到空位设置一个 ForwardingNode
        else if ((f = tabAt(tab, i)) == null)  
            advance = casTabAt(tab, i, null, fwd);
        // 如果遇到 ForwardingNode，说明已经处理过
        else if ((fh = f.hash) == MOVED)  
            advance = true; // already processed  
        else {  
            synchronized (f) {  
                if (tabAt(tab, i) == f) {  
                    Node<K,V> ln, hn;  
                    if (fh >= 0) {  
                        int runBit = fh & n;  
                        Node<K,V> lastRun = f;  
                        for (Node<K,V> p = f.next; p != null; p = p.next) { 
                            int b = p.hash & n;  
                            if (b != runBit) {  
                                runBit = b;  
                                lastRun = p;  
                            }  
                        }  
                        if (runBit == 0) {  
                            ln = lastRun;  
                            hn = null;  
                        }  
                        else {  
                            hn = lastRun;  
                            ln = null;  
                        }  
                        for (Node<K,V> p = f; p != lastRun; p = p.next) {  
                            int ph = p.hash; K pk = p.key; V pv = p.val;  
                            if ((ph & n) == 0)  
                                ln = new Node<K,V>(ph, pk, pv, ln);  
                            else  
                                hn = new Node<K,V>(ph, pk, pv, hn);  
                        }  
                        setTabAt(nextTab, i, ln);  
                        setTabAt(nextTab, i + n, hn);  
                        setTabAt(tab, i, fwd);  
                        advance = true;  
                    }  
                    else if (f instanceof TreeBin) {  
                        TreeBin<K,V> t = (TreeBin<K,V>)f;  
                        TreeNode<K,V> lo = null, loTail = null;  
                        TreeNode<K,V> hi = null, hiTail = null;  
                        int lc = 0, hc = 0;  
                        for (Node<K,V> e = t.first; e != null; e = e.next) {  
                            int h = e.hash;  
                            TreeNode<K,V> p = new TreeNode<K,V>  
                                (h, e.key, e.val, null, null);  
                            if ((h & n) == 0) {  
                                if ((p.prev = loTail) == null)  
                                    lo = p;  
                                else  
                                    loTail.next = p;  
                                loTail = p;  
                                ++lc;  
                            }  
                            else {  
                                if ((p.prev = hiTail) == null)  
                                    hi = p;  
                                else  
                                    hiTail.next = p;  
                                hiTail = p;  
                                ++hc;  
                            }  
                        }  
                        ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :  
                            (hc != 0) ? new TreeBin<K,V>(lo) : t;  
                        hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :  
                            (lc != 0) ? new TreeBin<K,V>(hi) : t;  
                        setTabAt(nextTab, i, ln);  
                        setTabAt(nextTab, i + n, hn);  
                        setTabAt(tab, i, fwd);  
                        advance = true;  
                    }  
                }  
            }  
        }  
    }  
}
```
