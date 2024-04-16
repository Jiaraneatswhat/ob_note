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