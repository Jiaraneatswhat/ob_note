# 1 基本语法
## 1.1 变量类型
### 1.1.1 基本类型
```python
a = 1
b = 10.0
c = True
d = 4 + 3j

# <class 'int'> <class 'float'> <class 'bool'> <class 'complex'>
print(type(a), type(b), type(c), type(d))

# 运算
a + 1 # 2
b * 2 # 20.0
c + 1 # 2 True 按 1 计算
d ** 2 # (7 + 24j)
```
### 1.1.2 字符串
#### 1.1.2.1 定义
```python
str1 = 'abc'
str2 = 'def'

# 字符串拼接
str1 + str2 # 'abcdef'

# 字符串运算
str1 * 2 # 'abcabc'

# 判断是否包含
'a' in str1 # True
'd' not in str1 # True
```
- 切片
```python
str = '123456789'
str[0: 3] # '123'
str[0: 5: 2] # '135' step = 2
str[0:: 2] # 省略尾
str[-1:: -3] # '963' 倒序索引
```
#### 1.1.2.2 方法
```python
str = 'aabcdef'
str.capitalize() # Aabcdef 不改变原字符串
str.count('aa') # 2
str.endswith('def') # True
str.startswith('aa') # True
str.index('bc') # 2
str.rindex('a') # 5 从右向左查找
str.upper()/lower()
'a-b-c-d'.split('-') # ['a', 'b', 'c', 'd']

```
### 1.1.3 list
#### 1.1.3.1 定义
```python
list1 = [1, 2, 3, 4, 5]
list2 = [1, 'a', True] # 可以存放不同类型的数据
list3 = list1 + list2 # 拼接：[1, 2, 3, 4, 5, 1, 'a', True]
```
- 切片
```python
list3[0: : 2] = [1, 3, 5, 'a', True]
list3[0: 2] = [] # 赋空值 [3, 4, 5, 1,..]
# 'sep'.join(seq) 将 list 中的元素拼接成字符串
" ".join([1, 2, 3]) # 1 2 3
```
#### 1.1.3.2 方法
```python
list1 = [0, 1, 2]
list2 = ['d', 'a', 'c', 'b']

len(list1) # 3
min/max(list1) # 0 2

# 元素操作
list1.append(3) # [0, 1, 2, 3]
list1.remove(1) # 移除指定索引处的元素 [0, 2, 3], 会修改原 list
list1.reverse() # [3, 2, 0]
list1.insert(0, 'd') # ['d', 3, 2, 0]
list1.pop() # 弹出最后一个元素 ['d', 3, 2]
list2.sort() # ['a', 'b', 'c', 'd']

# 通过 del 关键字删除元素
del list1[2] # [0, 1, 3]
```
### 1.1.4 tuple
```python
t1 = (1, 3, 5)
t2 = (7, 'a', True)
t1 + t2 # (1, 3, 5, 7, 'a', True)

t3 = ()
t4 = (1,) # 单元素的元组需要加','

t5 = (['1', 3], 5) # tuple 嵌套 list

t5[0] = 7 # tuple 不支持修改会报错
```
### 1.1.5 set
#### 1.1.5.1 定义
```python
# 空集合用 {} 表示
set0 = {}
set1 = {1, 1, 3} # {1, 3} 自动去重

set2 = {1, 2, 3, 4, 5, 6}
set3 = {1, 3, 5, 7}
set2 - set3 # {2, 4, 6}
set2 | set3 # 并集 {1, 2, 3, 4, 5, 6 ,7}
set2 & set3 # 交集 
set2 ^ set3 # 对称差集 a - b ∪ b - a {2, 4, 6, 7} 
```
#### 1.1.5.2 方法
```python
s = {'a', 'b', 'c'}
```