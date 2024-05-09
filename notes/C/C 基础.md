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
#### 1.2.1.1 表示
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
- short 等价于 signed short int, 其他类似
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
#### 1.2.1.2 定义
```c
	// 连续定义
	int a, b = 3, d = b;
	a = b = 34; // 先执行 b = 34
	
	// 查看地址
	int e = -45;
	// x86: 00D5F9DC x64: 0000004FBF6FFB44
	printf("%p", &e);
```
#### 1.2.1.3 scanf()
```c
	// 输入
	// scanf() 在 c11 后的标准中删除
	// #define _CRT_SECURE_NO_WARNINGS 才可以使用 scanf
	// scanf_s() 中 ""内无分隔符时，控制台可以用空格或 \n 分隔
	scanf_s("%d%d", &a, &b);
	// 有其他分隔符时，控制台必须使用相同的
	scanf_s("%d,%d", &a, &b); // ""里不要加 \n
	printf("a = %d, b = %d\n", a, b);
```
- scanf() 存在缓冲区溢出的问题，第一次读完剩下的字符在缓冲区中，第二次调用 scanf() 时就会影响结果
- scanf_s() 新增了一个参数，可以在读取时指定最大的读取长度
- scanf_s() 也要求字符串以 `'\0'` 结尾，否则返回一个错误码，不会存储任何字符到变量中
#### 1.2.1.4 sizeof
```c
// 关键字
// 对变量使用时不需要 ()
int a = 4;
unsigned int b = sizeof a; // sizeof 返回 unsigned int
unsigned int c = sizeof(int); // 对类型关键字使用时需要 ()

printf("%u", sizeof(short)); // 2
printf("%u", sizeof(int)); // 4
printf("%u", sizeof(long)); // 4
printf("%u", sizeof(long long)); // 8
```
#### 1.2.1.5 整型在内存中的存储
- 原码，反码和补码
	- 正数的原码反码补码相同
	- 负数的原码在绝对值的基础上最高位变 1
	- 负数的反码最高位不变，其他位取反
	- 负数的补码 = 反码 + 1
- 整型在内存中以补码的形式存放
	- 使用补码可以将符号位和数值域统一处理
	- 加法和减法可以统一处理
#### 1.2.1.6 大小端
- 大端 -> 数据的低位保存在高地址中，高位保存在低地址中
- 小端 -> 数据的低位保存在低地址中，高位保存在高地址中
- 以 `00000001` 为例
	- 小端：`01 00 00 00`
	- 大端：`00 00 00 01`
### 1.2.2 float
#### 1.2.2.1 IEEE 754 标准
- 通过符号位，指数偏移和分数值来表示浮点数
- $Value=S\times E \times M$
	- S:
		- 0 表示正数，1 表示负数
	- E：
		- 2 的幂，对浮点数加权
	- M:
		- 有效数字位，二进制小数
- 32 位单精度浮点数
	- M
		- 23 位, 有效数字有 $2^{23+1}$ 个
		- 二进制数的范围是 `(0, 16777216)`
		- $10^7<16777216<10^8$ , 因此单精度浮点数有效位数是 7 位
	- E
		- 8 位，有效数字有 $2^8=256$ 个，实际指数范围 $(-126, 127)$
- 异常值
	- 零值：指数尾数部分均为 0，规定`-0 = +0`
	- 非规格化值：指数为 0 尾数非 0
	- 无穷值：指数全 1，尾数全 0，根据符号位分别表示 $\pm \infty$
	- NAN: 指数全 1，尾数非 0
#### 1.2.2.2 表示
| 类型   | bytes | 后缀 | 格式说明符 | 精度(有效位) |
| ------ | ----- | ---- | ---------- | ------------ |
| float  | 4     | f/F  | %f         | 8(6)         |
| double | 8     | 无   | %lf        | 17(10)       |
| long double       | 8/10/16      | l/L     | %Lf           | 17/22/38(10)             |
- 使用 `%e` 统一输出为科学计数法形式
```c
// 有效数字从第一个不为 0 的算起
// 默认输出 6 位小数
printf("%f", 123456789.123456f); // 123456792.000000
// "%.xf" 输出 x 位小数
printf("%.6lf", 123456789.123456);
```
### 1.2.3 char
- 等价于 1byte 的整数
- 格式说明符：
	- %c / %hhd -> 有符号
	- %c / %hhu -> 无符号
	- %c 输出的是字符，%hhx 输出的是数字
```c
char a = '!';
printf("%hhd, %c", a, a); // 33 !
```
#### 1.2.3.1 getchar() 与 putchar()
- getchar() 
	- 用于读取用户键盘的单个字符，返回值类型为 `int`
	- 读取错误时返回 `-1`
	- 会将结束输入的回车也存放在缓冲区中
- putchar()
	- 向终端输出一个字符，不包含`'\n'`
	- 当输入的 `char` 超过八位时，会进行截断
```c
char a = 'a';
putchar(a);
scanf_s("%c", &a, 1);
putchar(a);

char b = getchar();
putchar(b);
```
#### 1.2.3.2 缓冲区问题
- 所有按键都以字符形式存在输入缓冲区，`%c` 会将所有数据都读出来
```c
char c = 'A', d = 'B';
scanf_s("%c%c", &c, 1, &d, 1);
// 输入 "C D\n" 得到 d = 空格

char c = 'A';
int a = 12;
scanf_s("%d", &a);
c = getchar();
// 输入 "12\n", scanf_s 将 12 赋给 a, getchar 将 '\n' 赋给 c 
```
- 解决方法：在使用 `scanf_s()` 后，调用 `rewind(stdin)` 清空缓冲区