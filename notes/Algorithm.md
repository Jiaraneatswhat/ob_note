# 1 LinkedList
- 随机访问的复杂度为 $O(n)$
## 1.1 unidirectional
```java
public class SingleLinkedList {
	Node head;

	class Node {
		int value;
		Node next;
	}
}
```
### 1.1.1 addFirst()
```java
// addFirst
public void addFirst(int value) {
	/*
	if (head == null) {
		head = new Node(value, null);
	} else {
		head = new Node(value, head);
	} */
	head = new Node(value, head);
}
```
### 1.1.2 traversal
```java
// traversal
public void traversal() {
	Node p = head;
	while (p != null) {
		// print(p.value);
		p = p.next;
	}
}

public void traversal() {
	Node p = head;
	for (Node p = head; p != null; p = p.next) // print
}

// 实现 Iterable 接口后，通过迭代器迭代
public Iterator<Integer> iterator() {  
return new Iterator<Integer>() {  
	Node p = head;  

	@Override  
	public boolean hasNext() {  
		return p != null;  
	}  

	@Override  
	public Integer next() { // 返回当前值，指向下一个元素  
		int v = p.value;  
		p = p.next;  
		return v;  
	};  
}
```
### 1.1.3 addLast()
```java
public Node findLast() {
	Node p;
	for (p = head; p.next != null; p = p.next) return p;
}

public void addLast(int value) {
	if (head == null) {
		addFirst(value);
	} else {
		Node last = findLast();
		last.next = new Node(value, null);
	}
}
```
### 1.1.4 get()
```java
// 获取指定索引节点的值
public Node findNode(int value) {
	int i = 0;
	// 遍历一次索引加 1
	for (Node p = head; p != null; p = p.next; i++) {
		if (i == index) {
			return p;
		}
	}
	return null;
}
public int get(int value) {
	Node target = findNode(value);
	if (target == null) {
		// Exception
	}
	return target.value;
}
```
### 1.1.5 insert()
```java
public void insert(int index, int value) {
	if (index == 0) {
		addFirst(value);
		return;
	}
	// 找到上一个元素
	Node prev = findNode(index - 1);
	if (prev == null) // Exception
	prev.next = new Node(value, prev.next);
}
```
### 1.1.6 remove()
```java
public void removeFirst() {
	if (head != null) head = head.next;
}

public void remove(int index) {
	if (index == 0) {
		removeFirst();
		return;
	}
	Node prev = find(index - 1);
	if (prev != null) prev.next = prev.next.next;
}
```
### 1.1.7 with sentinel
```java
public class SingleLinkedListWithSentinel {
	Node head = null;
	// 可以省去空链表判断
	Node sentinel = new Node(%anyVal, null);
	head = sentinel;

	// addLast()
	public void addLast(int value) {
		Node last = findLast()
		last.next = new Node(value, null);
	}	

	// traversal, 遍历从哨兵的 next 开始
	public void traversal() {
		for (Node p = head.next; p != null; p = p.next)
	}

	// insert
	public void insert(int index, int value) {
		// findNode index 从 -1 开始匹配
		Node prev = findNode(index - 1);
		prev.next = new Node(value, prev.next);
	}

	// remove
	public void remove(int index) {
		Node prev = findNode(index - 1);
		prev.next = prev.next.next;
	}
}
```
## 1.2 bidirectional with sentinel
```java
public class BiLinkedList {
	// 双哨兵
	Node head;
	Node tail;
	class Node {
		Node prev;
		int value;
		Node next;
	}

	public BiLinkedList() {
		head = new Node(null, &anyVal, null);
		tail = new Node(null, &anyVal, null);
		head.next = tail;
		tail.prev = head;
	}
}
```
### 1.2.1 insert()
```java
public Node findNode(int index) {
	int i = -1;
	for (Node p = head; p != tail; p = p.next, i++) {
		if (i == index) return p;
	}
	return null;
}

public insert(int index, int value) {
	// prev 空值判断
	Node prev = findNode(index - 1);
	Node elem = new Node(prev, value, prev.next);
	prev.next = elem;
	prev.next.prev = elem;
}

public void addLast(int value) {
		Node last = tail.prev;
		Node added = new Node(last, value, tail);
		last.next = added;
		tail.prev = added;
	}
```
### 1.2.2 remove()
```java
public void remove(int index) {
	Node prev = findNode(index - 1);
	// 违法索引, prev == null, 抛异常
	// prev 为最后一个元素时
	if (prev.next == tail) // 不合法
	prev.next = prev.next.next;
	prev.next.prev = prev;
}

public void removeLast() {
	Node last = tail.prev;
	// last 是 head, 即链表为空时, 抛异常
	Node prev = last.prev;
	prev.next = tail;
	tail.prev = prev;
}
```
## 1.3 bidirectional circular with sentinel
```java
public class CircularLinkedListWithSentinel {
	
	class Node ...

	private Node s = new Node(null, %anyVal, null) // sentinel

	public CircularLinkedListWithSentinel{
		s.prev = s;
		s.next = s;
	}
}
```
### 1.3.1 insert()
```java
public addFirst(int value) {
	Node first = s.next;
	Node added = new Node(s, value, first);
	s.next = added;
	first.prev = added;
}

public addLast(int value) {
	Node prev = s.prev;
	Node added = new Node(prev, value, s);
	s.prev = added;
	prev.next = added;
}
```
### 1.3.2 traversal
```java
public Iterator<Integer> iterator(){
	return new Iterator<Integer>() {
		Node p = s.next;
		@Override
		public boolean hasNext() {
			return p != s;
		}

		@Override
		public Integer next() {...}
	}
}
```
### 1.3.3 remove() 
```java
public void removeFirst() {
	Node removed = s.next;
	if (removed == s) // Exception
	s.next = removed.next;
	s.next.prev = s;
}

public removeLast() {
	Node removed = s.prev;
	s.prev = removed.prev;
	s.prev.next = s;
}

public Node findByVal)(int value) {
	Node p = s.next;
	while(p != s) {
		if (p.value == value) return p;
		p = p.next;
	}
	return null;
}

public void removeByVal(int value) {
	Node removed = findByValue(value);
	if (removed != null) {
		removed.prev.next = removed.next;
		removed.next.prev = removed.prev;
	}
	
}
```
## 1.4 LeetCode
### Q206(S) - 反转单向链表
- 给定链表的头结点 `head`，反转链表并返回
```
input:
head = [1, 2, 3, 4, 5]

output:
[5, 4, 3, 2, 1] 循环打印

input = []
output: []
```
#### solution1
- 从旧链表依次拿到每个节点，创建新节点添加到链表头部
```java
public Node reverseList(Node o1) {
	Node n1 = null;
	Node p = o1;
	while(p != null) {
		n1 = new Node(p.val, n1); // 头插
		p = p.next;
	}
	return n1;
}
```
#### solution2
- 从旧链表头部移除节点添加到新链表头部
```java
public void addFirst(Node added) {
	added.next = head;
	head = added;
}

public Node removeFirst(Node removed) {
	if (head != null) {
		head = head.next;
	}
	return first;
}

public Node reverseList(Node head) {
	List list1 = new List(head);
	List list2 = new List(null);
	while(true) {
		Node removed = list1.removeFirst();
		if (removed == null) break;
		list2.addFirst(removed);
	}
}
```
####  solution3
- 递归反转
```java
public Node reverseList(Node p) {
	// p == null 直接返回
	if (p == null || p.next == null) {
		return p; // 最后一个节点
	}
	Node last = reverseList(p.next);
	p.next.next = p; // 反向指
	p.next = null; // 正向指置空 a -> b => b -> a -> null
	return last;
}
```
####  solution4
- 双指针
- 从链表每次拿到第二个节点，将其从链表断开，插入头部直到 `null`
```java
/**
 *  n1 o1   o2
 *    1  -> 2 -> 3 -> 4 -> 5 -> null
 *  n1 o1                        o2
 *    1  -> 3 -> 4 -> 5 -> null, 2
 *   o2   n1 o1
 *    2 ->  1  -> 3 -> 4 -> 5 -> null
 *  n1 o2  o1
 *    2  -> 1 -> 3 -> 4 -> 5 -> null
 *   n1  o1   o2
 *   2 -> 1 -> 3 -> 4 -> 5 -> null
 */
public Node reverseList(Node o1) {
	Node o2 = o1.next;
	Node n1 = o1;
	while(o2 != null) {
		// 断开 o2
		o1.next = o2.next;
		// o2 插入头部
		o2.next = n1;
		n1 = o2;
		// o2 指向 o1 的下一个节点
		o2 = o1.next;
	}
	return n1;
}
```
####  solution5
- 链表 2 的头向链表 1 的头移动元素 
 ```java
 public Node reverseList(Node o1) {
	Node n1 = null;
	while(o1 != null) {
		Node o2 = o1.next;
		o1.next = n1;
		n1 = o1;
		o1 = o2;	
	}
	return n1;
}
```
### Q203 - 根据值删除节点
- 存在多个重复的值时全部删除
#### solution1
- p1 指向哨兵，p2 指向下一个元素
- p2 值相同，删除后，p2 向后移，p1 不变
- 值不相同，两者均后移
```java
public Node removeAllElems(Node head, int val) {
	Node s = new Node(-1, head);
	Node p1 = s;
	Node p2;
	while((p2 = p1.next) != null) {
		if (p2.val == val) {
			p1.next = p2.next;
		} else {
			p1 = p1.next;
		}
	} 
	return s.next;
}
```
#### solution2
- 递归
- `curr = val` 时，跳过当前值，返回 `next` 的递归结果
- `curr != val` 时，返回当前值，`next` 进行递归后更新 `next`
```java
public removeAllElems(Node p, int val) {
	if (p == null) {
		return null;
	}
	if (p.val == val) {
		return removeAllElems(p.next, val);
	} else {
		p.next = removeAllElems(p.next, val);
		return p;
	}

}
```
### Q19(M) - 删除倒数第 N 个节点
```
input:
head = [1, 2, 3, 4, 5], n = 2

output: [1, 2, 3, 5]

```
#### solution1
- 递归：null 返回 0，上一个节点返回 1, ...
```java
private int recursion(Node p, int n) {
	if (p == null) return 0;
	int nth = recursion(p.next, n); // 下一个节点的倒数位置
	if (nth == n) {
		p = p.next.next;
	}
	return nth + 1; // +1 得到当前节点的倒数位置
}

private Node remove(Node p, int n) {
	Node s = new Node(-1, p); // 哨兵的下一个节点是倒数最后一个
	recursion(s, n);
	return p;
}
```
#### solution2
```java
/**
 *  n=2
 *  p1 p2
 *    s -> 1  -> 2 -> 3 -> 4 -> 5 -> null
 *   p1              p2  
 *    s -> 1 -> 2 -> 3 -> 4 -> 5 -> null
 *                   p1              p2
 *    s -> 1 -> 2 -> 3 -> 4 -> 5 -> null
 *   p2 到达 null 时，p1 的位置为待删除节点的上一个节点
 */

public removeNthFromEnd(Node head, int n) {
	Node s = new Node(-1, head);
	Node p1 = s;
	Node p2 = s;
	for (int i = 0; i < n + 1; i++) p2 = p2.next;
	while(p2 != null) {
		p1 = p1.next;
		p2 = p2.next;
	}
	p1.next = p1.next.next;
	return s.next;
}
```
### Q83 - 有序链表去重(重复元素保留一个)
#### solution1
- 双指针同时后移，比较值
```java
public Node removeDublicate(Node head) {
	// 节点数 > 2
	if (head == null || head.next == null) return head;
	Node p1 = head;
	Node p2;
	while((p2 = p1.next) != null) {
		if (p1.val == p2.val) {
			p1.next = p2.next;
		} else p1 = p1.next;
	}
	return head;
}
```
#### solution2
- 递归
- `curr = next`, 返回 next 的递归结果
- `curr != next`, 更新 next 为 next 的递归结果
```java
public Node removeDublicate(Node head) {
	if (p == null || p.next == null) return p;
	if (p.val = p.next.val) {
		return removeDublicate(p.next);
	} else {
		p.next = removeDublicate(p.next);
		return p;
	}
}
```
### Q82 - 有序链表去重(重复元素不保留)
#### solution1
- 递归
- `curr = next`, 一直找到不重复的，返回其递归结果
- `curr != next`, 更新 next 为 next 的递归结果
```java
public Node removeDublicate(Node head) {
	if (p == null || p.next == null) return p;
	if (p.val = p.next.val) {
		Node nextTo = p.next.next;
		while (nextTo != null && nextTo.val = p.val) nextTo = nextTo.next;
		return removeDublicate(nextTo)
	} else {
		p.next = removeDublicate(p.next);
		return p;
	}
}
```
#### solution2
```java
/**
 *  n=2
 *  p1   p2   p3
 *  s -> 1  -> 1 -> 1 -> 2 -> 3 -> null
 *  p1   p2             p3  
 *  s -> 1 -> 1 -> 1 -> 2 -> 3 -> null
 *  p1   p3
 *  s -> 2 -> 3 -> null
 *  p1   p2   p3
 *  s -> 2 -> 3 -> null 
 */

public Node removeDublicate(Node p) {
	if (p == null || p.next == null) return p;
	Node s = new Node(-1, p);
	Node p1 = s;
	Node p2, p3;
	while((p2 = p1.next) != null && (p3 = p2.next) != null) {
		if (p2.val == p3.val) {
			while((p3 = p3.next) != null && p3.val == p2.val) 
			p1.next = p3;
		} else {
			p1 = p1.next;
		}
	}
	return s.next;
}
```
### Q21 - 合并两个有序链表
```
input:
l1 = [1, 2, 4], l2 =[1, 3, 4]

output: [1, 1, 2, 3, 4, 4]

input:
l1 = [], l2 = []

output: []
```
#### solution1
```java
/**
 *  p1
 *  1 -> 3-> 8 -> 9 -> null
 *  p2                 
 *  2 -> 4 -> null
 *       p
 *  s -> 1
 *       p1
 *  1 -> 3-> 8 -> 9 -> null
 *  p1 和 p2 中小的链在 p 后, 同时和 p 向后移
 *  有一个为 null 时，比较结束，将剩下的直接链在 p 后
 */

public void mergeList(Node p1, Node p1) {
	Node s = new Node(-1, null);
	Node p = s;
	while (p1 != null && p2 != null) {
		if (p1.val < p2.val) {
			p.next = p1;
			p1 = p1.next;
		} else {
			p.next = p2;
			p2 = p2.next;
		}
		p = p.next;
	}
	if (p1 != null) p.next = p1;
	if (p2 != null) p.next = p2;
	return s.next;
}
```
#### solution2
- 递归
```java
public void mergeList(Node p1, Node p2) {
	if (p1 == null) return p2;
	if (p2 == null) return p1;
	if (p1.val < p2.val) {
		p1.next = mergeList(p1.next, p2);
		return p1;
	} else {
		p2.next = mergeList(p1, p2.next);
		return p2;
	}

}
```
### Q23(H) - 合并 K 个升序链表
```
input:
lists = [[1, 4, 5], [1, 3, 4], [2, 6]]

output:
[1, 1, 2, 3, 4, 4, 5, 6]
```
#### solution
```java
// 分治
public Node mergeKLists(Node[] lists) {
	return split(lists, 0, lists.length - 1);
}

// 返回合并后的链表, i, j 是左右边界
public Node split(Node[] lists, int i, int j) {
	// 只有一个链表
	if (i == j) return lists[i];
	int m = (i + j) >>> 1;
	Node left = split(lists, i, m);
	Node right = split(lists, m + 1, j);
	return mergeList(left, right);
}
```
### Q58 - 查找链表的中间节点
- 快慢指针法: 一个走一步，一个走两步
- 快指针走到尾，慢指针位置是中间节点
#### solution
```java
public findMidNode(Node head) {
	Node p1 = head;
	Node p2 = head;
	while (p2 != null && p2.next != null) { // 对应奇数 偶数情况
		p1 = p1.next;
		p2 = p2.next;
		p2 = p2.next;
	}
	return p1;
	
}
```
### Q234 - 判断回文
```
input:
[1, 2]

output: false
```
#### solution1
- 反转中间点后半部分链表对比
```java
public boolean isPalindrome(Node head) {
	Node mid = findMidNode(head);
	Node newHead = reverseNode(mid);
	while (newHead != null) {
		if (newHead.val != head.val) return false;
		newHead = newHead.next;
		head = head.next;
	}
	return true;
}

public Node findMidNode(Node head) {
	Node p1 = head;
	Node p2 = head;
	while (p2 != null && p2.next != null) {
		p1 = p1.next;
		p2 = p2.next.next;
	}
	return p1;
}

public Node reverseNode (Node o1) {
	Node n1 = null;
	while (o1 != null) {
		Node o2 = n1.next;
		o1.next = n1;
		n1 = o1;
		o1 = o2;
	}
	return n1;
}
```
#### solution2 
- 找中间点同时反转前半链表
```java
public boolean isPalindrome(Node head) {
	Node p1 = head;
	Node p2 = head;
	Node n1 = null;
	Node o1 = head;
	while (p2 != null && p2.next != null) {
		p1 = p1.next;
		p2 = p2.next.next;

		o1.next = n1;
		n1 = o1;
		o1 = p1;
	}

	if (p2 != null) { // 奇数 反转后慢指针需要后移
		p1 = p1.next;
	}
	while (n1 != null) {
		if (n1.val != p1.val) return false;
		n1 = n1.next;
		p1 = p1.next;
	}
	return true;
}
```
# 2 Array
## 2.1 动态数组
```java
public class DynamicArray {  
    int size = 0;  
    int capacity = 8;  
    int[] array = new int[capacity];  
}
```
### 2.1.1 add
```java
// addLast
 void addLast(int e) {  
    // array[size++] = e;  
    add(size, element);
 }

// 按索引插入
void add(int index, int e) {  
	if (index >= 0 && index < size) {
	    // 将 index 后的元素后移  
		System.arraycopy(array, index, array, index + 1, size - index);  
	}
	array[index] = e;  
	size++; 
}
```
### 2.1.2 traversal
```java
void traversal(Consumer<Integer> consumer) {  
    for (int i = 0; i < size; i++) {  
        consumer.accept(array[i]);  
    }  
}

// 实现 Iterable 接口
@Override  
public Iterator<Integer> iterator() {  
    return new Iterator<>() {  
        int i = 0;  
        @Override  
        public boolean hasNext() {  
            return i < size;  
        }  
  
        @Override  
        public Integer next() { // 返回当前元素，移动到下一个元素  
            return array[i++];  
        }  
    };  
}
```
### 2.1.3 remove
```java
int remove(int index) {  
    int removed = array[index]; 
	if (index < size - 1) { // 要删的不是最后一个元素时才需要 copy 数组
		// 后面向前移动  
	    System.arraycopy(array, index + 1, array, index, size - index - 1);
	} 
    size--;
    return removed;  
}
```
### 2.1.4 resize
```java
private void checkAndGrow() {  
    if (size == capacity) {  
        capacity += capacity >>> 1;  
        int[] newArr = new int[capacity];  
        System.arraycopy(array, 0, newArr, 0, size);  
        array = newArr;  
    }  
}
```
# 3 Stack
# 4 Queue
# 5 Tree
## 5.1 B-Tree
- B- 树和 B+ 树的节点一般会映射磁盘文件，大小为磁盘页(4kb)的整数倍
- 多路可以降低树的高度，从而减少磁盘 IO
- 2-3 树是一种特殊的 B-树，它满足：
	- `2-节点` 有 {0, 2} 个子节点
	- `3-节点` 有 {0, 3} 个子节点
	- 所有叶子节点都在同一层
