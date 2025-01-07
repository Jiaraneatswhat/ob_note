# Overview of Built-In Sequences
- 标准库提供了用 C 实现的丰富的序列类型可供选择：
	- *Container sequences*
		- 可以存储不同类型的数据，包括嵌入嵌套容器
		- 如 `list, tuple, collections.deque` 等
	- *Flat sequences*
		- 存储单一类型的数据
		- 如 `str, bytes, array.array` 等
- 容器序列存放的是对象的引用，而扁平序列在自己的内存中存放数据的值，而不是不同的 Python 对象 (见 Fig. 2-1)
--------------------------------------------------------------------
Fig. 2-1 tuple 和 array 简化的内存图，各含三个元素
![[fig 2-1.png]]
- `tuple` 中存放的是引用的序列，每个元素是不同的 Python 对象，对象中还可以存放其他对象的引用；而 array 是单个对象，存放一个 C 数组
--------------------------------------------------------------------
- 因此扁平序列更加紧凑，但是只能存储一些原始机器值，如 `bytes, integers, floats`
- 内存中的每个 Python 对象都有着标记元数据的 `header`，最简单的 Python 对象 `float`，包含值字段和两个元数据字段：
	- ob_refcnt: 对象的引用数
	- ob_type: 对象类型的指针
	- ob_fval: 存放值的 C `double` 变量
- 在 64 位设备中每个字段占 8 个字节
- 另外也可按可变性对序列类型进行分组:
	- *Mutable sequences*
		- `list, bytearray, array.array, collections.deque`
	- *Immutable sequences*
		- `tuple, str, bytes`

- <font color='darkred'>Fig. 2-2</font> 可视化了可变序列如何从不可变序列中继承所有的方法，实现了几个其他的方法。内置的具体序列类型不是 `Sequence` 和 `MutableSequence` 抽象基类的子类，而是这两个抽象基类注册的虚拟子类，因此它们可以通过以下测试：
```python
>>> from collections import abc
>>> issubclass(tuple, abc.Sequence)
True

>>> issubclass(list, abc.MutableSequence)
True
```
--------------------------------------------------------------------
Fig. 2-2 collections.abc 的一些类的简化 UML 类图 (父类在左侧，斜体表示抽象的类和方法)
![[fig 2..2.png]]
# List Comprehensions and Generator Expressions
## List Comprehensions and Readability
- 使用列表推导式速度更快：
<font color='darkred'>Example 2-1</font>. 从字符串中构建 Unicode 码点列表
```python
>>> symbols = '$¢£¥€¤'
>>> codes = []
>>> for symbol in symbols:
...     codes.append(ord(symbol))
...
>>> codes
[36, 162, 163, 165, 8364, 164]
```
<font color='darkred'>Example 2-2</font>. 从列表推导式中构建 Unicode 码点列表
```python
>>> symbols = '$¢£¥€¤'
>>> codes = [ord(symbol) for symbol in symbols]
>>> codes
[36, 162, 163, 165, 8364, 164]
```
- 列表推导式和生成器表达式的局部作用域
	- Python 3 中的列表推导式和生成器表达式，以及 `set` 和 `dict` 的推导式，for 语句中的变量都在局部作用域内
	- 使用海象运算符 (*Walrus operator*) `:=` 赋值的变量可以在推导式或表达式返回后继续访问——与函数中的本地变量不同
