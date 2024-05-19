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
#### 1.1.2.2 常用方法
##### 1.1.2.2.1 capitalize()
```python
str = 'abcdef'
str.capitalize() # Abcdef
```
- builtins.pyi
```
```