### 5.1.1 B-树的性质

- B 树非叶子节点的结构

![[b_tree_node.svg]]
- $n$: key 的个数
- $p$：指针
- $k$：key
- $p_0$ 至 $p_n$ 从小到大排列用于划分区间
- $k$ 的个数比 $p$ 的个数少一个

- $m$ 阶 B -树的性质
	- 每个节点最多有 $m$ 个子节点，至多含有 $m-1$ 个 `key`
	- 除根节点外，每个非叶子节点至少有 $ceil(\frac{m}{2})$ 个子节点
	- 根节点有 0 或者至少 2 个子节点
	- 叶子节点都在同一层
- B-树将文件以 <font color='red'>Page</font> 为单位进行存储，每个 Page 作为 B-树的一个节点

![[b_tree_page.svg]]

B-树每个节点都存放着索引和数据，搜索可能在非叶子节点结束，最好的情况是 $O(1)$
### 5.1.2 B-树的删除
#### 5.1.2.1 要删除的键在叶子节点中
##### 5.1.2.1.1 不超过子结点 key 下限

![[b_tree_del1.svg]]

- 此时删除 `31` 没有影响(3 阶要求非叶子节点且非根节点有 $ceil(3/2)=2$ 个结点)，直接删除
##### 5.1.2.1.2 超过子结点 key 下限

![[b_tree_del2.svg]]

- 此时不能直接删除 `32`
- 对树结构做调整：从左至右向兄弟节点借用 `key`，借到 `28`
	- `28` 上移，`30` 下移后删除 `32`

![[b_tree_del3.svg]]

##### 5.1.2.1.3 可以合并的情况

![[b_tree_del4.svg]]

- 要删除 `30` 时，左右兄弟节点都不能借节点时，需要进行合并
	- `28` 下移与 `25` 合并后删除 `30`

![[b_tree_del5.svg]]

#### 5.1.2.2 要删除的键在内部节点中

![[b_tree_del6.svg]]

- 后继 `key` 替换掉待删除的 `key` 后，在叶子结点中删除 `key`
	- `35` 替代 `33`，删除 `33`
- 删除后，按删除叶子节点的情况进行调整
	- `32` 上移，`35` 下移

![[b_tree_del7.svg]]

## 5.2 B+Tree
### 5.2.1 B+树的性质
- $m$ 阶 B+树满足
	- 每个节点最多有 $m$ 个子节点
	- 每个非叶子节点至少有 $ceil(\frac{m}{2})$ 个 `key`
	- 子结点个数与 key 的个数相同
	- 叶子节点存有相邻叶子节点的指针，由小到大顺序连接
	- 内部节点只保存索引，不保存数据，数据全部在叶子节点中
### 5.2.2 B+树的插入

- 不超过 4 个节点时直接插入

![[b+insert1.svg]]

- 后续再插入时需要分裂

![[b+insert2.svg]]

- 前 $ceil(\frac{m}{2})=2$ 个记录放在左子节点，剩下的放在右子节点，第 $ceil(\frac{m}{2})+1$ 个记录的 `key` 进位至父节点(索引节点)，与之类似，当索引节点达到 `5` 个，需要进行分裂时，进位后的索引节点在子节点中不会保留

![[b+insert3.svg]]

### 5.2.3 B+树的删除

![[b+del1.svg]]

- 要删除的 `key` 是索引节点的 `key`，向兄弟节点借 `key` 后替换索引节点并删除

![[b+del2.svg]]

- 要删除的 `key` 是索引节点的 `key`，向兄弟节点借不到 `key` 时
- 合并后删除父节点中的 `key`
- B+ 树删除后索引节点中的 `key` 在叶子节点中不一定存在对应的记录
### 5.2.4 MySQL 中的 B+树索引
- MySQL 中索引可以分为两类
	- 主键索引(聚簇索引)，对应 `B+` 树的叶子节点存放的是实际的数据记录
	- 二级索引(非聚簇索引或辅助索引)，对应 `B+` 树的叶子节点只存放主键值和索引值
	- 如果通过非聚簇索引查找一条完整的数据记录，需要先找到非聚簇索引的叶子节点获取主键值，再去主键索引的 `B+` 树查询数据，这个过程也叫<font color='red'>回表</font>
	- 当查询的数据是主键值时，通过非聚簇索引一次就能查找到数据，被称为<font color='red'>索引覆盖</font>
- 联合索引指将多个字段组合成一个索引
	- 遵循最左匹配原则，越靠前的字段被用于索引过滤的概率越高，在建立时要把区分度大的字段排在前面
##  5.3 红黑树
- 高效的自平衡树结构
### 5.3.1 红黑树的规则
- 根节点为黑色
- 每个叶子节点都是黑色的空节点(Nil)
- 从根节点到叶子节点不会出现连续的红色节点
- 从任意一个节点出发到其子节点的所有路径都包含相同个数的黑色节点
### 5.3.2 2-3-4 树节点和红黑树的对应关系

![[rbtree_to_234_tree.svg]]
### 5.3.3 红黑树的插入
#### 5.3.3.1 第一次插入为根节点，染黑
#### 5.3.3.2 第二次插入的为红色节点

![[rbtree_insert1.svg]]

#### 5.3.3.3 插入 3 个节点时，有 4 种需要调整的情况

![[rbtree_insert2.svg]]
- 针对(3)(4)的情况，父节点进行旋转得到(1)(2)
- 之后祖父节点再进行旋转，变色

![[rbtree_insert3.svg]]

#### 5.3.3.4 插入第 4 个节点
##### 5.3.3.4.1 需要变色的情况

![[rbtree_insert4.drawio.svg]]
##### 5.3.3.4.2 不需要变色的情况
- 5.3.3.4.1 中变色后的情况，不需要做操作
#### 5.3.3.5 TreeMap 中的插入
```java
private void fixAfterInsertion(Entry<K,V> x) {  
    // 新插入的 node 是红色
    x.color = RED;  
  
    while (x != null && x != root && x.parent.color == RED) {  
        if (parentOf(x) == leftOf(parentOf(parentOf(x)))) {  
            Entry<K,V> y = rightOf(parentOf(parentOf(x)));  
            if (colorOf(y) == RED) {  
                setColor(parentOf(x), BLACK);  
                setColor(y, BLACK);  
                setColor(parentOf(parentOf(x)), RED);  
                x = parentOf(parentOf(x));  
            } else {  
                if (x == rightOf(parentOf(x))) {  
                    x = parentOf(x);  
                    rotateLeft(x);  
                }  
                setColor(parentOf(x), BLACK);  
                setColor(parentOf(parentOf(x)), RED);  
                rotateRight(parentOf(parentOf(x)));  
            }  
        } else {  
            Entry<K,V> y = leftOf(parentOf(parentOf(x)));  
            if (colorOf(y) == RED) {  
                setColor(parentOf(x), BLACK);  
                setColor(y, BLACK);  
                setColor(parentOf(parentOf(x)), RED);  
                x = parentOf(parentOf(x));  
            } else {  
                if (x == leftOf(parentOf(x))) {  
                    x = parentOf(x);  
                    rotateRight(x);  
                }  
                setColor(parentOf(x), BLACK);  
                setColor(parentOf(parentOf(x)), RED);  
                rotateLeft(parentOf(parentOf(x)));  
            }  
        }  
    }  
    root.color = BLACK;  
}
```
### 5.3.4 红黑树的旋转
- 左旋思路一

