# 1 线性表
- 由同一类型的数据元素构成的有序序列的线性结构
- 线性表中的元素个数就是线性表的长度
## 1.1 顺序表
- 底层采取顺序存储实现
- 插入,删除的平均时间复杂度为 $O(n)$
- 查找的时间复杂度为 $O(1)$
- 顺序表是一种随机存取的存储结构
```c
#include <stdio.h>  
#include <malloc.h>  
  
typedef int E;  
  
struct List  
{  
    // 指向底层数组的指针  
    E * array;  
    int capacity;  
    int size;  
};  
  
typedef struct List* ArrayList;  
  
_Bool init_list(ArrayList list)  
{  
    // 局部变量离开作用域会销毁，不能直接 E arr[10];    
    list->array = malloc(sizeof(E) * 10);  
    if (list->array == NULL) return 0;  
    list->capacity = 10;  
    list->size = 0;  
    return 1;  
}  

_Bool insert_list(ArrayList list, E e, int index)  
{  
    if (index < 0 || index > list->size) return 0;  
    if (list->size == list->capacity) {  
        int new_cap = (list->capacity >> 1) + list->capacity;  
        // new_cap 需要乘元素大小  
        E *new_array = realloc(list->array, new_cap * sizeof(E));  
        if (new_array == NULL) return 0;  
        list->array = new_array;  
        list->capacity = new_cap;  
    }  
    for (int i = list->size; i > index; i--) {  
        list->array[i] = list->array[i - 1];  
    }  
    list->array[index] = e;  
    list->size++;  
    return 1;  
}  
  
_Bool delete_list(ArrayList list, int index)  
{  
    if(index < 0 || index > list->size) return 0;  
    for (int i = index; i < list->size - 1; i++)  
        list->array[i] = list->array[i + 1];  
    list->size--;  
    return 1;  
}  
  
int length(ArrayList list)  
{  
    return list->size;  
}  
  
E * get(ArrayList list, int index)  
{  
    if (index < 0 || index > list->size - 1) return NULL;  
    return &list->array[index];  
}  
  
void print_list(ArrayList list)  
{  
    for (int i = 0; i< list->size; i++)  
        printf("%d  ", list->array[i]);  
    printf("\n");  
}  
  
int find_element(ArrayList list, E e)  
{  
    for (int i = 0; i < list->size; i++)  
    {  
        if (list->array[i] == e)  
        {  
            return i;  
        }  
    }  
    return -1;  
}
```
## 1.2 单向链表
```c
#include <stdio.h>  
#include <malloc.h>  
  
typedef int E;  
  
struct ListNode  
{  
    E value;  
    struct ListNode *next;  
};  
  
typedef struct ListNode * Node;  
  
void init_list(Node node)  
{  
    node->next = NULL;  
}  
  
Node find_node(Node node, int index)  
{  
    for (int i = 0; node != NULL; node = node->next, i++)  
    {  
        if (i == index)  
        {  
            return node;  
        }  
    }  
    return NULL;  
}  
  
E * get_elem(Node node, int index)  
{  
    Node target = find_node(node, index);  
    if (target != NULL)   
        return &target->value;  
    return NULL;  
}  
  
_Bool insert_node(Node node, int value, int index)  
{  
    Node prev = find_node(node, index);  
    Node added = malloc(sizeof(struct ListNode));  
    if (added == NULL) return 0;  
    added->next = prev->next;  
    prev->next = added;  
    added->value = value;  
    return 1;  
}  
  
_Bool remove_node(Node node, int index)  
{  
    if (node == NULL) return 0;  
    Node prev = find_node(node, index);  
    Node tmp = prev->next;  
    prev->next = prev->next->next;  
    free(tmp);  
    return 1;  
}  
  
void print_list(Node node)  
{  
    while (node->next)  
    {  
        // 头结点不存数据，先移动一位  
        node = node->next;  
        printf("%d   ", node->value);  
    }  
}
```
- 插入，查找，删除的复杂度均为 $O(n)$
- 删除最后一个元素的操作与长度有关
## 1.3 双向链表
```c
#include <stdio.h>  
#include <malloc.h>  
  
typedef int E;  
  
struct ListNode  
{  
    E value;  
    struct ListNode * prev;  
    struct ListNode * next;  
};  
  
typedef struct ListNode * Node;  
  
void init_list(Node node)  
{  
    node->prev = node->next = NULL;  
}  
  
_Bool insert_node(Node node, E value, int index)  
{  
    if (index < 0) return 0;  
    while (index--)  
    {  
        node = node->next;  
        if (node == NULL) return 0;  
    }  
    Node added = malloc(sizeof (struct ListNode));  
    if (added == NULL) return 0;  
    added->value = value;  
    if (node->next != NULL)  
    {  
        added->next = node->next;  
        node->next->prev = added;  
    } else added->next = NULL;  
    node->next = added;  
    added->prev = node;  
    return 1;  
}  
  
_Bool remove_node(Node node, int index)  
{  
    while (index--)  
    {  
        node = node->next;  
        if (node == NULL) return 0;  
    }  
    if (node->next == NULL) return 0;  
    Node removed = node->next;  
    if (node->next->next) // 待删除的不是最后一个节点  
    {  
        node->next->next->prev = node;  
        node->next = node->next->next;  
    } else node->next = NULL;  
    free(removed);  
    return 1;  
}
```
## 1.4 循环链表
```c
#include <stdio.h>  
#include <malloc.h>  
  
typedef int E;  
  
struct ListNode  
{  
    E value;  
    struct ListNode* next;  
};  
  
typedef struct ListNode * Node;  
  
void init_list(Node node)  
{  
    node->next = node;  
}  
  
_Bool insert_node(Node node, E value, int index)  
{  
    Node head = node;  
    while (index--)  
    {  
        node = node->next;  
        if (node == head) return 0;  
    }  
    Node added = malloc(sizeof (struct ListNode));  
    if (added == NULL) return 0;  
    added->value = value;  
    if (node->next != head)  
    {  
        added->next = node->next;  
    } else added->next = head;  
    node->next = added;  
    return 1;  
}  
  
_Bool remove_node(Node node, int index)  
{  
    Node head = node;  
    while (index--)  
    {  
        node = node->next;  
        if (node == head) return 0;  
    }  
    if (node->next == NULL) return 0;  
    Node removed = node->next;  
    if (node->next->next)  
    {  
        node->next = node->next->next;  
    } else node->next = head;  
    free(removed);  
    return 1;  
}
```
## 1.5 栈
- 数组实现
```c
#include <stdio.h>
#include <malloc.h>

typedef int E;
struct Stack_base
{
    E * array;
    int capacity;
    int top; // 栈顶元素下标
};
typedef struct Stack_base * Stack;

_Bool init_stack(Stack stack)
{
    stack->array = malloc(sizeof (E) * 10);
    if (stack->array == NULL) return 0;
    stack->capacity = 10;
    stack->top = -1;
    return 1;
}

_Bool push(Stack stack, E e)
{
    if (stack->top + 1 == stack->capacity)
    {
        int new_cap = stack->capacity + (stack->capacity << 1);
        E * new_arr = realloc(stack->array, new_cap * sizeof (E));
        if (new_arr == NULL) return 0;
        stack->array = new_arr;
        stack->capacity = new_cap;
    }
    stack->array[++stack->top] = e;
    return 1;
}

_Bool is_empty(Stack stack)
{
    return stack->top == -1;
}

E pop(Stack stack)
{
    return stack->array[stack->top--];
}

void print_stack(Stack stack)
{
    for (int i = 0; i < stack->top + 1; i++)
        printf("%d  ", stack->array[i]);
    printf("\n");
}
```
- 链表实现
```c
#include <stdio.h>  
#include <malloc.h>  
typedef int E;  
struct ListNode  
{  
    E value;  
    struct ListNode * next;  
};  
  
typedef struct ListNode * Node;  
  
void init_stack(Node node)  
{  
    node->next = NULL;  
}  
  
_Bool push(Node node, E e)  
{  
    // 头插法  
    Node pushed = malloc(sizeof (struct ListNode));  
    if (pushed == NULL) return 0;  
    pushed->value = e;  
    pushed->next = node->next;  
    node->next = pushed;  
    return 1;  
  
}  
  
void print_stack(Node node)  
{  
    do {  
        node = node->next;  
        printf("%d  ", node->value);  
    } while (node->next != NULL);  
}  
  
_Bool is_empty(Node node)  
{  
    return node->next == NULL;  
}  
  
E pop(Node node)  
{  
    Node popped = node->next;  
    E value = popped->value;  
    node->next = node->next->next;  
    free(popped);  
    return value;  
}
```
## 1.6 队列
- 队首和队尾一开始都在 0 的位置
- 顺序表实现
```c
#include <stdio.h>  
#include <malloc.h>  
  
typedef int E;  
  
struct Queue_base  
{  
    E * array;  
    int capacity;  
    int front;  
    int rear;  
};  
  
typedef struct Queue_base * Queue;  
  
_Bool init_queue(Queue queue)  
{  
    queue->array = malloc(sizeof (E) * 10);  
    if (queue->array == NULL) return 0;  
    queue->capacity = 10;  
    queue->front = queue->rear = 0;  
    return 1;  
}  
  
_Bool offer(Queue queue, E e)  
{  
	// 循环数组
    int new_rear = (queue->rear + 1) % queue->capacity;  
    if (new_rear + 1 == queue->front) return 0; // 数组已满  
    queue->array[new_rear] = e;  
    queue->rear = new_rear;  
    return 1;  
}  
  
_Bool is_empty(Queue queue)  
{  
    return queue->front == queue->rear;  
}  
  
E poll(Queue queue)  
{  
    queue->front = (queue->front + 1) % queue->capacity;  
    return queue->array[queue->front];  
}  
  
void print_queue(Queue queue)  
{  
    int i = queue->front;  
    do  
    {  
        i = (i + 1) % queue->capacity;  
        printf("%d  ", queue->array[i]);  
    } while (i != queue->rear);  
}
```
- 链表实现
```c
#include <stdio.h>  
#include <malloc.h>  
  
typedef int E;  
struct Queue_node  
{  
    E e;  
    struct Queue_node * next;  
};  
  
typedef struct Queue_node * Node;  
  
struct Queue_base  
{  
    Node front, rear;  
};  
  
typedef struct Queue_base * Queue;  
  
_Bool init_queue(Queue queue)  
{  
    Node node = malloc(sizeof (struct Queue_node));  
    if (node == NULL) return 0;  
    queue->front = queue->rear = node;  
    return 1;  
}  
  
_Bool offer(Queue queue, E e)  
{  
    Node added = malloc(sizeof (struct Queue_node));  
    if (added == NULL) return 0;  
    added->e = e;  
    queue->rear->next = added;  
    queue->rear = added;  
    added->next = NULL;  
    return 1;  
}  
  
_Bool is_empty(Queue queue)  
{  
    return queue->front == queue->rear;  
}  
  
E poll(Queue queue)  
{  
    Node polled = queue->front->next;  
    E e = polled->e;  
    queue->front->next = queue->front->next->next;  
    if (queue->rear == polled) queue->rear = queue->front; // 如果队尾是待出队的节点，将队尾移至队首  
    free(polled);  
    return e;  
}  
  
void print_queue(Queue queue)  
{  
    Node node = queue->front->next;  
    while (node)  
    {  
        printf("%d  ", node->e);  
        node = node->next;  
    }  
}
```
# 2 树
- 每个结点连接的子结点数目称为该结点的度，最大的度称为树的度
## 2.1 二叉树
### 2.1.1 基本概念
- 所有分支结点都有左右子树，且叶子节点都在同一层，称为满二叉树
- 每个结点的度都是 2

