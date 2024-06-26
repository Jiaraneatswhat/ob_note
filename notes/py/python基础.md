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
s = {'a', 'b', 'c', 'd'}
s.add('e')
s.remove('a')
s.discard('f') # 丢弃不存在的元素不报错
s.clear()

s1 = {1, 3, 5, 7}
s2 = {2, 4, 6, 8}
s1.difference(s2) # 差集
s1.intersection(s2) # 交集
s1.union(s2) # 并集
s1.update(s2) # 将 s2 中的元素加到 s1 中
```
### 1.1.6 字典
#### 1.1.6.1 定义
```python
dict0 = {} # 空字典
dict1 = {'a': 1, 'b': 2, 'c': 3}
dict2 = {'a': 1, 'a': 2, 'c': 3} # {'a': 2, 'c': 3} 自动覆盖重复的 v
# 通过赋值添加元素
dict1['d'] = 4 # {'a': 1, 'b': 2, 'c': 3, 'd': 4}
```
#### 1.1.6.2 方法
```python
dict1 = {'a': 1, 'b': 2, 'c': 3, 'd': None}
dict2 = {'1': 'a', '2': 'b', '3': 'c'}

len(dict1) # 4
dict1.get('a') # 通过 k 获取 v
dict1.items() # 返回 dict_items 对象: dict_items([('a', 1), ...])
dict1.keys() # 返回 dict_keys 对象: dict_keys(['a', ...])
dict1.update(dict2) # 将 dict2 中的 kv 对添加到 dict1 中

dict1.pop('a') # 根据 k 删除 kv 对
dict1.popitem() # 删除最后一个 kv 对
```
# 2 流程控制，推导式，迭代器与生成器
## 2.1 流程控制
- if 语句
```python
if ...:
	...
elif ...:
	...
else:
	...
```
- while
```python
while ...:
	...

# while 也可以搭配 else，在退出循环后执行:
while ...:
	...
else:
	...
```
- for
```python
for _ in range(start, end, step)
	...
else:
	...
```
## 2.2 推导式
### 2.2.1 list
```python
list1 = ['a', 'b', 'c', 'd', 'e']
list2 = [e for e in list1] # ['a', 'b', 'c', 'd', 'e']

list3 = [i for i in range(5)] # [0, 1, 2, 3, 4]

# 搭配 if-else
list4 = [i for i in range(10) if i % 2 == 0] # [0, 2, 4, 6, 8]
list5 = [i for i in range(10) if i % 2 == 0 else i * 2]
```
### 2.2.2 set
```python
set1 = {i for i in (1, 3, 5, 7)}
set2 = {i for i in range(5) if i % 2 == 0 else 2 * i}
```
### 2.2.3 tuple
```python
# 直接用 '()' 返回的是生成器
tuple1 = (i for i in range(5))
type(tuple1) # <generator object <genexpr> at ...>

# 需要调 tuple 的构造器
class tuple(Sequence[_T_co]): # Sequence 是容器类的抽象基类
    ...

tuple(tuple1) # (0, 1, 2, 3, 4)
```
## 2.3 迭代器
```python
# builtins.py 中定义了获取迭代器的方法:
@overload
def iter(__object: SupportsIter[_SupportsNextT]) -> _SupportsNextT: ...
@overload
def iter(__object: _GetItemIterable[_T]) -> Iterator[_T]: ...
@overload
def iter(__object: Callable[[], _T | None], __sentinel: None) -> Iterator[_T]: ...
@overload
def iter(__object: Callable[[], _T], __sentinel: object) -> Iterator[_T]: ...

# 通过 list 获取一个迭代器
itr = iter([1, 3, 5, 7]) # 返回一个 list_iterator
next(itr) # 通过 next 迭代，迭代结束时抛出 StopIteration 异常

# next(iterator[, default]) 也可以传入一个 default，在迭代结束时返回
```
- 创建自己的迭代器
```python
class MyNumbers:
	# 需要实现 __iter__() 和 __next__()
    def __iter__(self):
	    # 初始值
	    self.value = 1
	    return self
	def __next__(self):
		x = self.value
		self.value += 1
	    return x

my_class = MyNumbers()
itr = iter(my_class)
[next(itr) for i in range(10)]
```
## 2.4 生成器
```python
# 使用 yield 的函数称为生成器，返回值是一个迭代器
def fibonacci(n): 
    a, b, cnt = 0, 1, 0
    while True:
        if (cnt > n):
            return
        # 遇到 yield 时暂停执行，将 yield 后的表达式返回
        yield a
        # 调用 next() 或 for 循环时，从暂停的位置执行到下一个 yield
        a, b = b, a + b
        cnt += 1