![[rotate_left.svg]]

①：用旋转点的值创建一个新节点，左旋要保留左子树，因此将旋转点的左子树作为新节点的左子树
②：将旋转点的右子树的左子树作为新节点的右子树
③：用旋转点的右子节点的值替换旋转节点的值(4 -> 6)
④：此时替换后的右子节点已经没用了，用右子节点的右子树作为旋转点的右子树
⑤：用新节点作为旋转点的左子树
- 左旋思路二：TreeMap 的实现

![[rotate_left2.svg]]

①：用右子树的左子树替代旋转点的右子树
	  如果右子树的左子树不为 `null`，让其指向旋转点，相当于删除
②：旋转点的父节点作为旋转点右子树的父节点
	  相当于将右子树旋转上来
	  此时如果旋转点是其父节点的左子树
	      ③：将右子树作为旋转点父节点的左子树
	             将旋转点作为右子树的左子树
	   如果旋转点是根节点，将右子树直接作为根节点
	   如果旋转点是其父节点的右子树，同上
```java
private void rotateLeft(Entry<K,V> p) {  
    if (p != null) {  
        // 右子节点
        Entry<K,V> r = p.right;  
        // 右子节点左子树指向旋转点的右子节点
        p.right = r.left;  
        // 不为空的话再让左子树指向旋转点
        if (r.left != null)  
            r.left.parent = p; 
		// 右子节点的父节点作为旋转点的父节点(即用右子节点替换了父节点的位置)
        r.parent = p.parent;  
        // 旋转点是根节点，是父节点的左节点或右节点
        // 更改右子节点和父节点的指向
        if (p.parent == null)  
            root = r;  
        else if (p.parent.left == p)  
            p.parent.left = r;  
        else  
            p.parent.right = r;  
	    // 连接旋转后的右子节点和旋转节点
        r.left = p;  
        p.parent = r;  
    }  
}
```
### 5.3.5 红黑树的删除
#### 5.3.5.1 删除规则
- 分三种情况
	- 叶子节点直接删除
	- 有一个子节点的结点，删除后用子节点替代父节点
	- 有两个子节点的节点，找到前驱节点或后继节点，用 `kv` 值更新待删除的节点后删除，这样只用删除一次指针
- TreeMap 中求前驱节点
```java
static <K,V> Entry<K,V> predecessor(Entry<K,V> t) {  
    if (t == null)  
        return null;  
    // 从左边开始找
    else if (t.left != null) {  
        Entry<K,V> p = t.left; 
        // 存在右子树则向右找 
        while (p.right != null)  
            p = p.right;  
        return p;  
    } else {  
        Entry<K,V> p = t.parent;  
        Entry<K,V> ch = t;  
        while (p != null && ch == p.left) {  
	        // 向上寻找父节点，直到第一个父节点的右孩子
            ch = p;  
            p = p.parent;  
        }  
        return p;  
    }  
}
```
- TreeMap 中求后继节点
```java
static <K,V> Entry<K,V> successor(Entry<K,V> t) { 
	if (t == null) {
		return null;
	}
	else if (t.right != null) {
		Entry<K,V> p = t.right;
		while (p.left != null)
			p = p.left;
		return p;
	} else {
		Entry<K,V> p = t.parent;
		Entyr<K,V> ch = t;
		while (p != null && ch == p.right) {
			ch = p;
			p = p.parent;
		}
		return p;
	}
}
```
- TreeMap 中的删除
```java
private void deleteEntry(Entry<K,V> p) {  
    modCount++;  
    size--;  
  
    // If strictly internal, copy successor's element to p and then make p  
    // point to successor.    
    // 获取到后继节点
    if (p.left != null && p.right != null) {  
        Entry<K,V> s = successor(p);  
        p.key = s.key;  
        p.value = s.value;  
        p = s;  
    } // p has 2 children  
  
    // Start fixup at replacement node, if it exists.    
    //后继节点不是叶子节点，有左子节点返回左子节点
    Entry<K,V> replacement = (p.left != null ? p.left : p.right);  
    // 用 replacement 替代 p
    if (replacement != null) {  
        // Link replacement to parent  
        replacement.parent = p.parent;  
        if (p.parent == null)  
            root = replacement;  
        else if (p == p.parent.left)  
            p.parent.left  = replacement;  
        else  
            p.parent.right = replacement;  
  
        // Null out links so they are OK to use by fixAfterDeletion.  
        // 删除掉之前的节点 p
        p.left = p.right = p.parent = null;  

		// 修正
        // Fix replacement  
        if (p.color == BLACK)  
            fixAfterDeletion(replacement);  
    } else if (p.parent == null) { // return if we are the only node.  
        // p 没有子节点，也没有父节点，说明只有他自己
        root = null;  
    } else { 
	    //  No children. Use self as phantom replacement and unlink.  
	    // 叶子节点直接删除
        if (p.color == BLACK)  
            fixAfterDeletion(p);  
        if (p.parent != null) {  
            if (p == p.parent.left)  
                p.parent.left = null;  
            else if (p == p.parent.right)  
                p.parent.right = null;  
            p.parent = null;  
        }  
    }  
}
```
#### 5.3.5.2 红黑树删除和 2-3-4 树的对应

![[rbtree_del_rb_to_234.svg]]

- 删除红黑树的内部节点，可以转换为删除叶子节点或是叶子节点的父节点
- 而叶子节点或叶子节点的父节点，对应的是 2-3-4 树的叶子节点
- 2-3-4 树删除 `3-节点` 中一个元素不需要调整
	- 例如从 `0，1` 中删除红色的 `0`
- 2-3-4 树删除 `4-节点` 中的一个元素不需要调整
	- 例如从 `10，11，12` 中删除 `10` 或 `12`
- 2-3-4 树删除 2-节点需要调整
	- 例如删除 `3`，会违反 `2-节点` 子节点的要求
	- 此时需要向兄弟节点借节点
- 因此删除情况有
	- 不用借节点
		- 删除 `3-节点` 中的红色节点或 `4-节点` 中的红色节点：直接删除
		- 删除 `3-节点` 中的黑色节点或 `4-节点` 中的黑色节点
			- 用子节点替换父节点，变黑(2-3-4 树 `2-节点` 是黑色)
		- 叶子节点直接删除
	- 需要借节点（黑色的叶子节点）
		- 兄弟节点是 `3-节点` 或 `4-节点`，父节点下移，兄弟节点上移后删除
		- 兄弟节点也是 `2-节点`
#### 5.3.5.3 删除后的调整

- <font color='red'>case1. </font> 删除 `3-节点 / 4-节点` 中的黑色节点(B)时，替换的子节点一定是红色节点，变为黑色

![[rbtree_del1.svg]]

- <font color='red'>case2 </font> 删除 `2-节点` 时，首先需要判断兄弟节点是否是 2-3-4 树的兄弟节点
- <font color='red'>sib 为红色时，并不是 2-3-4 树上的兄弟节点</font>
![[rbtree_del2.svg]]

- sib 为黑色时，对应 2-3-4 树上的兄弟节点

![[rbtree_del3.svg]]

- 首先需要将红色的 sib 转换为黑色的对应 2-3-4 树的 sib

![[solve_red_sib.svg]]

- `sib` 变为黑色，`parent` 变为红色，以 `parent` 为旋转点左旋
- 成功转换为能够对应到 2-3-4 树的情况
- <font color='red'>case2.1 </font> sib 是 3-节点
	- 需要让 `sib` 的 `child` 上去充当 `parent`

![[sib_one_node.svg]]

- sib 有左孩子
	- `sib` 为黑色，其 `child` 必为红色
	- `sib` 和 `sib` 的左孩子变色
	- `sib` 右旋后指向 `node` 的 `parent`
- sib 有右孩子
	- 处理情况同 `sib` 是 `4-节点` 的情况(<font color='red'>case 2.2 </font>)
	- 旋转后要让 sib 成为 parent，为了不破坏之前的平衡，让 `sib` 变为 `parent` 的颜色
	- `parent` 变为黑色
	- `sib` 的右孩子变为黑色
	- `parent` 左旋
- <font color='red'>case 2.2 </font> `sib` 是 `4-节点`

![[sib_two_node.svg]]

- <font color='red'>case 3 </font> sib 也是 2-节点

![[sib_no_node.svg]]

①：`parent` 是红色时，为了平衡黑色节点个数，让 `sib` 变为黑色即可
②：`parent` 不是红色时，让 `sib` 变为红色，进行递归
	  如果 `parent` 不是红色，让 `sib` 变为红色，继续向上
	  直到递归到红色节点，跳出 `while` 循环后让 `parent` 变为黑色
- TreeMap 的调整
```java
private void fixAfterDeletion(Entry<K,V> x) {
    // replacement 为黑色，需要调整
    while (x != root && colorOf(x) == BLACK) {
	    // 左右两种情况  
        if (x == leftOf(parentOf(x))) {  
            Entry<K,V> sib = rightOf(parentOf(x));  
			// 修正 sib 节点
            if (colorOf(sib) == RED) {  
                setColor(sib, BLACK);  
                setColor(parentOf(x), RED);  
                rotateLeft(parentOf(x));  
                sib = rightOf(parentOf(x));  
            }  
			// == BLACK 等价于 null
			// case 3
            if (colorOf(leftOf(sib))  == BLACK &&  
                colorOf(rightOf(sib)) == BLACK) {  
                setColor(sib, RED);  
                // 向上递归
                x = parentOf(x);  
            } else {  
	            // case 2.1 
                if (colorOf(rightOf(sib)) == BLACK) {  
                    setColor(leftOf(sib), BLACK);  
                    setColor(sib, RED);  
                    rotateRight(sib);  
                    sib = rightOf(parentOf(x));  
                }  
                // case 2.2
                setColor(sib, colorOf(parentOf(x)));  
                setColor(parentOf(x), BLACK);  
                setColor(rightOf(sib), BLACK);  
                rotateLeft(parentOf(x));  
                x = root;  
            }  
        } else { // symmetric  
            Entry<K,V> sib = leftOf(parentOf(x));  
  
            if (colorOf(sib) == RED) {  
                setColor(sib, BLACK);  
                setColor(parentOf(x), RED);  
                rotateRight(parentOf(x));  
                sib = leftOf(parentOf(x));  
            }  
  
            if (colorOf(rightOf(sib)) == BLACK &&  
                colorOf(leftOf(sib)) == BLACK) {  
                setColor(sib, RED);  
                x = parentOf(x);  
            } else {  
                if (colorOf(leftOf(sib)) == BLACK) {  
                    setColor(rightOf(sib), BLACK);  
                    setColor(sib, RED);  
                    rotateLeft(sib);  
                    sib = leftOf(parentOf(x));  
                }  
                setColor(sib, colorOf(parentOf(x)));  
                setColor(parentOf(x), BLACK);  
                setColor(leftOf(sib), BLACK);  
                rotateRight(parentOf(x));  
                x = root;  
            }  
        }  
    }  
    // 跳出循环或 case1, 将节点变黑
    setColor(x, BLACK);  
}
```
## 5.4 LSMTree
- Log-Structured-Merge-Tree
- 利用顺序写提高写性能，分为内存和文件两部分，牺牲小部分读性能换高性能写
- 分为 level0, level1, ..., level n 多层子树，第 0 层在内存中，其余在磁盘中
- level0 子树采用排序树，跳表等有序数据结构方便顺序写磁盘
### 5.4.1 组成

![[lsm.svg]]
- MemTable
	- 内存中的数据结构，用于保存最近更新的数据
	- 通过 `wal` 保证数据可靠性
- Immutable MemTable
	- MemTable 达到一定大小后转为 Immutable MemTable
- SSTable(Sorted String Table)
	- 磁盘中的数据结构
	- 有序键值对集合
### 5.4.2 删除与修改
- 数据在内存中
	- 插入删除标记覆盖数据
- 数据在磁盘中
	- 向内存中插入删除标记即可
- 数据不存在
	- 同上