![[full_bin_tree.svg]]
- 只有最后一层有空缺，所有叶子结点都是按照由左往右的顺序排列的，称为完全二叉树

![[complete_bin_tree.svg]]
- 满二叉树一定是完全二叉树
### 2.1.2 树之间的转换
- 树转二叉树

![[tree2bin1.svg]]
- 将兄弟节点间相连

![[tree2bin2.svg]]

- 删除右子节点相连的线

![[tree2bin3.svg]]

- 将添加的线作为右节点的连线

![[tree2bin4.svg]]
- 森林转换二叉树
- 先将每棵树转化为二叉树
- 从左向右依次将 i+1 棵树的根结点作为 i 棵树根节点的右孩子

![[4rest2bin.svg]]
### 2.1.3 二叉树的性质
- 对于一棵二叉树，第 $i$ 层的最大节点数为 $2^{i-1}$ 个
- 对于一棵深度为 $k$ 的二叉树，最大节点数量为 $\sum_{i=0}^{k-1}2^{i}$
	- 用等比数列求和转为 $2^k-1$
	- 结点的边数为 $2^k-2$
- 假设一棵二叉树中度为 i 的节点个数为 $n_i$，则其结点总数为 $n=n_0+n_1+n_2$
	- 度为 0 的结点边数为 0，可得边数总和为 $n_1+2n_2$
	- 结合性质二中 `最大节点数 - 1 = 边数` 可推得最大节点数 $n = n_1+2n_2+1$ 
