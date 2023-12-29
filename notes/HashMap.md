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
## 2.2 