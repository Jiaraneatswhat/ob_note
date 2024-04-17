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
### 1.1.2 遍历
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
### 1.3.2 遍历
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
### Q83
- 有序链表去重(重复元素保留一个)
- 双指针同时后移，比较值
```java
public Node removeDublicate(Node head) {
	

}
```