- 完全二叉树的结点数 n 满足 $2^{k-1}-1<n\le2^{k-1}$
	- $n$ 必为整数，可改写为 $2^{k-1}\le n\le2^{k-1}$
	- 对左边取对数，有 $k-1\le log_2n$
	- 则一棵具有 $n$ 个结点的完全二叉树深度为 $k=\left \lfloor log_2n \right \rfloor +1$
- 一个有 $n$ 个结点的完全二叉树，对于任意一个结点 $i$，从上到下，从左到右：
	- 左孩子为 $2i$, 右孩子为 $2i+1$, 超出 $n$ 则说明不存在对应的子结点
	- $i=1$ 的是根节点，父节点为 $\left \lfloor i/2 \right \rfloor$
### 2.1.4 性质相关练习
- 给定 N 个结点，能构造多少种二叉树
	- 有 0 个或 1 个结点，$h(0)=h(1)=1$
	- 有 2 个结点，$h(2)=h(1)\times h(0) + h(0) \times h(1) = 2$
	- 有 3 个结点，$h(3)=h(2) \times h(0) \times 2 + h(1) \times h(1) = 5$
```c
int dp[size + 1];
dp[0] = dp[1] = 1;
for (int i = 2; i <= size; i++)
{
	dp[i] = 0;
	for (int j = 0; j < i; j++)
		dp[i] += dp[i - j - 1] * dp[j]
}
```
- 通项式：
$$
C_{n} =\frac{1}{n+1}C_{2n}^{n}=\frac{1}{n+1}\times \frac{(2n)!}{n!\times (2n-n)!}=\frac{(2n)!}{n!\times (n+1)!}   
$$
- 一棵完全二叉树有 $1001$ 个结点，求叶子结点的个数
	- 第九层的叶子节点 + 第十层全部结点
	- 深度为 $k=\left \lfloor log_2n \right \rfloor +1 = 1001 = 10$ 
	- 前九层的结点数 $n=2^9-1=511$
	- 第十层有 $1001-511=490$ 个结点
	- 第九层可能有度为 $1$ 的结点
		- $(490 + 1) / 2 = 245$
		- 多 + 1 用于排除度为 $1$ 的结点
	- $256-245+490=501$
