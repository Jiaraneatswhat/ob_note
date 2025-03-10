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
![[table 2-1_1.png]]
![[table 2-1_2.png]]
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
- 在并行赋值时，`'*'` 可以用于任意位置的单个变量
```python
>>> a, *body, c, d = range(5)
>>> a, body, c, d
(0, [1, 2], 3, 4)
```
## Unpacking with * in Function Calls and Sequence Literals
- 在函数调用中，可以多次使用 `'*'`
```python
def fun(a, b, c, d, *rest):
...    	return a, b, c, d, rest
...
>>> fun(*[1, 2], 3, *range(4, 7)) # 将 [1, 2] 和 range(4, 7) 拆包传入
(1, 2, 3, 4, (5, 6))
```
- `'*'` 也可以用来定义 `list, tuple` 或 `set` 字面量:
```python
>>> * range(4), 4
(0, 1, 2, 3, 4)
>>> [*range(4), 4]
[0, 1, 2, 3, 4]
>>> {*range(4), 4, *(5, 6, 7)}
{0, 1, 2, 3, 4, 5, 6, 7}
```
## Nested Unpacking
<font color='darkred'>Example 2-8</font> 是嵌套拆包的一个例子：
```python
metro_areas = [
('Tokyo', 'JP', 36.933, (35.689722, 139.691667)),
('Delhi NCR', 'IN', 21.935, (28.613889, 77.208889)),
('Mexico City', 'MX', 20.142, (19.433333, -99.133333)),
('New York-Newark', 'US', 20.104, (40.808611, -74.020386)),
('São Paulo', 'BR', 19.649, (-23.547778, -46.635833)),
]

def main():
	print(f'{"":15} | {"latitude":>9} | {"longtitue":>9}')
	# 将最后一个字段赋给了嵌套的元组，拆包坐标
	for name, _, _, (lat, lon) in metro_areas:
		if lon <= 0:
			print(f'{name:15} | {lat:9.4f} | {lon:9.4f}')

if __name__ = '__main__':
	main()

# 输出如下：
			    |  latitude |  longitude
Mexico City       |  19.4333  |  -99.1333
New York-Newark   |  40.8086  |  -74.0204
São Paulo         |  -23.5478 |  -46.6358
```
## Pattern Matching with Sequences
- Python 3.10 最显著的功能是使用 `match/case` 进行模式匹配：
<font color='darkred'>Example 2-9</font> 是嵌套拆包的一个例子：
```python
def handle_command(self, messgae):
	match message:
		case ['BEEPER', frequency, times]:
			self.beep(times, frequency)
			...
		# 默认的 case 子句
		case _: 
			raise InvalidCommand(message)
```
- `match` 的一大改进是<font color='red'>析构</font> (deconstructing)，析构广泛用于支持模式匹配的语言中——例如 Scala
<font color='darkred'>Example 2-10</font> 展示了析构的操作，重写了 <font color='darkred'>Example 2-8</font> 中的一些部分：
```python
metro_areas = [
('Tokyo', 'JP', 36.933, (35.689722, 139.691667)),
('Delhi NCR', 'IN', 21.935, (28.613889, 77.208889)),
('Mexico City', 'MX', 20.142, (19.433333, -99.133333)),
('New York-Newark', 'US', 20.104, (40.808611, -74.020386)),
('São Paulo', 'BR', 19.649, (-23.547778, -46.635833)),
]

def main():
	print(f'{"":15} | {"latitude":>9} | {"longtitue":>9}')
	for record in metro_areas:
		# match 匹配的对象是 metro_areas 中的每一个元组
		match record:
			case [name, _, _, (lat, lon)] if lon <= 0:
				print(f'{name:15} | {lat:9.4f} | {lon:9.4f}')
```
- 序列模式可以匹配 `collections.abc.Sequence` 的大部分实际子类或虚拟子类，`str, bytes, bytearray` 除外
- 在 `match/case` 上下文中，`str, bytes, bytearray` 不被视为序列，因为这些类型被当做是原子值对待，要想使用必须先在 `match` 语句中进行转换
- 在标准库中，这些类型与序列模式兼容：
	- list
	- memoryview
	- array.array
	- tuple
	- range
	- collecitions.deque
- 与拆包不同，模式不会析构序列以外的可迭代对象
- 模式中的任意一部分可以使用关键字 as 绑定到变量上：
```python
case [name, _, _, (lat, lon) as coord]: 
```
- 也可以添加类型信息让模式更具体：
```python
case [str(name), _, _, (float(lat), float(lon))]: 
```
- 如果想略过中间几项，只匹配第一项为 `str`，最后一项为包含两个 `float` 的 `tuple` 的序列：
```python
case [str(name), *_, (float(lat), float(lon))]
```
# Slicing
## Why Slices and Ranges Exclude the Last Item
- 切片和区间排除最后一项与 Python，C 和其他语言中从 `0` 开始的索引相匹配，有以下好处：
	- 只知道停止位置时，可以很容易地得到切片或区间的长度
	- 知道起始和停止位置时，可以很容易地计算长度：`stop - start`
	- 给定任意一个索引 `x`，可以很容易地分割成两个不重叠的部分
