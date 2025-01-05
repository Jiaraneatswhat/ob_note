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

# 与普通集合类似，通过 len() 方法可以计算出卡牌的数目：
>>> deck = FrenckDeck()
>>> len(deck)
52

# 实现了 __getitem__ 方法让我们可以索引卡牌：
>>> deck[0]
Card(rank='2', suit='spades')

# python 已经有一个从序列中获取随机元素的方法: random.choice
>>> from random import choice
>>> choice(deck)
Card(rank='9', suit='spades')

# 可以看出使用特殊方法的两个好处：不用记标准操作的方法名; 不用重复造轮子


```