- 深度为 $h$ 的满 $m$ 叉树的第 $k$ 层有多少个结点 $m^{k-1}$,与 $h$ 无关
- 一棵有 $1025$ 个结点的二叉树层数 $k$ 的取值范围
	- 直接排成一条线的深度最大, $1025$
	- 完全二叉树深度最小，取 $log$ $+ 1$ 得 $11$
### 2.1.5 构建
```c
typedef char E;  
  
typedef struct TreeNode  
{  
    E e;  
    struct TreeNode * left;  
    struct TreeNode * right;  
} * Node;  
  
int main(void)  
{  
    Node a = malloc(sizeof(struct TreeNode));  
    Node b = malloc(sizeof(struct TreeNode));  
    Node c = malloc(sizeof(struct TreeNode));  
    Node d = malloc(sizeof(struct TreeNode));  
    Node e = malloc(sizeof(struct TreeNode));  
  
    a->e = 'A';  
    b->e = 'B';  
    c->e = 'C';  
    d->e = 'D';  
    e->e = 'E';  
  
    a->left = b;  
    a->right = c;  
    b->left = d;  
    b->right = e;  
    // 叶子节点置为 NULL    c->left = c->right =NULL;  
    d->left = d->right =NULL;  
    e->left = e->right =NULL;  
  
    printf("%c\n", a->left->left->e);  
    printf("%c\n", a->left->right->e);  
  
}
```
### 2.1.6 遍历
- 前序遍历
```c
// 递归
void pre_order(Node node)
{
	if (node == NULL) return;
	pre_order(node->left);
	pre_order(node->right);
}

// 非递归
// 每走一步，输出结点的值，入栈
// 走到头出栈，有右子树的话重复上述步骤

void pre_order_traverse(Node node)  
{  
    struct StackNode head;  
    init_stack(&head);  
    while (node || !is_empty(&head))  
    {  
        // pop 右子树为空后跳过下面的循环继续 pop        
        while (node) {  
            printf("%c", node->e);  
            push(&head, node);  
            node = node->left;  
        }  
        Node popped = pop(&head);  
        node = popped->right;  
    }  
}
```
- 中序遍历
```c
void in_order_traverse(Node node)
{
	struct StackNode head;  
    init_stack(&head);  
    while (node || !is_empty(&head))  
    {  
        // 处理完左子树        
        while (node) {  
            push(&head, node);  
            node = node->left;  
        }  
        Node popped = pop(&head);  
        printf("%c", popped->e);  
        node = popped->right;  
    }  
}
```
- 后序遍历
```c
// 在 Node 中增加一个标志位
typedef struct TreeNode  
{  
    E e;  
    struct TreeNode * left;  
    struct TreeNode * right;  
    int flag; // 左右子树都需要遍历，0 表示左子树遍历完成，1 表示右子树遍历完成  
} * Node;

void post_order_traverse(Node node)  
{  
    struct StackNode head;  
    init_stack(&head);  
    while (node || !is_empty(&head))  
    {  
        while (node) {  
            push(&head, node);  
            node->flag = 0;  
            node = node->left;  
        }  
        // 不出栈，只进行 peek        
        node = peek(&head);  
        if (node->flag == 0) {  
            node->flag = 1;  
            node = node->right;  
        } else  
        // 右子树已经遍历完, 出栈  
        {  
            printf("%c", node->e);  
            pop(&head);  
            node = NULL; // 跳过下一轮 while，直接去处理栈中元素  
        }  
    }  
}
```
- 层序遍历
	- 将根节点存入队列中
	- 循环执行直到队列为空：
		- 出队一个元素并打印
		- 左右结点入队
```c
void level_order_traverse(Node node)  
{  
    struct Queue_base queue;  
    init_queue(&queue);  
    offer(&queue, node);  
    while (!is_empty(&queue))  
    {  
        Node polled = poll(&queue);  
        printf("%c ", polled->e);  
        if(polled->left)  
            offer(&queue, polled->left);  
        if(polled->right)  
            offer(&queue, polled->right);  
    }  
}
```
### 2.1.7 遍历相关练习
- 有一棵二叉树前序遍历结果为 `ABCDE`，中序遍历结果为 `BADCE`，后序遍历结果为_____
	- 前序遍历每个根节点在前
	- 前序遍历得出 `A` 是根节点，`D, E` 是 `C` 的孩子
	- 中序遍历得出 `B` 是 `A` 的左子树，`C, D, E` 在 `A` 的右边
	- 则后序结果是 `BDECA`

![[tree_traverse.svg]]

- 对二叉树结点从 1 开始连续编号，要求每个结点的编号大于左右孩子结点的编号，应采用哪种遍历方式 -- 后序遍历
## 2.2 线索化二叉树