- 可以看出删除的操作始终等价于向内存中写入删除标记，代价很低
- 修改与删除类似，都是只针对于内存
### 5.4.3 查询
- 按顺序查找 level0, level1, ..., level n，最先找到的肯定是最新的，直接返回
# 6 Hash
## 6.1 HashTable
- 每份数据分配一个编号，放入有限长度的数组
- 重复索引用链表解决
```java
public class HashTable {  
  
    static class Entry {  
        int hash;  
        Object key;  
        Object value;  
        Entry next;  
  
        public Entry(int hash, Object key, Object value) {  
            this.hash = hash;  
            this.key = key;  
            this.value = value;  
        }  
    }  
  
    // 数组中存储头结点  
    // 2^n 可以将 mod 转为 hash & (len - 1)    
    Entry[] table = new Entry[16];  
    int size;
    float loadFactor = 0.75f;    
	int threshold = (int) (loadFactor * table.length);
}
```
### 6.1.1 get()
```java
// 根据 hash 获取 value
Object get(int hash, Object key) {  
    int ind = hash & (table.length - 1);  
    if (table[ind] != null) {  
        Entry p = table[ind];  
        while (p != null) {  
            if (p.key.equals(key)) {  
                return p.value;  
            }  
            p = p.next;  
        }  
    }  
    return null;  
}
```
### 6.1.2 put()
```java
void put(int hash, Object key, Object value) {  
    int ind = hash & (table.length - 1);  
    if (table[ind] == null) {  
        table[ind] = new Entry(hash, key, value);  
    } else {  
        Entry p = table[ind];  
        while (true) {  
            if (p.key.equals(key)) {  
                p.value = value;  
                return;  
            }  
            if (p.next == null) {  
                break;  
            }  
            p = p.next;  
        }  
        p.next = new Entry(hash, key, value);  
    }  
    size++;  
}
```
### 6.1.3 remove()
```java
Object remove(int hash, Object key) {  
    int ind = hash & (table.length - 1);  
    if (table[ind] == null) {  
        return null;  
    } else {  
        Entry p = table[ind];  
        Entry prev = null;  
        while (p != null) {  
            if (p.key.equals(key)) {  
                if (prev == null) {  
                    table[ind] = p.next;  
                } else {  
                    prev.next = p.next;  
                }  
                size--;  
                return p.value;  
            }  
            prev = p;  
            p = p.next;  
        }  
    }  
    return null;  
}
```
### 6.1.4 resize()
- hash & (len - 1)
	- 10 进制中除以 `10, 100, 1000` 时, 余数是被除数的后 `1, 2, 3` 位
	- 2 进制中除以 `10, 100, 1000` 时, 余数是被除数的后 `1, 2, 3` 位, 即只保留最后几位即可, 均是 $2^n-1$ 即 `len - 1`
- 链表拆分
	- 旧数组长度换成二进制后，其中的 1 就是要检查的倒数第几位
		- 8 -> 1000 检查倒数第 4 位
		- 16 -> 10000 检查倒数第 5 位
	- hash & 旧数组长度，检查前后索引位置(余数)会不会变
		- `00000000-00000111 & 00001000` 为 0, 不变
		- `00001000~ & 00001000` 不为 0, 变
```java
private void resize() {  
    Entry[] newTable = new Entry[table.length << 1];  
    for (int i = 0; i < table.length; i++) {  
        Entry p = table[i]; // 每个链表头  
        if (p != null) {  ``
            // 拆分链表，移动到新数组  
            /*  
             * 拆分规律：  
             *   一个链表最多拆成两个  
             *      hash & table.length == 0 的一组  
             *      hash & table.length != 0 的一组  
             */  
            Entry a = null;  
            Entry b = null;  
            Entry aHead = null;  
            Entry bHead = null;  
            while (p != null) {  
                if ((p.hash & table.length) == 0) {  
                    // 分配给 a                    
                    if (a != null) {  
                        a.next = p;  
                    } else {  
                        aHead = p;  
                    }  
                    a = p;  
                } else {  
                    if (b != null) {  
                        b.next = p;  
                    } else {  
                        bHead = p;  
                    }  
                    b = p;  
                }  
                p = p.next;  
            }  
            if (a != null) {  
                a.next = null;  
                // a链表位置不变, b链表移动数组长度个位置  
                newTable[i] = aHead;  
            }  
            if (b != null) {  
                b.next = null;  
                newTable[i + table.length] = bHead;  
            }  
        }  
    }  
    table = newTable;  
    threshold = (int) (loadFactor * table.length);  
}
```
## 6.2 hash 算法
- 常见的有 MD5, SHA1, SHA256, SHA512, CRC32
- 摘要算法，散列算法
- String 的 hash
``` java
int hash = 0;  
for (int i = 0; i < s1.length(); i++) {  
    char c = s1.charAt(i);  
    System.out.println((int) c);  
    // hash = hash * 31 + c;  
    hash = (hash << 5) - hash + c;
}
```
## 6.3 LeetCode
### Q1(S) -- 两数之和
- 给定一个整数数组 nums 和一个整数目标值 target，在该数组中找到和为目标值的两个整数，返回下标
```
example：
	input: 
		nums = [2, 7, 11, 15], target = 9
	output: [0, 1]

	input:
		nums =  [3, 3], target = 6
	output: [0, 1]
```
#### solution
- 使用 hash 表存放对应的数字和索引
- 遍历数组，找到 sum 至 target 所需要的另一个数
- 去 hash 表中找另一个加数，不存在时将当前数和索引 put 到 map 中
```java
public int[] sumTwo(int[] nums, int target) {  
    HashMap<Integer, Integer> map = new HashMap<>();  
    for (int i = 0; i < nums.length; i++) {  
        int n = nums[i];  
        int diff = target - n;  
        if (map.containsKey(diff)) {  
            return new int[]{map.get(diff), i};  
        } else {  
            map.put(n, i);  
        }  
    }  
    return null;  
}
```
### Q3(M) -- 无重复字符的最长子串
- 给定一个字符串 s, 找出其中不含有重复字符的最长子串的长度
```
example:
	input: s = "abcabcbb"
	output: 3 // "abc"

	input: s = "bbbbb"
	output: 1

	input: s = "pwwkew"
	output: 3
```
#### solution
```java
/*
 * 滑动窗口解法
 * 用 begin 和 end 表示子串的开始和结束位置
 * 用 hash 表检查重复字符
 * 从左向右查看
 *    没遇到重复字符，调整 end
 *    遇到重复字符，调整 begin
 *    将当前字符放入 hash 表
 * end - begin + 1 是当前子串长度
*/
public int findLongestSub(String s) {  
        HashMap<Character, Integer> map = new HashMap<>();  
        int begin = 0;  
        int maxLen = 0;  
        for (int end = 0; end < s.length(); end++) {  
            char ch = s.charAt(end);  
            if (map.containsKey(ch)) {  
                // 防止类似 abba 重复情况 begin 回退的问题  
                begin = Integer.max(begin, map.get(ch) + 1);  
                map.put(ch, end);  
            } else {  
                map.put(ch, end);  
            }  
//            System.out.println(s.substring(begin, end + 1));  
            maxLen = Integer.max(maxLen, end - begin + 1);  
        }  
  
        return maxLen;  
    }
```
### Q49(M) -- 字母异位词分组
- 给一个字符串数组，返回结果列表
- 字母异位词指字母相同，排列顺序不同的词
- str\[i]只包含小写字母
```
example:
	input: 
		strs = ["eat", "tea", "tan", "ate", "nat", "bat"]
	output: 
		[["bat"], ["nat", "tan"], ["ate", "eat", "tea"]]
```
#### solution1
- 遍历字符串数组，每个字符串中的字符重新排序后作为 key
- 准备一个集合，把这些词加入到 key 相同的集合中
```java
public List<List<String>> groupAnagrams(String[] strs) {  
  
    HashMap<String, List<String>> map = new HashMap<>();  
    for (String str : strs) {  
        char[] chars = str.toCharArray();  
        Arrays.sort(chars);  
        String key = new String(chars);  
        List<String> list = map.get(key);  
        // if (list == null) {  
        //    list = new ArrayList<>();  
        //    map.put(key, list);  
        // }
        // key 不存在对应的映射时，将其映射至后面的函数中
        List<String> list = map.computeIfAbsent(
	        key, k -> new ArrayList<>());  
        list.add(str);  
    }  
  
    return new ArrayList<>(map.values());  
}
```
#### solution2
- 仅包含小写字母，可以通过数组计数
```java
// 通过 ArrayKey 分组，需要重写 equals 和 hashcode，因此进行封装
static class ArrayKey{  
    int[] key = new int[26];  
  
    public ArrayKey(String str) {  
        for (int i = 0; i < str.length(); i++) {  
            char ch = str.charAt(i);  
            key[ch - 97]++;  
        }  
    }  
  
    @Override  
    public boolean equals(Object o) {...}  
  
    @Override  
    public int hashCode() {  
        return Arrays.hashCode(key);  
    }
}

public List<List<String>> groupAnagrams(String[] strs) {  
    HashMap<ArrayKey, List<String>> map = new HashMap<>();  
    for (String str : strs) {  
        ArrayKey key = new ArrayKey(str);  
        List<String> list = map.computeIfAbsent(key, k -> new ArrayList<>());  
        list.add(str);  
    }  
    return new ArrayList<>(map.values());  
}
```
### Q217(S) -- 存在重复元素
- 给一个整数数组 nums，判断是否存在重复元素
```
example：
	input: nums = [1, 2, 3, 1]
	output: true
```
#### solution
```java
public boolean containsDuplicate(int[] nums) {  
  
    Map<Integer, Object> map = new HashMap<>();  
    Object o = new Object();  
    for (int num : nums) {  
        if (map.put(num, o) != null) {  
            return true;  
        }  
    }  
    return false;  
}
```
### Q136(S) -- 只出现一次的数字
- 给一个非空整数数组 nums，除了某个元素只出现一次，其余每个元素均出现两次，找到只出现一次的元素
```
example:
	input: nums = [2, 2, 1]
	output: 1
```
#### solution1
- 向 hashSet 添加元素，遇到重复的删除
```java
public int singleNumber(int[] nums) {  
    HashSet<Integer> set = new HashSet<>();  
    for (int num : nums) {  
        if (!set.add(num)) { // 出现重复元素  
            set.remove(num);  
        }  
    }  
    // toArray(T [])
    // new Integer[0] 相当于在不占用空间的前提下指定了泛型
    return set.toArray(new Integer[0])[0];  
}
```
#### solution2
- 相同的数字异或，结果是 0
- 任何数字与 0 异或，结果是本身
- 从第一个元素开始向后异或，最终结果就是只出现一次的元素
```java
public int singleNumber(int[] nums) {  
    int num = nums[0];  
    for (int i = 1; i < nums.length; i++) {  
        num ^= nums[i];  
    }  
    return num;  
}
```
### Q242(S) -- 有效的字母异位词
- 给定两个字符串 t 和 s，判断 s 和 t 是否为字母异位词
```
example：
	input: 
		s = "anagram", t = "nagaram"
	output: true
```
#### solution
- 与之前类似，通过 26 个字母的数组进行比较
```java
public int[] getKey(String s) {  
        int[] array = new int[26];  
	// for (int i = 0; i < s.length(); i++) {  
	//    ch = s.charAt(i);  
	//    array[ch - 97]++;  
	//    }  
        char[] chars = s.toCharArray();  
        for (char ch : chars) {  
            array[ch - 97]++;  
        }  
        return array;  
    }
```
### Q387(S) -- 字符串中第一个唯一字符
- 给定一个字符串 s，找到它的第一个不重复字符，返回索引，不存在返回 -1
```
example：
	input: s = "leetcode"
	output: 0

	input: s = "aabb"
	output: -1
```
#### solution
```java
public int firstUniqChar(String str) {  
    int[] array = new int[26];  
    char[] chars = str.toCharArray();  
    for (char ch : chars) {  
        array[ch - 97]++;  
    }  
    for (int i = 0; i < chars.length; i++) {  
        if (array[chars[i] - 97] == 1) {  
            return i;  
        }  
    }  
    return -1;  
}
```
### Q819(S) -- 最常见的单词
- 给你一个字符串 `paragraph` 和一个表示禁用词的字符串数组 `banned`, 返回出现频率最高的非禁用词
- 至少存在一个非禁用词，且答案 **唯一**
- `paragraph` 中的单词 **不区分大小写**
-  `paragraph` 由英文字母、空格 `' '`、和以下符号组成：`"!?',;."`
```
example:
	input:
		paragraph = "Bob hit a ball, the hit BALL flew far after it was hit.", banned = ["hit"]
	output: "ball"
```
#### solution
```java
public String mostCommonWord(String para, String[] banned) {  
	String[] words = para.toLowerCase().split("[^A-Za-z]+");  
	Set<String> set = Set.of(banned);  
	HashMap<String, Integer> map = new HashMap<>();  
	for (String word : words) {  
		if (!set.contains(word)) {  
//            Integer value = map.get(word);  
//            if (value == null) {  
//                map.put(word, 1);  
//            } else {  
//                map.put(word, ++value);  
//            }  
			map.compute(word, (k, v) -> v == null ? 1 : ++v);  
		}  
	}  
	Optional<Map.Entry<String, Integer>> max = map.entrySet().stream().max(Map.Entry.comparingByValue());  
	return max.map(Map.Entry::getKey).orElse(null);  
}

// 改进
public String mostCommonWord(String para, String[] banned) { 
	// 截取段落
	char[] chars = para.toLowerCase().toCharArray();  
	StringBuilder sb = new StringBuilder();  
	for (char ch : chars) {  
		if (ch >= 'a' && ch <= 'z') {  
			sb.append(ch);  
		} else {  
			String word = sb.toString();  
			if (!set.contains(word)) {  
			map.compute(word, (k, v) -> v == null ? 1 : ++v);  
			}  
//                sb = new StringBuilder();  
			sb.setLength(0);  
		}  
        if (!sb.isEmpty()) { // 只有一个单词  
            String word = sb.toString();  
            if (!set.contains(word)) {  
                map.compute(word, (k, v) -> v == null ? 1 : ++v);  
            }  
        }
	...
	// 获取到 map 后，找到次数最多的单词
	int max = 0;
	String maxWord = null;
	for (Map.Entry<String, Integer> e: map.entrySet()) {
		Integer value = e.getValue();
		if(value > max) {
			max = value;
			maxWord = e.getKey();
		}
	}
	return maxKey;
}

```
# 7 Sort
| 算法 | 最好       | 最坏       | 平均       | 空间   | 稳定 | 思想 |
| ---- | ---------- | ---------- | ---------- | ------ | ---- | ---- |
| 冒泡 | $O(n)$     | $O(n^2)$   | $O(n^2)$   | $O(1)$ | Y    | 比较 |
| 选择 | $O(n^2)$   | $O(n^2)$   | $O(n^2)$   | $O(1)$ | N    | 比较 |
| 堆   | $O(nlogn)$ | $O(nlogn)$ | $O(nlogn)$ | $O(1)$ | N    | 选择 |
| 插入 | $O(n)$     | $O(n^2)$   | $O(n^2)$   | $O(1)$ | Y    | 比较 |
| 希尔 | $O(nlogn)$ | $O(n^2)$   | $O(nlogn)$ | $O(1)$ | N    | 插入 |
| 归并 | $O(nlogn)$ | $O(nlogn)$ | $O(nlogn)$ | $O(n)$ | Y    | 归并 |
| 快速     | $O(nlogn)$           | $O(n^2)$           | $O(nlogn)$           | $O(logn)$       | N     | 分治     |
- 稳定：相同元素排序前后不会交换位置
## 7.1 bubble
- 每轮冒泡不断地比较相邻两个元素，如果是逆序地，则交换他们的位置
- 下一轮冒泡可以调整未排序的右边界，减少不必要比较
![[bubble.svg]]