tuple(fibonacci(10))
```
# 3 函数
- 传入不可变类型是值传递，可变类型是引用传递
```python
def add_num(lst):
	lst.append([1, 1])
	print(id(lst)) # id 返回对象地址
	return lst

lst = [1, 2, 3]
add_num(lst)
print(id(lst)) # [[1, 2, 3, [1, 1]]] 地址与 add_num() 中相同
```
- 默认参数
```python
def sum(a, b = 10):
	...
```
- 不定长参数
```python
# 加了 * 的参数用 tuple 存储
def print_nums(a, *b):
	print(a, b)

print_nums(1, 2) # 1 (2,)
print_nums(1, 5, 7) # 1 (5, 7)

# 加了 ** 的参数用 dict 存储
def print_dict(a, **b):
	print(a, b)

print_dict(1) # 1
# 传入时用 ‘key=value’
print_dict(1, k1='v1', a=1) # 1 {'k1': 'v1', 'a': '1'}
```
- lambda 函数
```python
f = lambda a, b: a ** b
f(2, 3) # 8
```
# 4 OOP
- 空参构造
```python
# 定义一个 Person 类：
class Person:
	name = 'bob'
	gender = 'male'
	age = 24
	# 方法中第一个参数必须为 self(或其他)，类似 this
	def print_person(self):
		print(self.name, self.gender, self.age)
		
# 自动调用空参方法 __init__()
p = Person()
print_person(p)
```
- 显式构造
```python
class Person:
	name = ''
	gender = ''
	age = 0
	def __init__(self, name, gender. age):
		self.name = name
		...		
```
- 继承
```python
class Student:
	score = 0

	def __init__(self, name, gender, age ,score):
		# 在构造器第一行调父类构造实现继承
		Person.__init__(self, name, gender, age)
		self.score = score
	# 可以重写父类方法
	def print_person(self)：
		...
```
- 私有属性和方法
```python
class Site:
	def __init__(self, name, url):
		self.name = name
		self.__url = url # __表示私有属性
	def who(self):
		print(self.name)
		print(self.__url)
	def __foo(self):
		print('私有方法')
	def foo(self):
		print('公共方法')

s = Site('百度', 'www.baidu.com')
s.who() # 百度 www.baidu.com
s.foo() # 公共方法
s.__foo() # 私有方法报错

s._Site__foo() # 名称重整调用私有方法
```
- 运算符重载
```python
class Person:
	age = 0
	def __init__(self, age):
		self.age = age

	def __add__(self, p) # 年龄求和
		return Person(self.age + p.age)
Person(10) + Person(20) # age = 30
```
# 5 NumPy
## 5.1 创建 ndarray
```python
# def array(...) -> ndarray: ...
np.array([1, 2, 3])
# 指定元素类型:
np.array([1, 2, 3], dtype=np.int64)

# def zeros(...) -> ndarray: ...
np.zeros(shape) # 创建 shape 形的全 0 array 
# array([0., 0., ...])
# 创建 shape 与元素类型均与 a 相同的 ndarray
np.zeros_like(a) 

# 全 1 array
# def ones(shape, ...) -> ndarray: ...
np.ones(shape)
np.ones_like(arr)

# def empty(...) -> ndarray: ...
np.empty(shape)
np.empty_like(arr) # 创建随机数占位的 ndarray

# 通过 range 创建 ndarray
# def arange(stop, dtype=..., *, like=...): ...
# def arange(start, stop, step=..., dtype=..., *, like=...): ...
np.arange(5) # array([0, 1, 2, 3, 4])
np.arane(2, 9, 2) # array([2, 4, 6, 8])

# def linspace(start, stop, num=50, ...)
# 将区间分为 num 份
np.linspace(0, 10, num=5) # array([ 0. , 2.5, 5. , 7.5, 10. ])
```
## 5.2 元素操作
```python
arr = np.array([2, 1, 5, 3, 7])

"""
	def sort(a, axis=-1, kind=None, order=None)
		axis： 默认 -1, 表示最后一个 axis
		kind：排序算法，默认快排
	默认升序排序，返回一个 copy
"""
np.sort(arr) # array([1, 2, 3, 5, 7])

# def argsort(a, axis=-1, kind=None, order=None)
# 返回索引 array：
np.argsort([5, 3, 9]) # [3, 5, 9] 对应索引 [1, 0, 2]

# 指定 axis 进行排序
x = np.array([[0, 3], [2, 2]])
np.argsort(x, axis=0) # 垂直方向
# array([[0, 1],
#      [1, 0]])

