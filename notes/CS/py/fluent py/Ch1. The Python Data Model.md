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

Card = collections.namedtuple()
```