- 可以在每轮结束后记录交换的索引位置为右边界
```java
// 递归实现
public void bubble(int[] nums, int bound) {
	if (bound == 0) return;
	int x = 0;
	for (int i = 0; i < bound: i++) {
		if (nums[i] > nums[i + 1]) {
			int t = nums[i];
			nums[i] = nums[i + 1];
			nums[i + 1] = t;
			x = i;
		}
	}
	bubble(nums, x);
}

// 非递归
public void bubble(int[] nums, int bound) {
	int bound = nums.length - 1;
	do {
		int x = 0;
		for (int i = 0; i < bound: i++) {
		if (nums[i] > nums[i + 1]) {
			int t = nums[i];
			nums[i] = nums[i + 1];
			nums[i + 1] = t;
			x = i;
		}
		j = x;
	} while (j != 0);
}
```
## 7.2 select
- 交换次数一般少于冒泡
- 每一轮选择找出最大(小的)元素，交换到合适的位置
![[select_sort.svg]]

```java
static void sort(int[] elems) {  
    // 轮数: length - 1  
    for (int bound = elems.length - 1; bound > 0; bound--) {  
        int max = bound;  
        for (int i = 0; i < bound; i++) {  
            if (elems[i] > elems[max]) {  
                max = i;  
            }  
        }  
        swap(elems, max, bound);  
    }  
}
```
## 7.3 heap
- 建立大顶堆
- 每次将堆顶元素(最大值)交换到末尾，调整堆顶元素，重新符合大顶堆特性
![[heap_sort.svg]]

```java
public class HeapSort {  
  
    static void heapify(int[] array, int size) {  
        for (int i = size / 2 - 1; i >= 0; i--) {  
            down(array, i, size);  
        }  
    }  
  
    static void down(int[] array, int parent, int size) {  
        while (true) {  
            int left = parent * 2 + 1;  
            int right = left + 1;  
            int max = parent;  
            if (left < size && array[left] > array[parent]) {  
                max = left;  
            }  
            if (right < size && array[right] > array[parent]) {  
                max = right;  
            }  
            if (max == parent) {  
                break;  
            }  
            swap(array, max, parent);  
            parent = max;  
        }  
    }  
  
    static void swap(int[] arr, int i, int j) {  
        int tmp = arr[i];  
        arr[i] = arr[j];  
        arr[j] = tmp;  
  
    }  
  
    static void sort(int[] arr) {  
        heapify(arr, arr.length);  
        for (int bound = arr.length - 1; bound > 0; bound--) {  
            swap(arr, 0, bound);  
            down(arr, 0, bound);  
        }  
    }
}
```
## 7.4 insert
- 将数组分为两部分 `[0, low - 1]` `[low, arr.length - 1]`
- 每次从右边的未排序区取出 `arr[low]` 插入到已排序区域
![[insert_sort.svg]]

```java
static void sort(int[] arr) {  
    for (int low = 1; low < arr.length; low++) {  
        int t = arr[low];  
        int i = low - 1;  
        // 从右向左找插入位置，如果比插入元素大，不断右移  
        while (i >= 0 && t < arr[i]) {  
            arr[i + 1] = arr[i];  
            i--;  
        }  
        // 找到了插入位置  
        if (i + 1 != low) { // low 的位置需要变  
            arr[i + 1] = t;  
        }  
    }  
}
```
## 7.5 Shell
- 分组实现插入，每组元素间隙称为 gap
- 每轮排序后 gap 逐渐变小，直至 gap 为 1
- 对插入排序的优化，让元素更快地交换到最终位置
![[shell_sort.svg]]
- gap = 1 进行最后一轮调整

```java
static void sort(int[] arr) {  
	// 初始 gap 长度为 length / 2, 每轮 /= 2
    for (int gap = arr.length >> 1; gap >= 1; gap = gap >> 1) { 
	    // 将插入排序中的 1 改为 gap 即可 
        for (int low = gap; low < arr.length; low++) {  
            int t = arr[low];  
            int i = low - gap;  
            // 从右向左找插入位置，如果比插入元素大，不断右移  
            while (i >= 0 && t < arr[i]) {  
                arr[i + gap] = arr[i];  
                i -= gap;  
            }  
            // 找到了插入位置  
            if (i + gap != low) { // low 的位置需要变  
                arr[i + gap] = t;  
            }  
        }  
    }  
}
```
## 7.6 merge
- 分而治之, 合并有序数组
- 自上而下(归并)
```java
public class MergeSort {  
    // 合并数组内两个区间的有序元素  
    static void merge(int[] a1, int i, int iEnd, int j, int jEnd, int[] a2) {  
        int k = 0;  
        while (i <= iEnd && j <= jEnd) {  
            if (a1[i] < a1[j]) {  
                a2[k] = a1[i];  
                i++;  
            } else {  
                a2[k] = a1[j];  
                j++;  
            }  
            k++;  
        }  
        if (i > iEnd) {  
            System.arraycopy(a1, j, a2, k, jEnd - j + 1);  
        }  
        if (j > jEnd) {  
            System.arraycopy(a1, i, a2, k, iEnd - i + 1);  
        }  
    }  
  
    static void sort(int[] arr) {  
        int[] tmp = new int[arr.length];  
        split(arr, 0, arr.length - 1, tmp);  
    }  
  
    static void split(int[] arr, int left, int right, int[] tmp) {  
        if (left == right) return;  
        int m = (left + right) >>> 1;  
        split(arr, left, m, tmp);  
        split(arr, m + 1, right, tmp);  
        merge(arr, left, m, m + 1, right, tmp);  
        System.arraycopy(tmp, left, arr, left, right - left + 1);  
    }  
}
```
- 自下而上(非递归)
```java
static void sort(int[] arr) {  
        int n = arr.length;  
        int[] tmp = new int[n];  
        // 每次合并有序数组的宽度  
        for (int width = 1; width < n; width *= 2) { 
            for (int left = 0; left < n; left += 2 * width) {  
	            // 下次的 left - 1  
                int right = Math.min(left + 2 * width - 1, n - 1); 
                int m = Math.min(left + width - 1, n - 1);  
                merge(arr, left, m, m + 1, right, tmp);  
            }  
            System.arraycopy(tmp, 0, arr, 0, n);  
        }  
  
    }
```
## 7.7 merge + insert
- 将递归的归并加上插入排序
- 当 left 和 right 的距离较近时，采用插入排序
```java
// 修改插入排序
static void insert(int[] arr, int left, int right) {  
    for (int low = left + 1; low <= right; low++) {  
        int t = arr[low];  
        int i = low - 1;  
        // 从右想左找插入位置，如果比插入元素大，不断右移  
        while (i >= left && t < arr[i]) {  
            arr[i + 1] = arr[i];  
            i--;  
        }  
        // 找到了插入位置  
        if (i + 1 != low) { // low 的位置需要变  
            arr[i + 1] = t;  
        }  
    }  
}

// 修改 split
static void split(int[] arr, int left, int right, int[] tmp) {  
    if (right - left <= 32) {  
        insert(arr, left, right);  
        return;  
    }  
    int m = (left + right) >>> 1;  
    split(arr, left, m, tmp);  
    split(arr, m + 1, right, tmp);  
    merge(arr, left, m, m + 1, right, tmp);  
    System.arraycopy(tmp, left, arr, left, right - left + 1);  
}
```
## 7.8 fast
### 7.8.1 单边循环快排 (Lomuto 分区方案)
- 每轮找到一个基准点元素，比它小的放在左边，比它大的放在右边
	- 选择最右侧元素作为基准点元素
	- j 找比基准点小的，i 找大的，找到后二者进行交换
		- j 找到小的且与 i 不相等
		- i 找到大的不自增
	- 基准点与 i 交换，i 为最终索引
![[quick_sort_lomuto.svg]]

```java
public class QuickSortLomuto {  
  
    static void quick(int[] arr, int left, int right) {  
        if (left >= right) return;  
        // 寻找基准点元素  
        int p = partition(arr, left, right);  
        quick(arr, left, p - 1);  
        quick(arr, p + 1, right);  
    }  
  
    private static int partition(int[] arr, int left, int right) {  
        int pv = arr[right]; // 基准点  
        int i = left;  
        int j = left;  
        while (j < right) {  
            if (arr[j] < pv) { // 找到大于基准点的，相当于没找到小于基准点的，i自增  
                if (i != j) {  
                    swap(arr, i, j);  
                }  
                i++;  
            }  
            j++;  
        }  
        swap(arr, i, right);  
        return i;  
    }  
  
    static void sort(int[] arr) {  
        quick(arr, 0, arr.length - 1);  
    }   
}
```
### 7.8.2 双边循环快排
- 选择最左侧元素作为基准点
- j 找小的，i 找大的，找到进行交换
	- i 从左向右
	- j 从右向左
- 最后 i 和基准点元素进行交换
![[quick_sort_bilateral.svg]]

```java
// 只修改 partition 方法
private static int partition(int[] arr, int left, int right) {  
    int pv = arr[left]; // 基准点  
    int i = left;  
    int j = right;  
    while (i < j) {  
        // j 从右向左找小的  
        // 内层循环 i < j 防止不停移动
        // 必须右边先走才能保证基准点在 i 的位置
        while (i < j && arr[j] > pv) j--;  
        while (i < j && arr[i] <= pv) i++;  
        swap(arr, i, j);  
    }  
    swap(arr, left, i);  
    return i;  
}
```
### 7.8.3 快排的问题
#### 7.8.3.1 随机基准点
- 存在的问题：极端情况下，分区结果不理想
- 此时排序的复杂度为 $O(n^2)$
![[quick_sort_random.svg]]

- 通过随机基准点来解决
```java
private static int partition(int[] arr, int left, int right) {  
	// 生成一个 [left, right] 的随机数
	int rand = ThreadLocalRnadom.current().nextInt(right - left + 1) + left;
	swap(arr, rand, left);
    int pv = arr[left]; // 基准点  
    int i = left;  
    int j = right;  
    while (i < j) {  
        // j 从右向左找小的  
        while (i < j && arr[j] > pv) j--;  
        while (i < j && arr[i] <= pv) i++;  
        swap(arr, i, j);  
    }  
    swap(arr, left, i);  
    return i;  
}

public void swap(int[] arr, int i, int j) {
	int t = arr[i];
	arr[i] = arr[j]

}
```
#### 7.8.3.2 大量重复元素
- 遇到大量重复元素时，基准点最终的位置在最右边，分区性能不好
- 改进
	- 循环内
		- i 从 left + 1 开始，从左到右找大于等于的
		- j 从 right 开始，从右向左找小于等于的
		- 交换元素，i++，j++
	- 循环外 j 和基准点交换，j 即为分区位置
- 为什么将 i < j 更改为 i <= j
	- i, j 重合时，j 会再进行一次循环，更改 j 的位置，得到正确结果
