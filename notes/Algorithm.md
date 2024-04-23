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

# Graph
## 基本知识
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
## DFS
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
## BFS
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
## 拓扑排序
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

```
# Greedy
### 分数背包问题
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
### 0-1 背包问题
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
# DP
## .1 Fibonacci
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
## .2 BellmanFord
- 开始时其他节点的最短距离设置为∞
- 计算 v1 -> v4 的最短距离
	- v1 -> v2 -> v4
	- v1 -> v3 -> v4
![[BellmanFord.svg]]
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
## .3 0-1 背包
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
## .4 完全背包
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
## .5 零钱兑换
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
## LeetCode
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