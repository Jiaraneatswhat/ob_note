## A Pythonic Card Deck
- 当我们想让对象支持且能够与基本语言结构互动时可以实现特殊方法，这些结构有：
	- 集合
	- 访问属性
	- 迭代
	- 运算符重载
	- 函数和方法调用
	- 字符串表示和格式化
	- 使用 `await` 进行异步编程
	- 创建和销毁对象
	- 使用 `with` 或 `async with` 语句管理上下文

<font color='darkred'>Example 1-1</font>. 一摞有序的纸牌
```python
import collections

# suit 是花色，rank 是牌的大小
# 使用 namedtuple 构建只有属性没有自定义方法的类
Card = collections.namedtuple('Card', ['rank', 'suit'])

# namedtuple(_typename_,_field_names_,...) 方法返回一个 tuple 的子类typename, 它的属性就是 field_names 中定义的
# 定义一个黑桃二：
#     c1 = Card('2', 'spades')
#     print(c1.rank, c1.suit) -> 2 spades

class FrenchDeck:
	ranks = [str(n) for n in range(2, 11)] + list('JQKA')
	suits = 'spades diamonds clubs hearts'.split()
    
    def __init__(self):
        self._cards = [Card(rank, suit) for suit in self.suits
                                        for rank in self.ranks]
    def __len__(self):
        return len(self._cards)

    def __getitem__(self, position):
        return self._cards[position]
```
- 与普通集合类似，通过 `len()` 方法可以计算出卡牌的数目：
```python
>>> deck = FrenckDeck()
>>> len(deck)
52
```
- 实现了 `__getitem__` 方法让我们可以索引卡牌：
```python
>>> deck[0]
Card(rank='2', suit='spades')

# python 已经有一个从序列中获取随机元素的方法: random.choice
>>> from random import choice
>>> choice(deck)
Card(rank='9', suit='spades')

# 可以看出使用特殊方法的两个好处：不用记标准操作的方法名; 不用重复造轮子
# 由于 __getitem__ 将操作交给了 self._cards 的 [] 运算符，我们的牌组就可以自动支持切片：
>>> deck[:3]
[Card(rank='2', suit='spades'), Card(rank='3', suit='spades'), Card(rank='4', suit='spades')]

>>> deck[12::13]
[Card(rank='A', suit='spades'), Card(rank='A', suit='diamonds'), Card(rank='A', suit='clubs'), Card(rank='A', suit='hearts')]
```
- 同时牌组也支持迭代:
```python
>>> for card in deck:
...    print(card)
Card(rank='2', suit='spades') 
Card(rank='3', suit='spades')
...

# 迭代通常是隐式的，如果一个集合没有 __contains__ 方法，那么 in 操作就会进行顺序扫描，对于我们的牌组来说：
>>> Card('Q', 'hearts') in deck
True
```
- 对于排序操作，这里实现一个排序方法：
```python
suit_values = dict(spades=3, hearts=2, diamonds=1, clubs=0)

def spades_high(card):
	# 获取卡牌对应索引, 2 为 0, 3 为 1, ...
	rank_value = FrenchDeck.ranks.index(card.rank)
	return rank_value * len(suit_values) + suit_values[card.suit]

# 定义了排序方法后，通过 sorted() 方法排序：
>>> for card in sorted(deck, key=spades_high):
...     print(card)
Card(rank='2', suit='clubs') 
Card(rank='2', suit='diamonds') 
Card(rank='2', suit='hearts')
...
```
## How Special Methods Are Used
- 特殊方法是由解释器调用的，不能写 `my_object.__len__()` 而是 `len(my_object)`，如果 `my_object` 是用户定义类的实例，那么 Python 就会调用你实现的 `__len__` 方法
- 对于一些内置类型如 `list, str, bytearray` 或 Numpy 数组时，Python 在底层存放这些可变大小的集合时使用的是 C 中的 `struct` 叫做 `PyVarObject`，这个对象包含一个 `ob_size` 属性，维护了集合中元素的数目。因此 `len(my_object)` 会从 `ob_size` 属性中获取值，比起调用方法来要快很多
- 大部分情况下，特殊方法是隐式调用的，例如语句 `for i in x`：它实际上会调用 `iter(x)`，迭代器就会调用 `x.__iter__()` 或 `x.__getitem__()`
## Emulating Numeric Types
- 我们实现一个类来代表二维向量，支持加法运算，求模，标量乘法
<font color='darkred'>Example 1-2</font>. 二维向量
```python
import math

class Vector:

    def __init__(self, x=0, y=0):
        self.x = x
        self.y = y

    def __repr__(self):
        # !r 等价于 %r, 表示用 repr() 处理 self.x
        return f'Vector({self.x!r}, {self.y!r})'
        
    def __abs__(self):
        # 返回欧氏范数 i.e. \sqrt{x_1^2 + x_2^2 + ...}
        return math.hypot(self.x, self.y)

    def __bool__(self):
	    return bool(abs(self))

    def __add__(self, other):
        x = self.x + other.x
        y = self.y + other.y
        return Vector(x, y)

    def __mul__(self, scalar):
        return Vector(self.x * scalar, self.y * scalar)
```
- 可以直接使用 `+` 运算符, `abs()` 方法来对 `Vector` 类进行计算
```python
>>> v1 = Vector(2, 4)
>>> v2 = Vector(2, 1)
>>> v1 + v2
Vector(4, 5)

>>> v = Vector(3, 4)
>>> abs(v)
5.0

>>> v * 3
Vector(9, 12)
```
## String Representation
-  `__repr__` 这个特殊方法由内置的 `repr` 方法调用，用于获取对象的字符串表达形式。不实现 `__repr__` 时控制台会打印出对象地址 (类似 `Java` 需要重写 `toString()` 方法)
- 与之对比，`__str__` 由内置的 `str()` 方法调用，由 `print` 函数隐式调用
- 有时 `__repr__` 方法返回的字符串足够友好，无须再定义 `__str__` 方法，因为继承 `object` 的类最终会调用 `__repr__` 方法
## Boolean Value of a Custom Type
- Python 在决定一个对象 `x` 是 `True` 还是 `False` 时会调用 `bool(x)`
- 用户定义的类的实例默认为真，除非实现了 `__bool__` 或 `__len__`
