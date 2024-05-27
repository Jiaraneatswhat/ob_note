# 1 基础
## 1.1 数据类型
### 1.1.1 整型

| type      | size                                         |
| --------- | -------------------------------------------- |
| short     | 2bytes                                       |
| int       | 4bytes                                       |
| long      | win 下 4bytes, linux 下 4bytes(32), 8bytes(64) |
| long long | 8bytes                                       |
- 每个整型都有 `signed` 和 `unsigned` 两个版本
- `short < int <= long <= long long`
### 1.1.2 浮点型

| type   | size   | sig-digits |
| ------ | ------ | ---------- |
| float  | 4bytes | 7          |
| double | 8bytes | 15-16      |
- 默认输出 6 位有效数字
### 1.1.3 char

| type    | size   |
| ------- | ------ |
| char    | 1byte  |
| wchar_t | 2bytes |
- `typedef unsigned short wchar_t`
- `wchar_t` 用于存储外文字符
- 定义时要以 `L` 开头，否则转换为 `char`
- `char` 类型字符串以 `'\0'` 结尾，`wchar_t` 以 `'\0\0'` 结尾
- 输出 `wchar_t` 类型需要 `wcout`

### 1.1.4 string
- 在 `C` 的 `char str[]`基础上，增加了新的定义方式 `string str = ""`
- `string` 是 `basic_string` 的一个实例化类