```java
private static int partition(int[] arr, int left, int right) {  
    int pv = arr[left]; // 基准点  
    int i = left + 1;  
    int j = right;  
    while (i <= j) {  
        while (i <= j && arr[i] < pv) i++;  
        while (i <= j && arr[j] > pv) j--;  
        if (i <= j) {  
            swap(arr, i, j);  
            i++;  
            j--;  
        }  
    }  
    swap(arr, left, j);  
    return j;  
}
```
## 7.9 count
- 找到数组的最大值，创建一个大小为 max + 1 的数组 count
- count 数组中索引对应原始数组元素，值对应于出现的次数
- 遍历 count 数组，根据元素及其出现次数，生成排序结果
```java
static void sort(int[] arr) {  
    int max = arr[0];  
    for (int i = 1; i < arr.length; i++) {  
        if (arr[i] > max) max = arr[i];  
    }  
    int[] cnt = new int[max + 1];  
    for (int e : arr) {  
        cnt[e]++;  
    }  
    int k = 0;  
    for (int i = 0; i < cnt.length; i++) {  
        // i 元素出现 cnt[i] 次  
        while (cnt[i] > 0) {  
            arr[k++] = i; // 放入一个元素后，索引 +1
            cnt[i]--;  
        }  
    }  
}
```
- 缺点：所有元素都要大于等于 0，且最大值不能太大
	- 需要找到最小值，让最小值映射至索引 0
```java
static void sort(int[] arr) {  
    int max = arr[0];  
    int min = arr[0];  
    for (int i = 1; i < arr.length; i++) {  
        if (arr[i] > max) max = arr[i];  
        if (arr[i] < min) min = arr[i];  
    }  
    int[] cnt = new int[max - min + 1];  
    for (int e : arr) {  
        // 原始数组元素 - 最小值 = cnt 索引  
        cnt[e - min]++;  
    }  
    int k = 0;  
    for (int i = 0; i < cnt.length; i++) {  
        // i + min 出现 cnt[i] 次  
        while (cnt[i] > 0) {  
            arr[k++] = i + min;  
            cnt[i]--;  
        }  
    }  
}
```
## 7.10 bucket
```java
static void sort(int[] ages) {  
    DynamicArray[] buckets = new DynamicArray[10];  
    for (int i = 0; i < buckets.length; i++) {  
        buckets[i] = new DynamicArray();  
    }  
  
    for (int age : ages) {  
        buckets[age / 10].addLast(age);  
    }  
    int k = 0;  
    for (DynamicArray bucket : buckets) {  
        int[] arr = bucket.array(); // 获取每个桶中元素
        // 桶内排序  
        InsertSort.sort(arr);  
        System.out.println(Arrays.toString(arr));  
        for (int e : arr) {  
            ages[k++] = e;  
        }  
    }  
}
```
- 改进：解决数据倾斜
```java
static void sort(int[] ages) {  
    DynamicArray[] buckets = new DynamicArray[(max - min) / range + 1];  
    for (int i = 0; i < buckets.length; i++) {  
        buckets[i] = new DynamicArray();  
    }  
  
    for (int age : ages) {  
        buckets[(age - min) / range].addLast(age);  
    }  
    int k = 0;  
    for (DynamicArray bucket : buckets) {  
        int[] arr = bucket.array(); // 获取每个桶中元素
        // 桶内排序  
        InsertSort.sort(arr);  
        System.out.println(Arrays.toString(arr));  
        for (int e : arr) {  
            ages[k++] = e;  
        }  
    }  
}
```
## 7.11 radix(基数排序)
- 按位比较字符，放入对应的桶中
```java
static void radixSort(String[] arr, int length) {  
	// 只有数字时
    // ArrayList<String>[] buckets = new ArrayList[10];  
    // ascii
    ArrayList<String>[] buckets = new ArrayList[128];
    for (int i = 0; i < buckets.length; i++) {  
        buckets[i] = new ArrayList<>();  
    }  
    for (int i = length - 1; i >= 0; i--) {  
        for (String s : arr) {  
            // buckets[s.charAt(i) - '0'].add(s); // '0' 的索引映射为 0        
            buckets[i].add(s)
		}  
        int k = 0;  
        for (ArrayList<String> bucket : buckets) {  
            for (String s : bucket) {  
                arr[k++] = s;  
            }  
            // 清空当前的桶  
            bucket.clear();  
        }  
    }  
}
```
# 8 Graph
## 8.1 基本知识
### 定义
- 图由顶点和边组成
- 有向图/无向图
- 度指与该顶点相连的边的数量
	- 有向图中进一步分为入度(指向该点)和出度(该点指出)
- 边可以由权重, 顶点间的度量
- 所有顶点都连通，称为连通图，子图连通称为连通分量
### 表示
- 无向图
![[graph_present1.svg]]
- 邻接矩阵
```
   A  B  C  D
A  0  1  1  0
B  1  0  0  1
C  1  0  0  1
D  0  1  1  0
```
- 邻接表
```
A -> B -> C
B -> A -> D
C -> A -> D
D -> B -> C
```
- 有向图
![[graph_present2.svg]]
- 邻接矩阵
```
   A  B  C  D
A  0  1  1  0
B  0  0  0  1
C  0  0  0  1
D  0  0  0  0
```
- 邻接表
```
A -> B -> C
B -> D
C -> D
D
```
- Java 表示
```java
public class Vertex {  
      
    String name;  
    List<Edge> edges;  
  
    public Vertex(String name) {  
        this.name = name;  
    }

	public static void main(String[] args) {  
  
    Vertex a = new Vertex("A");  
    Vertex b = new Vertex("B");  
    Vertex c = new Vertex("C");  
    Vertex d = new Vertex("D");  

	// a -> b, a -> c  
    a.edges = List.of(new Edge(b), new Edge(c)); 
    b.edges = List.of(new Edge(d));  
    c.edges = List.of(new Edge(d));  
    d.edges = List.of();  
}
}

public class Edge {  
  
    Vertex linked; // 连接的点  
    int weight;  
  
    public Edge(Vertex linked) {  
        this(linked, 1);  
    }  
  
    public Edge(Vertex linked, int weight) {  
        this.linked = linked;  
        this.weight = weight;  
    }  
}


```
## 8.2 DFS
![[dfs.svg]]
1 -> 3 -> 4 -> 5, 返回到 3
3 -> 6 -> 5, 返回到 1
1 -> 2 -> 4, 返回到 1
1 -> 6 -> 5

```java
// Vertex类添加一个 visited 属性
private static void dfs(Vertex start) {  
    start.visited = true;  
    System.out.println(start.name);  
    for (Edge edge : start.edges) {  
        if (!edge.linked.visited) {  
            dfs(edge.linked);  
        }  
    }  
}

private static void dfs2(Vertex start) {  
    LinkedList<Vertex> stack = new LinkedList<>();  
    stack.push(start);  
    while (!stack.isEmpty()) {  
        Vertex popped = stack.pop();  
        popped.visited = true;  
        System.out.println(popped.name);  
        for (Edge edge : popped.edges) {  
            if (!edge.linked.visited) {  
                stack.push(edge.linked);  
            }  
        }  
    }  
}
```
## 8.3 BFS
![[dfs.svg]]

按层遍历：1 -> 6, 1 -> 2, 1 -> 3
3 -> 4, 6 -> 5
4 -> 5(已遍历过)

```java
private static void bfs(Vertex start) {  
  
    // bfs 需要队列  
    LinkedList<Vertex> queue = new LinkedList<>();  
    queue.offer(start);  
    start.visited = true;  
    while (!queue.isEmpty()) {  
        Vertex polled = queue.poll();  
        System.out.println(polled.name);  
        for (Edge edge : polled.edges) {  
            if (!edge.linked.visited) {  
                edge.linked.visited = true;  
                queue.offer(edge.linked);  
            }  
        }  
    }  
}
```
## 8.4 拓扑排序
- 对一个 DAG 的顶点进行排序，对每一条有向边(u, v), 顶点 u 的排序都在 v 之前
- 从 DAG 图中选择一个入度为 0 的顶点
- 从图中删除该顶点和以它为起点的有向边
- 重复 12 步，直到 DAG 图为空或者图中不存在无前驱的顶点(存在环)
```java
public static void main(String[] args) {  
  
    Vertex v1 = new Vertex("web基础");  
    Vertex v2 = new Vertex("java基础");  
    Vertex v3 = new Vertex("javaWeb");  
    Vertex v4 = new Vertex("spring");  
    Vertex v5 = new Vertex("微服务");  
    Vertex v6 = new Vertex("database");  
    Vertex v7 = new Vertex("实战");  
  
    v1.edges = List.of(new Edge(v3));  
    v2.edges = List.of(new Edge(v3));  
    v3.edges = List.of(new Edge(v4));  
    v6.edges = List.of(new Edge(v4));  
    v4.edges = List.of(new Edge(v5));  
    v5.edges = List.of(new Edge(v7));  
    v7.edges = List.of();  
  
    List<Vertex> graph = new ArrayList<>(List.of(v1, v2, v3, v4, v5, v6, v7));  
  
    // 找到入度为 0 的顶点加入队列  
    for (Vertex vertex : graph) {  
        for (Edge edge : vertex.edges) {  
            edge.linked.inDegree++;  
        }  
    }  
    LinkedList<Vertex> queue = new LinkedList<>();  
    for (Vertex v : graph) {  
        if (v.inDegree == 0) {  
            queue.offer(v);  
        }  
    }  
    // 从队列移除顶点，每移除一个，相邻顶点入度 -1，减为 0 的入队  
    while (!queue.isEmpty()) {  
        Vertex polled = queue.poll();  
        System.out.println(polled.name);  
        for (Edge edge : polled.edges) {  
            edge.linked.inDegree--;  
            if (edge.linked.inDegree == 0) queue.offer(edge.linked);  
        }  
    }  
}
```
- 检测环：`拓扑排序结果.size != graph.size` 说明存在环
- DFS 实现拓扑排序
- 走过的顶点放在栈中，弹栈的顺序就是拓扑排序的顺序
```java
public static void main(String[] args) {  
  
    Vertex v1 = ...  
    v7.edges = List.of();  
  
    List<Vertex> graph = new ArrayList<>(List.of(v1, v2, v3, v4, v5, v6, v7));  
    LinkedList<String> stack = new LinkedList<>();  
  
    for (Vertex vertex : graph) {  
        dfs(vertex, stack);  
    }  
  
    System.out.println(stack);  
  
}  
  
private static void dfs(Vertex vertex, LinkedList<String> stack) {  
    if (vertex.status == 2) {  
        return;  
    }  
    if (vertex.status == 1) throw new RuntimeException("存在环！");  
    vertex.status = 1;  
    for (Edge edge : vertex.edges) {  
        dfs(edge.linked, stack);  
    }  
    vertex.status = 2;  
    // 相邻顶点处理完后，返回时压栈  
    stack.push(vertex.name);  
}
```
## 8.5 Dijkstra(狄克斯特拉) -- 单源最短路径
![[dfs.svg]]

- 算法流程
	- 所有顶点标记为未访问，组成一个未访问顶点的集合
	- 为每一个顶点分配一个临时距离值
		- 初始顶点设置为 0
		- 其他顶点设置为 ∞
	- 每次选择最小临时距离的未访问顶点作为新的当前顶点
	- 对当前顶点，遍历其所有未访问的邻居，更新临时距离为最小
	- 邻居处理完后，将当前顶点从未访问集合中删除

```java
// Vertex 添加 dist 属性
private static void dijkstra(List<Vertex> graph, Vertex start) {  
    ArrayList<Vertex> unvisited = new ArrayList<>(graph);  
    start.dist = 0;  
  
    while (!unvisited.isEmpty()) {  
        Vertex curr = minDistVertex(unvisited);  
        updateDist(curr, unvisited);  
        unvisited.remove(curr);  
    }  
    for (Vertex vertex : graph) {  
        System.out.println(vertex.name + " " + vertex.dist);  
    }  
}  
  
private static void updateDist(Vertex curr, ArrayList<Vertex> neighbor) { // 只需处理集合内的邻居  
    for (Edge edge : curr.edges) {  
        Vertex v = edge.linked;  
        if (neighbor.contains(v)) {  
            v.dist = Integer.min(v.dist, curr.dist + edge.weight);  
        }  
    }  
}  
  
private static Vertex minDistVertex(ArrayList<Vertex> vertices) {  
    Vertex min = vertices.get(0);  
    for (int i = 1; i < vertices.size(); i++) {  
        if (vertices.get(i).dist < min.dist) {  
            min = vertices.get(i);  
        }  
    }  
    return min;  
}
```
- 改进
- 记录路径
```java
// vertex 添加 prev 属性
private static void updateDist(Vertex curr, ArrayList<Vertex> neighbor) { // 只需处理集合内的邻居  
    for (Edge edge : curr.edges) {  
        Vertex v = edge.linked;  
        if (neighbor.contains(v)) {  
            v.dist = Integer.min(v.dist, curr.dist + edge.weight);  
            v.prev = curr; // update 时保存 prev
        }  
    }  
}

// 也可以在 list.remove 后将 curr.visited 设置为 true
// 只需一个参数
private static void updateDist(Vertex curr) {  
    for (Edge edge : curr.edges) {  
        Vertex v = edge.linked;  
        if (!v.visited) {  
            v.dist = Integer.min(v.dist, curr.dist + edge.weight);  
            v.prev = curr; // update 时保存 prev
        }  
    }  
}
```
- 优先级队列
```java
private static void dijkstra(List<Vertex> graph, Vertex start) {  
    // 创建一个优先级队列(最小堆)  
    PriorityQueue<Vertex> queue = new PriorityQueue<>(Comparator.comparingInt(v -> v.dist));  
    start.dist = 0;  
    for (Vertex vertex : graph) {  
        queue.offer(vertex);  
    }  
    while (!queue.isEmpty()) {  
        Vertex curr = queue.peek();  
        updateDist(curr, queue);  
        queue.poll();  
        curr.visited = true;  
    }  
    for (Vertex vertex : graph) {  
        System.out.println(vertex.name + " " + vertex.dist + " " + (vertex.prev != null ? vertex.prev.name : "null"));  
    }  
}  
  
private static void updateDist(Vertex curr, PriorityQueue<Vertex> queue) { // 只需处理集合内的邻居  
    for (Edge edge : curr.edges) {  
        Vertex v = edge.linked;  
        if (!v.visited) {  
            int dist = curr.dist + edge.weight;  
            if (dist < v.dist) {  
                v.dist = dist;  
                v.prev = curr;  
                queue.offer(v); // 重新加入队列，调整位置  
            }  
        }  
    }  
}
```
## 8.6 Bellman-Ford -- Dijkstra 的改进
![[bellmanford_dijkstra.svg]]

