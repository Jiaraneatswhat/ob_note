# 1.B-Tree
- B- 树和 B+ 树的节点一般会映射磁盘文件，大小为磁盘页(4kb)的整数倍
- 多路可以降低树的高度，从而减少磁盘 IO
- 2-3 树是一种特殊的 B-树，它满足：
	- `2-节点`有 {0, 2} 个子节点
	- `3-节点`有 {0, 3} 个子节点
	- 所有叶子节点都在同一层
## 1.1 B-树的性质

- B 树非叶子节点的结构

![[b_tree_node.svg]]
- $n$: key 的个数
- $p$：指针
- $k$：key
- $p_0$ 至 $p_n$ 从小到大排列用于划分区间
- $k$ 的个数比 $p$ 的个数少一个

- $m$ 阶 B -树的性质
	- 每个节点最多有 $m$ 个子节点，至多含有 $m-1$ 个`key`
	- 除根节点外，每个非叶子节点至少有 $ceil(\frac{m}{2})$ 个子节点
	- 根节点有 0 或者至少 2 个子节点
	- 叶子节点都在同一层
- B-树将文件以 <font color='red'>Page</font> 为单位进行存储，每个 Page 作为 B-树的一个节点

![[b_tree_page.svg]]

B-树每个节点都存放着索引和数据，搜索可能在非叶子节点结束，最好的情况是 $O(1)$
## 1.2 B-树的删除
### 1.2.1 要删除的键在叶子节点中
#### 1.2.1.1 不超过子结点 key 下限

![[b_tree_del1.svg]]

- 此时删除 `31` 没有影响(3 阶要求非叶子节点且非根节点有 $ceil(3/2)=2$ 个结点)，直接删除
#### 1.2.1.2 超过子结点 key 下限

![[b_tree_del2.svg]]

- 此时不能直接删除 `32`
- 对树结构做调整：从左至右向兄弟节点借用 `key`，借到 `28`
	- `28` 上移，`30` 下移后删除 `32`

![[b_tree_del3.svg]]

#### 1.2.1.3 可以合并的情况

![[b_tree_del4.svg]]

- 要删除 `30` 时，左右兄弟节点都不能借节点时，需要进行合并
	- `28` 下移与 `25` 合并后删除 `30`

![[b_tree_del5.svg]]

### 1.2.2 要删除的键在内部节点中

![[b_tree_del6.svg]]

- 后继 `key` 替换掉待删除的 `key` 后，在叶子结点中删除 `key`
	- `35` 替代 `33`，删除 `33`
- 删除后，按删除叶子节点的情况进行调整
	- `32` 上移，`35` 下移

![[b_tree_del7.svg]]

# 2.B+Tree
## 2.1 B+树的性质
- $m$ 阶 B+树满足
	- 每个节点最多有 $m$ 个子节点
	- 每个非叶子节点至少有 $ceil(\frac{m}{2})$ 个 `key`
	- 子结点个数与 key 的个数相同
	- 叶子节点存有相邻叶子节点的指针，由小到大顺序连接
	- 内部节点只保存索引，不保存数据，数据全部在叶子节点中
## 2.2 B+树的插入

- 不超过 4 个节点时直接插入

![[b+insert1.svg]]

- 后续再插入时需要分裂

![[b+insert2.svg]]

- 前 $ceil(\frac{m}{2})=2$ 个记录放在左子节点，剩下的放在右子节点，第 $ceil(\frac{m}{2})+1$ 个记录的 `key` 进位至父节点(索引节点)，与之类似，当索引节点达到 `5` 个，需要进行分裂时，进位后的索引节点在子节点中不会保留

![[b+insert3.svg]]

## 2.3 B+树的删除

![[b+del1.svg]]

- 要删除的 `key` 是索引节点的 `key`，向兄弟节点借 `key` 后替换索引节点并删除

![[b+del2.svg]]

- 要删除的 `key` 是索引节点的 `key`，向兄弟节点借不到 `key` 时
- 合并后删除父节点中的 `key`
- B+ 树删除后索引节点中的 `key` 在叶子节点中不一定存在对应的记录
## 2.4 MySQL 中的 B+树索引
- MySQL 中索引可以分为两类
	- 主键索引(聚簇索引)，对应 `B+` 树的叶子节点存放的是实际的数据记录
	- 二级索引(非聚簇索引或辅助索引)，对应 `B+` 树的叶子节点只存放主键值和索引值
	- 如果通过非聚簇索引查找一条完整的数据记录，需要先找到非聚簇索引的叶子节点获取主键值，再去主键索引的 `B+` 树查询数据，这个过程也叫<font color='red'>回表</font>
	- 当查询的数据是主键值时，通过非聚簇索引一次就能查找到数据，被称为<font color='red'>索引覆盖</font>
- 联合索引指将多个字段组合成一个索引
	- 遵循最左匹配原则，越靠前的字段被用于索引过滤的概率越高，在建立时要把区分度大的字段排在前面
# 3.红黑树
- 高效的自平衡树结构
## 3.1 红黑树的规则
- 根节点为黑色
- 每个叶子节点都是黑色的空节点(Nil)
- 从根节点到叶子节点不会出现连续的红色节点
- 从任意一个节点出发到其子节点的所有路径都包含相同个数的黑色节点
## 3.2 2-3-4 树节点和红黑树的对应关系

![[rbtree_to_234_tree.svg]]
## 3.3 红黑树的插入
### 3.3.1 第一次插入为根节点，染黑
### 3.3.2 第二次插入的为红色节点

![[rbtree_insert1.svg]]

### 3.3.3 插入 3 个节点时，有 4 种需要调整的情况

![[rbtree_insert2.svg]]
- 针对(3)(4)的情况，父节点进行旋转得到(1)(2)
- 之后祖父节点再进行旋转，变色

![[rbtree_insert3.svg]]

### 3.3.4 插入第 4 个节点
#### 3.3.4.1 需要变色的情况

![[rbtree_insert4.drawio.svg]]
#### 3.3.4.2 不需要变色的情况
- 3.3.4.1 中变色后的情况，不需要做操作
### 3.3.5 TreeMap 中的插入
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
## 3.4 红黑树的旋转
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
## 3.5 红黑树的删除
### 3.5.1 删除规则
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
### 3.5.2 红黑树删除和 2-3-4 树的对应

![[rbtree_del_rb_to_234.svg]]

- 删除红黑树的内部节点，可以转换为删除叶子节点或是叶子节点的父节点
- 而叶子节点或叶子节点的父节点，对应的是 2-3-4 树的叶子节点
- 2-3-4 树删除 `3-节点`中一个元素不需要调整
	- 例如从 `0，1` 中删除红色的 `0`
- 2-3-4 树删除 `4-节点`中的一个元素不需要调整
	- 例如从 `10，11，12` 中删除 `10` 或 `12`
- 2-3-4 树删除 2-节点需要调整
	- 例如删除 `3`，会违反 `2-节点`子节点的要求
	- 此时需要向兄弟节点借节点
- 因此删除情况有
	- 不用借节点
		- 删除 `3-节点` 中的红色节点或 `4-节点` 中的红色节点：直接删除
		- 删除 `3-节点` 中的黑色节点或 `4-节点` 中的黑色节点
			- 用子节点替换父节点，变黑(2-3-4 树 `2-节点`是黑色)
		- 叶子节点直接删除
	- 需要借节点（黑色的叶子节点）
		- 兄弟节点是 `3-节点` 或 `4-节点`，父节点下移，兄弟节点上移后删除
		- 兄弟节点也是 `2-节点`
### 3.5.3 删除后的调整

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
