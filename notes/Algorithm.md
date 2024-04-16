# 1 LinkedList
- 随机访问的复杂度为 $O(n)$
## 1.1 单向
### 1.1.1 节点类
```java
public class SingleLinkedList {
	Node head;

	class Node {
		int value;
		Node next;
	}
}
```
### 1.1.2 addFirst()
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
### 1.1.3 遍历
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
        }  
    };  
}
```
### 1.1.4 addLast()
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
### 1.1.5 get()
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
### 1.1.6 insert()
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
### 1.1.7 remove()
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
### 1.1.8 with Sentinel
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
## 1.2 双向 with Sentinel
### 1.2.1 节点类
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
### 1.2.2 insert()
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
```
### 1.2.3 remove()
```java
public void remove(int index) {
	Node prev = findNode(index - 1);
	// 违法索引, prev == null, 抛异常
	// prev 为最后一个元素时
	if (prev.next == tail) // 不合法
	prev.next = prev.next.next;
	prev.next.prev = prev;
}
```