```python
>>> x = 'ABC'
>>> codes = [ord(x) for x in x]
>>> x
'ABC'
>>> codes
[65, 66, 67]

>>> codes = [last := ord(c) for c in x]
>>> last # last 仍然能够访问 
67
>>> c # c 不会保留
NameError: name 'c' is not defined
```
## Listcomps Versus map and filter
- 列表推导式可以实现 map 和 filter 函数的全部功能，而不像 lambda 表达式那样晦涩
<font color='darkred'>Example 2-3</font>. 从列表推导式和 map/filter 函数中构建同一个列表
```python
>>> symbols = '$¢£¥€¤'
>>> beyond_ascii = [ord(s) for s in symbols if ord(s) > 127]
>>> beyond_ascii
[162, 163, 165, 8364, 164]

>>> beyond_ascii = list(filter(lambda c: c > 127, map(ord, symbols)))
>>> beyond_ascii
[162, 163, 165, 8364, 164]
```
## Cartesian Products
- 列表推导式可以从两个以上的可迭代对象的笛卡尔积中构建列表，笛卡尔积的每一项元素是 `tuple`，得到的列表长度等于输入的每个可迭代对象的长度的乘积
<font color='darkred'>Example 2-4</font>. 从列表推导式中构建一个笛卡尔积
```python
>>> colors = ['black', 'white']
>>> sizes = ['S', 'M', 'L']
>>> tshirts = [(color, size) for color in colors for size in sizes]
>>> tshirts
[('black', 'S'), ('black', 'M'), ('black', 'L'), ('white', 'S'),
('white', 'M'), ('white', 'L')]

>>> tshirts = [(color, size) for size in sizes for color in colors]
>>> tshirts
[('black', 'S'), ('white', 'S'), ('black', 'M'), ('white', 'M'), ('black', 'L'), ('white', 'L')]
```
## Generator Expressions
- 列表推导式也可以生成元组，数组或其他类型的序列，但是生成器表达式可以节省内存，因为它会通过迭代器一个接一个地生成元素，而不是构建整个列表提供给其他构造函数
<font color='darkred'>Example 2-5</font>. 从生成器表达式中构建一个元组和数组
```python
>>> symbols = '$¢£¥€¤'
>>> tuple(ord(symbol) for symbol in symbols)
(36, 162, 163, 165, 8364, 164)

>>> import array
# array 构造函数接受两个参数，因此生成器表达式两侧需要圆括号
>>> array.array('I', (ord(symbol) for symbol in symbols))
array('I', [36, 162, 163, 165, 8364, 164])
```
<font color='darkred'>Example 2-6</font>. 使用生成器表达式计算笛卡尔积
```python
>>> colors = ['black', 'white']
>>> sizes = ['S', 'M', 'L']
# 生成器表达式会逐项地产生元素，而不是整个列表
>>> for tshirt in (f'{c} {s}' for c in colors for s in sizes):
... print(tshirt)
...
black S
black M
black L
white S
white M
white L
```
# Tuples Are Not Just Immutable Lists
## Tuples as Records
- 元组可以存放记录，元组中的一个项对应一个字段的数据，项的位置决定数据的意义
<font color='darkred'>Example 2-7</font>. 把元组当作记录使用
```python
# 经纬度
>>> lax_coordinates = (33.9425, -118.408056)
# chg: 人口变化百分比
>>> city, year, pop, chg, area = ('Tokyo', 2003, 32_450, 0.66, 8014)
>>> traveler_ids = [('USA', '31195855'), ('BRA', 'CE342567'),
... ('ESP', 'XDA205856')]
>>> for passport in sorted(traveler_ids):
... print('%s/%s' % passport)
...
BRA/CE342567
ESP/XDA205856
USA/31195855

# 拆包，对第二项不感兴趣将其赋给 _
>>> for country, _ in traveler_ids:
... print(country)
...
USA
BRA
ESP
```
## Tuples as Immutable Lists
- Python 标准库和解释器将元组作为不可变列表使用，这样做的两点好处：
	- 清晰：代码中的元组的长度不变
	- 性能：相同长度下元组所占内存比列表少
- 元组的不变性只针对其包含的引用。元组中的引用不能被删除或替换，但是如果引用指向一个可变对象，当这个对象变化时，元组的值就会变化
- 下面的代码创建两个元组—— `a` 和 `b` ——初始状态下相等。<font color='darkred'>Fig 2-4</font> 代表了内存中的 `b` 元组的初始布局：
```python
>>> a = (10, 'alpha', [1, 2])
>>> b = (10, 'alpha', [1, 2])
>>> a == b
True

>>> b[-1].append(99)
>>> a == b
False

# b 发生了变化
>>> b
(10, 'alpha', [1, 2, 99])
```
-------------------------------------------------------------------
Fig. 2-4 元组本身是不变的，但是只意味着其中的引用会始终指向同一个对象
![[fig 2.4.png]]
## Comparing Tuple and List Methods
--------------------------------------------------------------------
Table 2-1. `list` 和 `tuple` 的方法和属性
![[table 2-1.png]]
# Unpacking Sequences and Iterables
- 拆包可以避免从序列中通过索引来提取元素，减少出错的可能；拆包的目标可以是任何可迭代的对象——包括不支持 `'[]'` 索引的迭代器
- 唯一的要求是可迭代对象每次只能产生一个元素，但是使用 `'*'` 可以一次捕获剩余全部元素
- 拆包最明显的形式是*并行赋值* (*parallel* *assignment*)，即把可迭代对象中的项赋值给变量元组：
```python
>>> lax_coordinates = (33.9425, -118.408056)
>>> latitude, longitude = lax_coordinates # unpacking
>>> latitude
33.9425
```
- 使用拆包的一个优雅方式是不使用中间变量交换两个变量的值
```python
>>> b, a = a, b
```
- 另一个拆包的例子是调用函数时在参数前加上 `'*'`
```python
>>> divmod(20, 8)
(2, 4) # 20 / 8 的商是 2，余数是 4

# 拆包允许函数返回多个值
>>> t = (20, 8)
>>> divmod(*t)
(2, 4)
```
## Using * to Grab Excess Items
- 定义函数时可以用 `*args` 捕获余下的任意数量的参数，Python 3 将这一思想延伸到了并行赋值上：
```python
>>> a, b, *rest = range(5)
>>> a, b, rest
(0, 1, [2, 3, 4])

>>> a, b, *rest = range(2)
(0, 1, [])
```
- 在并行赋值时，`'*'` 可以用于任意wei