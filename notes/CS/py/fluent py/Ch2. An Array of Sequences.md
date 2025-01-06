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
![[fig_2-1.png]]
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
- <font color='darkred'>Fig. 2-2</font> 可视化了可变序列如何从不可变序列中继承所有的方法，实现了几个其他的方法。内置的具体序列类型不是 `Sequence` 和 `MutableSequence` 抽象基类的子类