## Slice Objects
- `s[a: b: c]` 可以用来指定步长 `c`，让切片跳过一些元素。步长也可以是负数，反向返回元素
```python
>>> s = 'bicycle'
>>> s[::3]
'bye'
>>> s[::-1]
'elcycib'
>>> s[::-2]
'eccb'
```
`a: b: c` 只在 `[]` 内部有效，表示索引或下标，得到的结果是一个切片对象：`slice(a, b, c)`
<font color='darkred'>Example 2-13</font>. 从纯文本形式的发票中提取商品信息
```python
>>> invoice = """
... 0.....6.................................40........52...55........
... 1909 Pimoroni PiBrella $17.50 3 $52.50
... 1489 6mm Tactile Switch x20 $4.95 2 $9.90
... 1510 Panavise Jr. - PV-201 $28.00 1 $28.00
... 1601 PiTFT Mini Kit 320x240 $34.95 1 $34.95
... """
>>> SKU = slice(0, 6)
>>> DESCRIPTION = slice(6, 40)
>>> UNIT_PRICE = slice(40, 52)
>>> QUANTITY = slice(52, 55)
>>> ITEM_TOTAL = slice(55, None)
>>> line_items = invoice.split('\n')[2:]
>>> for item in line_items:
...     print(item[UNIT_PRICE], item[DESCRIPTION])
...
	$17.50    Pimoroni PiBrella
	$4.95     6mm Tactile Switch x20
    $28.00    Panavise Jr. - PV-201
	$34.95    PiTFT Mini Kit 320x240
```
## Assigning to Slices
- 在赋值语句左侧使用切片，可以原地修改可变序列
```python
>>> l = list(range(10))
>>> l
[0, 1, 2, 3, 4, 5, 6, 7, 8, 9]

>>> l[2: 5] = [20, 30]
>>> l
[0, 1, 20, 30, 5, 6, 7, 8, 9]

>>> del l[5: 7]
>>> l
[0, 1, 20, 30, 5, 8, 9]

# 右侧必须是可迭代对象，只有一个值时，必须加 []
>>> l[2: 5] = 100
Traceback (most recent call last):
File "<stdin>", line 1, in <module>
TypeError: can only assign an iterable
```
# Using + and * with Sequences
## Building Lists of Lists
- 有时我们需要初始化一个有着特定数目的嵌套 `list` 的 `list`
<font color='darkred'>Example 2-14</font>. 一个列表中嵌套 3 个长度为 3 的列表，可以表示井字棋 (tic-tac-toe) 棋盘
```python
>>> board = [['_'] * 3 for i in range(3)]
>>> board
[['_', '_', '_'], ['_', '_', '_'], ['_', '_', '_']]
>>> board[1][2] = 'X'
>>> board
[['_', '_', '_'], ['_', '_', 'X'], ['_', '_', '_']]
```
<font color='darkred'>Example 2-15</font>. 在一个列表中 3 次引用同一个列表是没有意义的
```python
>>> weird_board = [['_'] * 3] * 3
>>> weird_board
[['_', '_', '_'], ['_', '_', '_'], ['_', '_', '_']]
# 所有引用都指向同一个数组，因此所有的元素都发生了变化
>>> weird_board[1][2] = 'O'
>>> weird_board
[['_', '_', 'O'], ['_', '_', 'O'], ['_', '_', 'O']]
```
## Augmented Assignment with Sequences
- 使得 `+=` 工作的特殊方法是 `__iadd__`，即就地求和
- 如果 `__iadd__` 没有实现，那么就会转回调用 `__add__`
```python
>>> a += b
# 如果 a 实现了 __iadd__，那就会调用
# 如果 a 是可变序列 (list, bytearray, array.array)，那么就地修改 a
# 如果没有实现 __iadd__, 那么该式等价于 a = a + b
```
- 以上内容也适用于 `__imul__` 和 `*=` 运算符
## A += Assignment Puzzler
<font color='darkred'>Example 2-16</font>. 一个谜题
```python
>>> t = (1, 2, [30, 40])
>>> t[2] += [50, 60]
>>> t
```
运算结果是 ( D )
- A: `t` 变成 `(1, 2, [30, 40, 50, 60])`
- B: `TypeError: 'tuple' object does not support item assignment`
- C: 以上都不对
- D: A 和 B
```python
# 输出为：
>>> t = (1, 2, [30, 40])
>>> t[2] += [50, 60]
Traceback (most recent call last):
File "<stdin>", line 1, in <module>
TypeError: 'tuple' object does not support item assignment
>>> t
(1, 2, [30, 40, 50, 60])
```
对字节码进行分析：
<font color='darkred'>Example 2-18</font>. `s[a] += b` 的字节码 
```python
# dis 用于获取字节码
>>> dis.dis('s[a] += b')
1        0 LOAD_NAME        0 (s)
        3 LOAN_NAME        1 (a)
        6 DUP_TOP_TWO
        # BINARY_SUBSCR 根据索引获取值，将 s[a] 放在栈顶
        7 BINARY_SUBSCR
        8 LOAD_NAME        2 (b)
        # 执行加法，如果栈顶指向可变对象那么操作就会成功
        11 INPLACE_ADD
        12 ROT_THREE
        # 将栈顶的值赋给 s[a]
        13 STORE_SUBSCR
        14 LOAD_CONST      0 (None)
        17 RETURN_VALUE 
```
- 从中可以总结出：
	- 不要在元组中存放可变的元素
	- 增量赋值不是原子操作——在操作后会抛出异常
	- 字节码是一个分析程序内部的好工具