np.argsort(x, axis=1) # 水平方向
# array([[0, 1],
#      [0, 1]])

# 拼接两个 array
x = np.array([[1, 2], [3, 4]])
y = np.array([[5, 6]])
np.concatenate((x, y), axis=0) # 竖直拼接
```
## 5.3 shape 相关
```python
arr = ([[[0, 1, 2, 3],
		[4, 5, 6, 7]],
		
		[[0, 1, 2, 3],
		[4, 5, 6, 7]],
		
		[[0, 1, 2, 3],
		[4, 5, 6, 7]]])

"""
class ndarray(_ArrayOrScalarCommon, Generic[_ShapeType, _DType_co]):
	@property
    def ndim(self) -> int: ...
    @property
    def size(self) -> int: ...
	@property
    def shape(self) -> _Shape: ...
    _Shape定义在_shape.py中： _Shape = Tuple[int, ...]
"""

arr.ndim # 3
arr.size # 元素个数 24
arr.shape # (3, 2, 4)

# reshape
a = np.arange(6) # arrat([0, 1, 2, 3, 4, 5])
# ndarray 中的 reshape()
"""
@overload  
class ndarray:
	def reshape(self, shape: _ShapeLike, /, *, 
	order: _OrderACF = ...  ) -> ndarray[Any, _DType_co]: ...
"""
b = a.reshape(3, 2)

# numpy 中的 reshape()
# def reshape(a, newshape, order='C')
np.reshape(a, shape)

# 1d array 转 2d array
# 1d array
a = np.array([1, 2, 3])
a.shape # (3,)
b = np.array([[1, 2, 3]])
b.shape # (1, 3)

# 1d array 扩展为行向量
a[np.newaxis, :]
a[:, np.newaxis] # 列向量 (3, 1)

# 使用 expand_dims() 扩展维度
# def expand_dims(a, axis)
np.expand(a, axis=1) # (3, 1)
np.expand(a, axis=0) # (1, 3)
```
## 5.4 索引和切片
```python
# 基本的索引切片与 python 相同
a = np.array([[1, 2, 3, 4], [5, 6, 7, 8], [9, 10, 11, 12]])

# 通过表达式筛选元素
a[a < 5] # [1, 2, 3, 4]
a[a%2==0] # [2, 4, 6, 8, 10, 12]

```
## 5.5 stack & split
```python
a1 = np.array([[1, 1],
			[2, 2]])

a2 = np.array([[3, 3],
			[4, 4]])

# def vstack(tup)
np.vstack((a1, a2))
# array([[1, 1],
#       [2, 2],
#       [3, 3],
#       [4, 4]])

np.hstack((a1, a2))
#  array([[1, 1, 3, 3],
#        [2, 2, 4, 4]])

# def hsplit(ary, indices_or_sections)
x = np.arange(16.0).reshape(4, 4)

np.hsplit(x, 2) # 将 array 划分为 [:2], [2:] 两部分
# [array([[  0.,   1.],
#        [  4.,   5.],
#        [  8.,   9.],
#        [12.,  13.]]),
#  array([[  2.,   3.],
#        [  6.,   7.],
#        [10.,  11.],
#        [14.,  15.]])]

np.hsplit(x, np.array([3, 6])) # 分为 [:3], [3: 6], [6:] 三部分
#[array([[ 0.,   1.,   2.],
#        [ 4.,   5.,   6.],
#        [ 8.,   9.,  10.],
#        [12.,  13.,  14.]]),
#  array([[ 3.],
#        [ 7.],
#        [11.],
#        [15.]]),
#  array([], shape=(4, 0), dtype=float64)] 不含元素的空 array

# np.vsplit() 类似
```
## 5.6 array 的运算
```python
arr1 = np.array([1, 2])
arr2 = np.ones(2)
arr1 + arr2 # array([2, 3])
arr1 / arr1 # array([1., 1.])
arr1.sum() # 3

arr3 = np.array([[1, 2], [3, 4]])
arr3.sum(axis=0) # array([4, 6])
arr3.sum(axis=1) # array([3, 7])

# 广播：array 和单个数运算
data = np.array([1.0, 2.0])
# 将 1.6 扩展到 data 的大小再计算
data * 1.6 # array([1.6, 3.2])
```
## 5.7 矩阵运算
```python
# def dot(a, b, out=None), 用于 1d-array 的点积
# 也可以计算多维矩阵的乘积
a = np.array([1, 3])
b = np.array([2, 4])
np.dot(a, b) # 14

