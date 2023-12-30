# 1.重要属性
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
# 2.数据结构
## 2.1 Node
- Node 相当于一个桶，Node 数组相当于一个 Hashtable
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
## 2.2 TreeNode
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

# 3.构造器
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
# 4.插入元素
## 4.1 putVal()
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
## 4.2 第一次 resize()
- 第一次扩容，生成一个容量为 16，阈值为 12 的 HashMap
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
## 4.3 第二次 resize()

![[hashmap_resize.svg]]

- table 中的 bin 个数达到 threshold 时，进行第二次扩容
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
                        if ((e.hash & oldCap) == 0) {  
                            if (loTail == null)  
                                loHead = e;  
                            else  
                                loTail.next = e;  
                            loTail = e;  
                        }  
                        else {  
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
- hash 计算新索引位置
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
```
- 