# When a List Is Not the Answer
## Arrays
- 如果一个 `list` 只包含数字，那么`array.array` 是一个更有效的替代选择，数组支持所有的可变序列操作 (`.pop, .insert, .extend`)，以及快速读取和存储的方法，例如 `.frombytes, .tofile`
- 创建 `array` 对象时提供一个类型代码 (`typecode`)，一个确定底层使用什么 C 类型存储数组元素的字母，例如 b 对应的是 C 中的 `signed char` 类型。如果使用 `array('b')` 创建一个数组，那么数组中的每一项都用一个字节存储，都被解释为整数。对于大型数值序列，这样可以节省大量内存
<font color='darkred'>Example 2-19</font>. 创建，保存，加载一个大型浮点数组
```python
>>> from array import array
>>> from random import random
    # 创建双精度浮点数数组
>>> floats = array('d', (random() for i in range(10**7)))
>>> floats[-1]
0.07802343889111107

>>> fp = open('floats.bin', 'wb')
    # 将数组存在二进制文件中
>>> floats.tofile(fp)
>>> fp.close()
	# 创建一个空数组
>>> floats2 = array('d')
>>> fp = open('floats.bin', 'rb')
    # 从文件中读取
>>> floats2.fromfile(fp, 10**7)
>>> fp.close()
>>> floats2[-1]
0.07802343889111107
>>> floats2 == floats
True
```
--------------------------------------------------------------------
Table 2-3. list 和 array 中的方法与属性比较
![[table 2-3.png]]
## Memory Views
- 内置的 `memoryview` 类是一个共享内存的序列类型，可以在不拷贝字节的情况下进行切片
- `memoryview` 在数据结构 (PIL(Python Imaging Library) 图像, SQLite, NumPy 数组) 间共享内存，而不是事先复制
- `memoryview.cast` 方法以多个字节为单位进行读写，而不移动位，最终返回一个分享同一块内存的 `memoryview` 对象
<font color='darkred'>Example 2-20</font> 展示了如何在同一个 6 字节数组上创建多个大小的视图：
```python
>>> from array import array
>>> octets = array('B', range(6))
>>> m1 = memoryview(octets)
>>> m1.tolist()
[0, 1, 2, 3, 4, 5]

>>> m2 = m1.cast('B', [2, 3])
>>> m2.tolist()
[[0, 1, 2], [3, 4, 5]]

>>> m3 = m1.cast('B', [3, 2])
>>> m3.tolist()
[[0, 1], [2, 3], [4, 5]]

>>> m2[1,1] = 22
>>> m3[1,1] = 33
>>> octets
array('B', [0, 1, 2, 33, 22, 5])
```
## Deques and Other Queues
- `collections.deque` 是一个线程安全的双端队列，可以在两端快速插入和移除元素
<font color='darkred'>Example 2-23</font>. `deque` 的使用
```python
>>> from collections import deque
>>> dq = deque(range(10), maxlen=10)
>>> dq
deque([0, 1, 2, 3, 4, 5, 6, 7, 8, 9], maxlen=10)
# 循环右移 n 位 (n > 0) 或循环左移 n 位 (n < 0)
>>> dq.rotate(3)
>>> dq
deque([7, 8, 9, 0, 1, 2, 3, 4, 5, 6], maxlen=10)
>>> dq.rotate(-4)
>>> dq
deque([1, 2, 3, 4, 5, 6, 7, 8, 9, 0], maxlen=10)
>>> dq.appendleft(-1)
>>> dq
deque([-1, 1, 2, 3, 4, 5, 6, 7, 8, 9], maxlen=10)
>>> dq.extend([11, 22, 33])
>>> dq
deque([3, 4, 5, 6, 7, 8, 9, 11, 22, 33], maxlen=10)
    # 后进先出
>>> dq.extendleft([10, 20, 30, 40])
>>> dq
deque([40, 30, 20, 10, 3, 4, 5, 6, 7, 8], maxlen=10)
```
--------------------------------------------------------------------
Table 2-4. list 和 deque 的方法
![[table 2-4_1.png]] 
![[table 2-4_2.png]]
- Python 标准库还实现了其他的队列：
	- queue: 提供了同步的类如 `SimpleQueue`, `Queue`, `LifoQueue` 以及 `PriorityQueue `
	- multiprocessing: 实现了无界的 `SimpleQueue` 和有界的 `Queue`
	- asyncio: 提供了 `Queue`, `LifoQueue`, `PriorityQueue` 和 `JoinableQueue`
	- heapq: `heapq` 没有实现队列类，但是提供了 `heappush` 和 `heappop` 等函数，可以把可变序列当作堆队列或优先级队列使用