- Dijkstra 算法在遇到负权边时，会出现问题，而 Bellman-Ford 算法可以处理
- 对每条边依次处理，处理顶点数 -1 轮
- 无法处理负环(环上权值和为负)

```java
private static void bellmanFord(List<Vertex> graph, Vertex start) {  
    start.dist = 0;  
    // 顶点个数 - 1 轮处理  
    for (int i = 0; i < graph.size() - 1; i++) {  
        // 处理所有的边  
        for (Vertex s : graph) {  
            for (Edge edge : s.edges) {  
                Vertex e = edge.linked;  
                if (s.dist + edge.weight < e.dist && s.dist != Integer.MAX_VALUE) {  
                    e.dist = s.dist + edge.weight;  
                    e.prev = s;  
                }  
  
            }  
        }  
    }  
    for (Vertex v : graph) {  
        System.out.println(v + " " + (v.prev != null ? v.prev.name : "null"));  
    }  
}
```
## 8.7 Floyd-Warshall -- 多源最短路径
![[floyd_warshall.svg]]
- 可以处理负边，不能处理负环
```
初始化最短路径
k=0, 直接连通
    v1   v2   v3   v4
v1   0    ∞  -2   ∞
v2   4    0    3    ∞
v3   ∞   ∞   0    2
v4   ∞   -1   ∞   0

k=1, 借助 v1 到达其他节点
    v1   v2   v3   v4
v1   0    ∞  -2   ∞
v2   4    0    2    ∞
v3   ∞   ∞   0    2
v4   ∞   -1   ∞   0

k=2, 借助 v2 到达其他节点
    v1   v2   v3   v4
v1   0    ∞  -2   ∞
v2   4    0    2    ∞
v3   ∞   ∞   0    2
v4   3   -1   2     0

k=3, 借助 v3 到达其他节点
    v1   v2   v3   v4
v1   0    ∞  -2    0
v2   4    0    2    4
v3   ∞   ∞   0    2
v4   3   -1    1    0

k=4, 借助 v4 到达其他节点
    v1   v2   v3   v4
v1   0   -1   -2    0
v2   4    0    2    4
v3   5    1    0    2
v4   3   -1    1    0
```

```java
/*
 * v2 -> v1 -> v_j: dist[1][0] + dist[0][j]
 */
static void floydWarshall(List<Vertex> graph) {  
    int size = graph.size();  
    int[][] dist = new int[size][size];  
    Vertex[][] prev = new Vertex[size][size];  
  
    // 初始化  
    for (int i = 0; i < size; i++) {  
        Vertex v = graph.get(i);  
        // 将 list 转为 map 查询效率高  
        Map<Vertex, Integer> map = v.edges.stream().collect(Collectors.toMap(e -> e.linked, e -> e.weight));  
        for (int j = 0; j < size; j++) {  
            Vertex u = graph.get(j);  
            if (u == v) {  
                dist[i][j] = 0; // 同一个顶点  
            } else {  
                dist[i][j] = map.getOrDefault(u, Integer.MAX_VALUE);  
                prev[i][j] = map.get(u) != null ? v : null; // 连通  
            }  
        }  
    }  
  
    // 看能否通过节点到达另一个  
    for (int k = 0; k < size; k++) { // 借助第 k 个顶点  
        for (int i = 0; i < size; i++) {  
            for (int j = 0; j < size; j++) {  
                // dist[i][k] + dist[k][j] 第 i 行借助第 k 个顶点到第 j 列  
                if ((dist[i][k] != Integer.MAX_VALUE  
                        && dist[k][j] != Integer.MAX_VALUE)  
                        && dist[i][j] > dist[i][k] + dist[k][j]) {  
                    dist[i][j] = dist[i][k] + dist[k][j];  
                    prev[i][j] =prev[k][j];  
                }  
            }  
        }  
    }  
}
```
- 负环的情况
![[floyd_warshall2.svg]]
```
k = 0
    v1   v2   v3   v4
v1   0    2    ∞   ∞
v2   ∞   0    -4   ∞
v3   1    3   0    1
v4   ∞   ∞   ∞   0

k = 1
    v1   v2   v3   v4
v1   0    2   -2   ∞
v2   ∞   0   -4   ∞
v3   1    3   -1    1
v4   ∞   ∞   ∞   0
```
- 对角线上出现了负值，说明有负环
## 8.8 最小生成树(MST)
- 在一给定的无向图 $G=(V,E)$ 中, $(u,v)$ 代表连接顶点 $u$ 与顶点 $v$ 的边, 而 $w(u, v)$ 代表此边的权重，若存在 $T$ 为 $E$ 的子集且为无循环图，使得联通所有结点的的 $w(T)$ 最小，则此 $T$ 为 $G$ 的最小生成树
![[mst_prim.svg]]
- Prim 实现
	- 与 Dijkstra 算法类似，将起点的距离设为 0，其他的设为∞
	- 不同之处在于每次都是通过 edge 的权值更新 end 节点的距离，而不是从起点到 end 的距离 + 权值
```java
private static void prim(List<Vertex> graph, Vertex start) {  
    ArrayList<Vertex> unvisited = new ArrayList<>(graph);  
    start.dist = 0;  
  
    while (!unvisited.isEmpty()) {  
        Vertex curr = minDistVertex(unvisited);  
        updateDist(curr, unvisited);  
        unvisited.remove(curr);  
    }  
    for (Vertex vertex : graph) {  
        System.out.println(vertex.name + " " + vertex.dist + " " + (vertex.prev != null ? vertex.prev.name : "null"));  
    }  
}  
  
private static void updateDist(Vertex curr, ArrayList<Vertex> neighbor) { // 只需处理集合内的邻居  
    for (Edge edge : curr.edges) {  
        Vertex v = edge.linked;  
        if (neighbor.contains(v)) {  
            if (edge.weight < v.dist) {  
                // 与 Dijkstra 只有此处不同  
                v.dist = edge.weight;  
                v.prev = curr;  
            }  
        }  
    }  
}  
  
private static Vertex minDistVertex(ArrayList<Vertex> vertices) {  
    Vertex min = vertices.get(0);  
    for (int i = 1; i < vertices.size(); i++) {  
        if (vertices.get(i).dist < min.dist) {  
            min = vertices.get(i);  
        }  
    }  
    return min;  
}
```
- Kruskal 实现
	- 将 edge 按权值排序，从最小的边开始，当两端节点不连通时进行连接
```java
// 优先级队列存放实现 Comparable 的 Edge 对象
private static void kruskal(int size, PriorityQueue<Edge> queue) {  
    List<Edge> res = new ArrayList<>();  
    // 使用并查集判断顶点是否连通
    DisjointSet set = new DisjointSet(size);  
    while (res.size() < size - 1) { // 需要找到 size - 1 条边  
        Edge polled = queue.poll();  
        int i = set.find(polled.start);  
        int j = set.find(polled.end);  
        if (i != j) { // 说明不连通  
            res.add(polled);  
            set.union(i, j); // 标记为连通  
        }  
    }  
    for (Edge e : res) {  
        System.out.println(e);  
    }  
}
```
## 8.9 并查集(union-find disjoint set)
![[disjoint_set.svg]]

```java
public class DisjointSet {  
  
    int[] s;  
  
    public DisjointSet(int size) {  
        s = new int[size];  
        for (int i = 0; i < size; i++) {  
            s[i] = i;  
        }  
    }  
  
    public int find(int x) {  
        // 索引与值相等的是起点  
        if (x == s[x]) {  
            return x;  
        }  
        // 不相等根据终点的值再次查找  
        return find(s[x]);  
    }  
  
    public void union(int x, int y) {  
        // x 的值给 y 处元素  
        s[y] = x;  
    }  
  
    @Override  
    public String toString() {  
        return Arrays.toString(s);  
    }  
  
    public static void main(String[] args) {  
  
        DisjointSet set = new DisjointSet(7);  

		/*
			start is: 0 3
			[0, 1, 2, 0, 4, 5, 6]  0 和 3 的值相同为同一组
		*/
        int x = set.find(0);  
        int y = set.find(3);  
        System.out.println("start is: " + x + " " + y);  
        if (x != y) {  
            set.union(x, y);  
            System.out.println(set);  
        }  

		/*
			start is: 5 6
			[0, 1, 2, 0, 4, 5, 5]
		*/
        x = set.find(5);  
        y = set.find(6);  
        System.out.println("start is: " + x + " " + y);  
        if (x != y) {  
            set.union(x, y);  
            System.out.println(set);  
        }  
  
    }  
}
```
- 路径压缩
![[disjoint_set_problem.svg]]

- 问题：要找到 6 的起点时，需要遍历整个数组
- 更改 find 方法，在递归返回的过程中更新数组元素
```java
// find(6) -> [0, 0, 0, 0, 0, 0, 0]
public int find(int x) {  
    // 索引与值相等的是起点  
    if (x == s[x]) {  
        return x;  
    }  
    // 不相等根据终点的值再次查找  
    return s[x] = find(s[x]);  
}
```
- UnionBySize
![[disjoint_set_problem2.svg]]
- 问题：多连少时，查找 start 节点时效率低
- 更改 union 方法
```java
public void union(int x, int y) {  
	// 少连多
    if (size[x] < size[y]) {  
        s[x] = y;  
        size[y] += size[x]; // 更新 start 元素个数  
    } else {  
        // x 的值给 y 处元素  
        s[y] = x;  
        size[x] += size[y]; // 更新 start 元素个数  
    }  
}

public void union(int x, int y) {  
    if (size[x] < size[y]) {   
        int tmp = x;  
        x = y;  
        y = tmp;  
    }  
    s[x] = y;  
    size[y] += size[x]; // 更新 start 元素个数  
}
```
# 9 Greedy
## 9.1 分数背包问题
- n 个物品都是液体，有重量和价值
- 取走 10L 液体，可以取一部分，求最高价值
```
编号 weight   value
 0     4       24      水
 1     8       160    牛奶
 2     2      4000    五粮液
 3     6      108     可乐
 4     1      4000    茅台
```
- 贪心：每轮取最贵的
	- `4000 + 4000 = 160 * 7/8`
```java
public class FracBackpack {  
  
    static class Item {  
        int index;  
        int weight;  
        int value;  
  
        public Item(int index, int weight, int value) {  
            this.index = index;  
            this.weight = weight;  
            this.value = value;  
        }  
  
        @Override  
        public String toString() {  
            return "Item{" +  
                    "index=" + index +  
                    '}';  
        }  
  
        public int unitValue() {  
            return value / weight;  
        }  
    }  
  
    public static void main(String[] args) {  
  
        Item[] items = new Item[] {  
                new Item(0, 4, 24),  
                new Item(1, 8, 160),  
                new Item(2, 2, 4000),  
                new Item(3, 6, 108),  
                new Item(4, 1, 4000),  
        };  
        select(items, 10);  
    }  
  
    private static void select(Item[] items, int total) {  
  
        Arrays.sort(items, Comparator.comparingInt(Item::unitValue).reversed());  
  
        int maxVal = 0;  
  
        for (Item item : items) {  
            // 全部拿完  
            if (total >= item.weight) {  
                total -= item.weight;  
                maxVal += item.value;  
            } else { // 只能拿一部分  
                maxVal += item.unitValue() * total;  
                break;  
            }  
        }  
        System.out.println(maxVal);  
    }  
}
```
## 0-1 背包问题
- n 个物体都是固体，有重量和价值
- 需要取走不超过 10g 的物品
- 求最大价值
```
编号 weight     value
  0    1      1_000_000     diamond
  1    4         1600       gold
  2    8         2400       ruby
  3    5          30        silver
```
- 贪心可能不会达到最优解
# 10 DP
## 10.1 Fibonacci
- 记忆法改进，将计算结果保存起来，下次继续使用
- 用一维或二维数组保存之前的计算结果
```java
public static int fibonacci(int n) {  
  
    int[] dp = new int[n + 1]; // 缓存结果  
    dp[0] = 0;  
    dp[1] = 1;  
  
    if (n == 0) return 0;  
    if (n == 1) return 1;  
  
    for (int i = 2; i <= n; i++) {  
        dp[i] = dp[i - 1] + dp[i - 2];  
    }  
    return dp[n];  
}

// 改进：只保存前两次计算结果
public static int fibonacci(int n) {  
  
    if (n == 0) return 0;  
    if (n == 1) return 1;  
      
    int a = 0;  
    int b = 1; // 前两次的结果  
  
    for (int i = 2; i <= n; i++) {  
        int c = a + b;  
        a = b;  
        b = c;  
    }  
    return b;  
}
```
## 10.2 BellmanFord
- 开始时其他节点的最短距离设置为∞
- 计算 v1 -> v4 的最短距离
	- v1 -> v2 -> v4
	- v1 -> v3 -> v4