# 两个 2d-array 通过 matmul() 或 a @ b 计算
x = np.array([[1, 2, 3], [4, 5, 6]])
y = np.array([[6, 23], [-1, 7], [8, 9]])
np.matmul(x, y) # 等价于 x @ y

# multiply() 计算对应位置的乘积
np.multiply(a, b) # array([2, 12])
# 大小不一样时，将小的进行扩充
a = np.array ([[1, 2, 3],[4, 5, 6]])
b = np.array([1, 2, 3])
np.multiply(a, b) # array([1, 4, 9], [4, 10, 18])

# 矩阵的转置
arr = np.arange(6).reshape((2, 3))
"""
class ndarray(_ArrayOrScalarCommon, Generic[_ShapeType, _DType_co]):
	@overload
	def transpose(self: _ArraySelf, axes: None | _ShapeLike, /) -> _ArraySelf: ...

class _ArrayOrScalarCommon:
	@property  
	def T(self: _ArraySelf) -> _ArraySelf: ...
"""
arr.transpose()
arr.T
```
# 6. Pandas
## 6.1 数据类型
### 6.1.1 Series
```python
# Series
class Series(base.IndexOpsMixin, NDFrame):
	def __init__(
        self,
        data=None,
        index=None,
        dtype: Dtype | None = None,
        name=None,
        copy: bool = False,
        fastpath: bool = False,
    ): ...
    # data : array-like, Iterable, dict, or scalar value
    # index : array-like or Index (1d)

# 默认索引从 0 开始
s1 = pd.Series([1, 3, 5, -9])
0   1 
1   3 
2   5 
3  -9 
dtype: int64

# 通过 values 属性获取值
s1.values # array([ 1, 3, 5, -9], dtype=int64)

# 通过 index 属性获取索引
# 返回一个 RangeIndex 对象
s1.index
RangeIndex(start=0, stop=4, step=1)

# 指定索引
s2 = pd.Series([1, 3, 6, 2], index=['a', 'b', 'c', 'd'])
# 非数值索引的类型是 Index
Index(['a', 'b', 'c', 'd'], dtype='object')

# 类似 dict 的索引
s2['a'] # 1
s2['b'] # 3
s2[['b', 'd']] # 索引多个值
b   3
d   2
dtype: int64

# 过滤
s2[s2 > 3]
c    6 
dtype: int64

# 判断索引是否存在
'a' in s2 # True

# 用 dict 创建 Series
data = {'a': 1, 'b': 2, 'c': 5, 'd': 9}
pd.Series(data)
a   1
b   2
c   5
d   9
dtype: int64

# 可以传入一个索引来替换索引顺序，不存在对应索引的数据会变为 NaN
s2 = pd.Series(data, index=['b', 'e', 'a', 'c'])
b   2
e   NaN
a   1
c   5
dtype: int64

# 通过 isnull() 判空
pd.isnull(s2)
a   False 
b   False 
c   False 
d   False 
dtype: bool

# 可以给 Series 自身和 index 指定 name
s2.name = 'data'
s2.index.name = 'attr'
attr
b      2
e      NaN
a      1
c      5
Name: data, dtype: int64
```
### 6.1.2 DataFrame
```python
class DataFrame(NDFrame, OpsMixin):
	def __init__(
        self,
        data=None,
        index: Axes | None = None,
        columns: Axes | None = None,
        dtype: Dtype | None = None,
        copy: bool | None = None,
    ): ...
    # data: ndarray (structured or homogeneous), Iterable, dict, or DataFrame
    # index: Index or array-like
    # columns: Index or array-like

# 用 dict 创建一个 df
d = {'col1': [1, 2], 'col2': [3, 4]}
df = pd.DataFrame(data=d)
	 col1   col2
0      1      3
1      2      4

# head() 和 tail() 可以获取 df 的前(后) n 行数据
# 默认 5 行，底层调 iloc
def head(self: NDFrameT, n: int = 5) -> NDFrameT:
	return self.iloc[:n]
	
def tail(self: NDFrameT, n: int = 5) -> NDFrameT:
	if n == 0:
		return self.iloc[0:0]
	return self.iloc[-n:]

df.head()

# 指定 col 创建 df
data = {'state': ['Ohio', 'Ohio', 'Ohio', 'Nevada', 'Nevada', 'Nevada'],
        'year': [2000, 2001, 2002, 2001, 2002, 2003],
        'pop': [1.5, 1.7, 3.6, 2.4, 2.9, 3.2]}
pd.DataFrame(data, columns=['year', 'state', 'pop'])
	year    state    pop
0    2000    Ohio    1.5  
1    ...

# 传入不存在的 col 时，对应的值会变为 NaN
```