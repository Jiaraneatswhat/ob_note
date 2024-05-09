# 1 基础
## 1.1 main()的几种形式
```c
main() {}

// void 表示不接受参数，不加 void 表示参数的个数类型不确定 
int main(void) 
{
	return 0; // C99标准, 可不写
}

// 最通用的命令行参数的主函数
// char** argv 等价于 char* argv[]
int main(int argc, char** argv) 
{
	return 0; 
}

int main() 
{
	return 0; // C++ 的主函数, C 也支持
}

void main() // 不标准
```
## 1.2 数据类型
### 1.2.1 整型
| 类型           | bytes          |
| -------------- | -------------- |
| short          | 2              |
| unsigned short | 2              |
| int            | 2/4            |
| unsigned int   | 2/4            |
| long           | 4/8(64 位 gcc) |
| unsigned long  | 4/8            |
| long long(c99) | 8              |
| unsigned long long(c99)               | 8               |
- <font color='red'>也可以通过</font> `__int8, 32` <font color='red'>等来表示 x 位整型</font>
```c
// %x/%X: hex    %d: dec    %o: oct 
// 0开头的为 oct
printf("value16: %x, value10: %d, value8: %o\n", 26, 26, 26);
printf("value16: %x, value10: %d, value8: %o\n", 0253, 0253, 0253);
printf("value16: %x, value10: %d, value8: %o\n", 0xa4, 0xa4, 0xa4);
```
- 整型常量后缀
	- short, unsigned short, int 无后缀
	- unsigned int -> u / U
		- 23U, 0u
	- long -> l / L
	- unsigned long -> lu (u, l 顺序可交换，大小写都行)
	- long long -> ll
	- unsigned long long -> luu
- 