![[bellman_ford.svg]]
- 递推公式 
	- 初始
		- f(v) == 0, v 为起点
		- f(v) == ∞, v 不为起点
	- 递推
		- f(dist) = min(f(dist), f(src) + src.weight)
- 遍历更新
	- f(v4) = min(∞, f(v2) + v2.weight) = 22
	- f(v4) = min(22, f(v3) + v2.weight)) = 20
```java
public class BellmanFord {  
  
    static class Edge {  
        public int src;  
        public int dist;  
        public int weight;  
  
        public Edge(int src, int dist, int weight) {  
            this.weight = weight;  
            this.dist = dist;  
            this.src = src;  
        }  
    }  
  
    public static void main(String[] args) {  
  
        List<Edge> edges = List.of(  
                new Edge(6, 5, 9),  
                new Edge(4, 5, 6),  
                new Edge(1, 6, 14),  
                new Edge(3, 6, 2),  
                new Edge(3, 4, 11),  
                new Edge(2, 4, 15),  
                new Edge(1, 3, 9),  
                new Edge(1, 2, 7)  
        );  
        // 保存从起点出发到每个顶点的最短距离  
        int[] dp = new int[7];  
        dp[1] = 0; // 起点在索引 1 的位置  
        for (int i = 2; i < dp.length; i++) {  
            dp[i] = Integer.MAX_VALUE;  
        }  

		// 初始更新: [0, 0, 7, 9, ∞, ∞, 14]
        for (Edge edge : edges) {  
            if (dp[edge.src] != Integer.MAX_VALUE) {  
                dp[edge.dist] = Integer.min(dp[edge.dist], dp[edge.src] + edge.weight);  
            }  
        }  
  
        for (int i = 0; i < 5; i++) {  
            for (Edge edge : edges) {  
                if (dp[edge.src] != Integer.MAX_VALUE) {  
                    dp[edge.dist] = Integer.min(dp[edge.dist], dp[edge.src] + edge.weight);  
                }  
            }  
        }  
        System.out.println(Arrays.toString(dp));  
    }  
}
```
## 10.3 0-1 背包
- n 个物体都是固体，有重量和价值
- 需要取走不超过 10g 的物品
- 求最大价值
```
编号 weight     value
  1    4         1600       gold
  2    8         2400       ruby
  3    5          30        silver
  4    1      1_000_000     diamond
```

```java
/**
 * 每一列对应当前重量下的最大价值, 行对应物品编号
 * 装不下保持上次最大价值
 * 
 *     0    1    2    3    4    5    6    7    8    9    10
 * 0   0    0    0    0    G    G    G    G    G    G     G
 * 1   0    0    0    0    G    G    G    G    R    R     R
 * 2   0    0    0    0    G    G    G    G    R    R     R
 * 3   0    D    D    D    D   D+G  D+G  D+G  D+G  D+R   D+R 
 * 
*/

/* 
 * 装不下：dp[i][j] = dp[i-1][j] 上一行同列
 * 装得下: 
 *   max(item.value, dp[i-1][j]) 容量已满
 *   max(dp[i][j], dp[i-1][j]) 容量未满, 需要去找剩余容量对应最大价值dp[i-1][j-item.weight]
 *   即：max(dp[i-1][j], item.value+dp[i-1][j-item.weight])
 */

private static int select(Item[] items, int total) {  
    int[][] dp = new int[items.length][total + 1];  
    Item gold = items[0];  
    // 初始化第 0 行  
    for (int j = 0; j < total + 1; j++) {  
        if (j >= gold.weight) {  
            dp[0][j] = gold.value;  
        } else {  
            dp[0][j] = 0;  
        }  
    }  
    // 二维数组 length 返回行数
    for (int i = 1; i < dp.length; i++) {  
        Item item = items[i];  
        for (int j = 0; j < total + 1; j++) {  
            if (j >= item.weight) {  
                dp[i][j] = Integer.max(dp[i - 1][j], item.value + dp[i - 1][j - item.weight]);  
            } else {  
                dp[i][j] = dp[i - 1][j];  
            }  
        }  
    }  
    return dp[dp.length - 1][total];  
}

// 降维
private static int select(Item[] items, int total) {  
    int[] dp = new int[total + 1];  
    Item gold = items[0];  
    // 初始化  
    for (int j = 0; j < total + 1; j++) {  
        if (j >= gold.weight) {  
            dp[j] = gold.value;  
        } else {  
            dp[j] = 0;  
        }  
    }  
    // 从右向左防止覆盖
    for (int i = 1; i < items.length; i++) {  
        Item item = items[i];  
        for (int j = total; j > 0; j--) {  
            if (j >= item.weight) {  
                dp[j] = Integer.max(dp[j], item.value + dp[j - item.weight]);  
            } 
        }  
    }  
    return dp[total];  
}
```
## 10.4 完全背包
- 每件物品的数量不限
```
编号 weight     value
  1    2          3        bronze
  2    3          4        silver
  3    4          7        gold
```

```java
/**
 *      0     1     2     3     4     5     6
 *  1   0     0     1B    1B    2B    2B    3B
 *  2   0     0     1B    1S    2B   1S1B   3B
 *  3   0     0     1B    1S    1G    1G   1G1B
 *  
 */

/*
 * 放得下: max(dp[i-1][j], item.value + dp[i][j-item.weight])
 * 放不下: dp[i][j] = dp[i-1][j]
 */
 
private static int select(Item[] items, int total) {  
  
    int[][] dp = new int[items.length][total + 1];  
    Item item0 = items[0];  
    // 初始化第一行  
    for (int j = 0; j < total + 1; j++) {  
        if(j >= item0.weight) {  
            dp[0][j] = dp[0][j - item0.weight] + item0.value;  
        }  
        // else dp[0][j] = 0; 默认 0  
    }  
    for (int i = 1; i < items.length; i++) { // 第 0 个 item 已经初始化过了  
        Item item = items[i];  
        for (int j = 0; j < total + 1; j++) {  
            if (j >= item.weight) {  
                dp[i][j] = Integer.max(dp[i - 1][j], dp[i][j - item.weight] + item.value);  
            } else {  
                dp[i][j] = dp[i - 1][j];  
            }  
        }  
    }  
    return dp[items.length - 1][total];  
}

// 降维
// 完全背包不用考虑上一行，直接正向遍历即可
private static int selectSimplified(Item[] items, int total) {  
  
        int[] dp = new int[total + 1];  
        Item item0 = items[0];  
        // 初始化第一行  
//        for (int j = 0; j < total + 1; j++) {  
//            if(j >= item0.weight) {  
//                dp[j] = dp[j - item0.weight] + item0.value;  
//            }  
//        }  
        for (Item item : items) {   
            for (int j = 0; j < total + 1; j++) {  
                if (j >= item.weight) {  
                    dp[j] = Integer.max(dp[j], dp[j - item.weight] + item.value);  
                }  
            }  
        }  
        return dp[total];  
    }
```
## 10.5 零钱兑换
- 用最少的硬币凑够总金额
```java
/**
 * 类似完全背包问题
 *      0     1    2      3       4      5
 *  1   0    1*1  2*1    3*1     4*1    5*1 
 *  2   0    1*1  1*2  1*2+1*1   2*2  2*2+1*1
 *  5   0    1*1  1*2  1*2+1*1   2*2    1*5
 * 特殊情况标记
 * 10  flag  flag flag  flag     flag  flag 无法凑齐，返回-1
 */

 /*
  * 装不下：保留上次个数 dp[i][j] = dp[i-1][j]
  * 装得下：min(上次个数，1+余额最小的硬币数)
  *     dp[i][j] = min(dp[i-1][j], 1+dp[i][j-coin.weight])
  */

public int coinChange(int[] coins, int amount) {  
    int[][] dp = new int[coins.length][amount + 1];  
    for (int i = 1; i < amount + 1; i++) {  
        if (i >= coins[0]) {  
            dp[0][i] = dp[0][i - coins[0]] + 1;  
        } else {  
            dp[0][i] = amount + 1;  
        }  
    }  
    for (int i = 1; i < coins.length; i++) {  
        for (int j = 0; j < amount + 1; j++) {  
            if (j >= coins[i]) {  
                dp[i][j] = Integer.min(dp[i - 1][j], dp[i][j - coins[i]] + 1);  
            } else {  
                dp[i][j] = dp[i - 1][j];  
            }  
        }  
    }  
    return dp[coins.length - 1][amount] <= amount ? dp[coins.length - 1][amount] : -1;  
}

// 降维
public int coinChange(int[] coins, int amount) {  
    int[] dp = new int[amount + 1];  
    Arrays.fill(dp, amount + 1);  
    dp[0] = 0;  
  
    for (int coin : coins) {  
        for (int j = coin; j < amount + 1; j++) {  
            dp[j] = Integer.min(dp[j], dp[j - coin] + 1);  
        }  
    }  
    return dp[amount] <= amount ? dp[amount] : -1;  
}
```
## 10.6 LeetCode
### Q62 - 不同路径
- 机器人从左上角走到右下角，每次只能 → 或 ↓，有多少种走法
```
+-------+-----+-----+-----+
| start |  1  |  1  |  1  |
--------+-----+-----+-----+
|   1   |  2  |  3  |  4  |
--------+-----+-----+-----+
|   1   |  3  |  6  | end |
--------+-----+-----+-----+

单行单列特殊情况只有一种方法
(i, j)格的走法 = (i, j - 1)的走法 + (i - 1, j)的走法

```
#### solution
```java
private static int uniquePaths(int m, int n) {  
  
    int[][] dp = new int[m][n];  
    for (int i = 0; i < m; i++) {  
        dp[i][0] = 1;  
    }  
    for (int j = 0; j < n; j++) {  
        dp[0][j] = 1;  
    }  
    for (int i = 1; i < m; i++) {  
        for (int j = 1; j < n; j++) {  
                dp[i][j] = dp[i - 1][j] + dp[i][j - 1];  
        }  
    }  
    return dp[m - 1][n - 1];  
}

// 降维
// 1 1 1 1 1 1 1 每次和左边的数相加进行更新 
// 1 2 3 4 5 6 7 -> 1 3 6 10 15 21 28
private static int uniquePaths(int m, int n) {  
	int[] dp = new int[n];  
	for (int i = 0; i < n; i++) {  
	    dp[i] = 1;  
	}  
	for (int i = 0; i < m; i++) {  
	    dp[0] = 1;  
	    for (int j = 1; j < n; j++) {  
	        dp[j] = dp[j] + dp[j - 1];  
	    }  
	}  
	return dp[n - 1];
}
```
### Q518 - 零钱兑换
- 凑金额有多少种凑法
#### solution
```java
/*
 *      0    1    2        3         4               5
 *  1   1    1    11      111       1111           11111
 *  2   1    1   2,11    12,111  22,211,1111   11111,212,2111
 *  5   1    1   2,11    12,111  22,211,1111  11111,212,2111,5
 */

/*
 * 放不下: dp[i][j] = dp[i-1][j]
 * 放得下: dp[i][j] = dp[i-1][j] + dp[i][j-coin] j=coin时，0列取1
 */

public int coinChange(int[] coins, int amount) {  
  
    int[][] dp = new int[coins.length][amount + 1];  
    for (int i = 0; i < coins.length; i++) {  
        dp[i][0] = 1;  
    }  
    for (int i = 1; i < amount + 1; i++) {  
        if (i >= coins[0]) {  
            dp[0][i] = dp[0][i - coins[0]];  
        }  
    }  
    for (int i = 1; i < coins.length; i++) {  
        for (int j = 1; j < amount + 1; j++) {  
            if (j >= coins[i]) {  
                dp[i][j] = dp[i][j - coins[i]] + dp[i - 1][j];  
            } else {  
                dp[i][j] = dp[i - 1][j];  
            }  
        }  
    }  
    return dp[coins.length - 1][amount];  
}

// 降维
public int coinChange(int[] coins, int amount) {  
  
    int[] dp = new int[amount + 1];  
    dp[0] = 1;  
  
    for (int i = 1; i < amount + 1; i++) {  
        dp[i] = dp[i - coins[0]];  
    }  
    for (int i = 1; i < coins.length; i++) {  
        for (int j = 1; j < amount + 1; j++) {  
            if (j >= coins[i]) {  
                dp[j] = dp[j - coins[i]] + dp[j];  
            }  
        }  
    }  
    return dp[amount];  
}
```


# 11 Divide
# 12 Backtrack