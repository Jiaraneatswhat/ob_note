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
## 4.2 首次 resize()
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
## 4.3 非首次 resize()

![[hashmap_resize.svg]]

- table 中的 bin 个数达到 threshold 时，进行第二次扩容
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
e.hash & oldCap == 0 说明前后索引位置一样
e.hash & oldCap != 0 说明需要移动 oldCap 位置
```
## 4.4 rbTree 扩容
### 4.4.1 生成 rbNode 
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
### 4.4.2 构建 rbTree
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
### 4.4.3 插入重平衡
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


