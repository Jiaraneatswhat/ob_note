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
- 原型：`inline int__cdecl scanf(const char* const _Format, ...)`
	- `inline int__cdecl scanf_s(const char* const _Format, ...)`
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
- 原型
	- `int__cdecl getchar(void)`
	- `int putchar(int _Character)`
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
- `void __cdecl rewind(FILE *_Stream)`
## 1.3 运算符的优先级与结合性
| priority |                | 说明       | 举例                     | 结合性 |
| -------- | -------------- | ---------- | ------------------------ | ------ |
| 1        | ++             | 后加       | b++                      | 左右   |
|          | --             | 后减       | b--                      |        |
| 2        | ++             | 前加       | ++a                      | 右左   |
|          | --             | 前减       | --a                      |        |
|          | +              | 正号       | +20                      |        |
|          | -              | 负号       | -20                      |        |
|          | ~              | 按位取反   | ~b                       |        |
|          | !              | 取反       | !1                       |        |
|          | sizeof         |            |                          |        |
|          | *              | 内存操作符 | *pp                      |        |
|          | &              | 取址       |                          |        |
| 3        | (type name)    | 复合文字   | (int\[5]){1, 2, 3, 4, 5} | 右左   |
| 4        | *， /， %      | 算术运算   |                          | 左右   |
| 5        | +， -          | 算术运算   |                          | 左右   |
| 6        | >>, <<         |            |                          | 左右   |
| 7        | >, <, <=, >=   |            |                          | 左右   |
| 8        | ==, !=         |            |                          | 左右   |
| 9        | &              | 按位与     |                          | 左右   |
| 10       | ^              | 按位异或   |                          | 左右   |
| 11       | \|             | 按位或     |                          | 左右   |
| 12       | &&             | 逻辑与     |                          | 左右   |
| 13       | \|\|           |            |                          | 左右   |
| 14       | ?:             | 三目       |                          | 右左   |
| 15       | =, +=, *=, ... |            |                          | 右左   |
| 16         | ,               |            | (2 + 3, 4)                         | 左右       |
- 同一个表达式中出现了 `a++` 等其中之一后，不能出现其他的 `++a, --a` 等，不同编译器的结果不一样
# 2 数组与指针
## 2.1 数组
- 一维
```c
// 创建
int a[10] = {1, 3, ...} // c-type 
int a[] = {1, 3, 5, 7} // 可不写元素个数 

// 访问
int a[5] = {4, 2, 7, 8, 4};
scanf_s("%d%d%d%d%d", &a[0], &a[1], &a[2], &a[3], &a[4]);
for (int i = 0; i < 5; i++) printf("%d\n", a[i]);

// 计算数组大小
sizeof a; // 20 bytes
```
- 二维
```c
int a[3][2] = {{3, 2}, {1, 2}, {2, 4}};
// 不带大括号，依次初始化各元素
int a[3][2] = {3, 9, 8} // 3 9, 8 0, 0 0
// 可以省略行数
int a[][2] = {3, 9, 8} // 会生成 2 * 2 的数组

// 计算大小
printf("%zd, %zd", sizeof a, sizeof(int[3][4]));
```
## 2.2 指针
### 2.2.1 指针定义与运算
- 基本数据类型指针
- 声明
	- `short* ps`
	- `char* pc`
```c
int a = 12;
int* pa = &a;  

float b = 2.3f;
float* pb = &b;

// 初始化指针
// vcruntime.h 中：#define NULL ((void *)0)
double* pd = NULL;
```
- 地址操作符 '\*', 取数据
	- '\*' + 空间地址是该空间本身
	- '\*' + 变量地址是该变量本身
	- \*&a == a
	- \*p == a
- 指针只能操作跟类型同样大小的地址
```c
int a = 0x1234;
char *p = &a;
*p = 0x56;

/* 
 * 小端存储，低位的 34 会存在地址低位
 * +----+----+----+----+
 * | 34 | 12 | 00 | 00 |
 * +----+----+----+----+
 * char 指针只能操作一个字节，将 56 写入低位，输出
 * +----+----+----+----+
 * | 56 | 12 | 00 | 00 |
 * +----+----+----+----+, 1256
*/
```
- 二级指针
```c
int** pp = &p;
*pp == p == &a;
**pp == *p == a;
```
### 2.2.2 指针数组和数组指针
- 指针数组
```c
int a[10];
int* c[10]; // 每个元素都是指针

int a, b, c, d;
int* arr[4] = {&a, &b, &c, &d};
*arr[1] == a;
```
- 指针的偏移运算
```c
// type* p;
// p + n == p 移动 sizeof(type) * n 个字节

int a[5] = {1, 2, 3, 4, 5};
int* p = &a[0];
p + 1 == &a[1];

// a[0 + n] = *&a[0 + n] = *(p + n)
int* p = &a[0];
int* p = a;
printf("%d", *(&a[0] + i));
printf("%d", *(p + i));
printf("%d", *p++);
printf("%d", p[i]);
printf("%d", i[p]);
```
- <font color='red'>*(p + n) == p[n] == n[p]</font>
- 数组指针
	- 对数组名取地址，得到数组指针
	- `int a[5] = {}`, `int (*p)[5] = &a`
	- 指针数组和数组指针
		- `int* p[5]` 没有(), \[]的优先级高，p 先和\[]结合，再和\*结合，说明是指针数组
		- `int (*p)[5]` 则是先让* 与 p 结合，说明变量是指针类型, 再与\[] 结合，说明是指向数组的指针
- 函数指针和指针函数
	- 函数指针
		- 保存函数入口地址的指针
		- `int fun(char)`, `fun` 本身就是一个地址常量
	- 指针函数
		- 返回指针的函数
		- `char* p(char)`
	- `*p(char)`
		- `p` 先和 `(char)` 结合，说明 `p` 是一个函数
		- `*p(char)` 就变成了函数的返回值
	- `(*p)(char)`
		- `(*p)` 说明 `p` 是一个指针变量
		- `(char)` 表示是一个参数, 说明 `p` 是函数名称
		- 因此 `p` 是一个指向相同输入型参数和相同返回类型值的函数的函数指针
- 函数指针数组
	- `int (*p[2])(char)`
		- () 由左向右结合，`p` 是数组指针
		- `(char)` 是函数的参数
		- `p` 是指向多个相同输入类型和返回类型的函数指针数组
		- 存放的是每个函数名的地址
- 数组指针操作
```c
int a[5] = { 1, 3, 7, 6, 9 };
int (*p)[5] = &a;
// 使用时用 p, p == &a
printf("%p  %p", p, &a);
for (int i = 0; i < 5; i++) printf("%d %d\n", a[i], (*p)[i]);
```
- 二维数组指针
```c
int a[2][3] = {{8, 5, 3}, {4, 9, 6}};
int (*p)[2][3] = &a;
printf("%d %d\n", a[i], (*p)[i][j], a[i][j]);

// 一维数组指针遍历二维数组
int (*p1)[3] = &a[0];
int (*p2)[3] = &a[1];
// p1 + 1 == p2
// *(p1)[n] == a[0][n] == p1[0][n]
// (*(p1 + 1))[n] == a[1][n]

// 用普通指针访问二维数组
int* p = &a[0][0];
// 输出 8 5 3 4 9 6
for (int i = 0; i < 6; i++) printf("%d %d", *(p + i), p[i]);
```
- 指针的大小: x86 -> 4bytes, x64 -> 8bytes
### 2.2.3 void*
- 通用类型指针，什么类型都可以
```c
// 只能存储，不能使用
int a;     void* pi = &a;
double b;  void* pd = &b;

// 不能使用
*pi = 2; // 报错
pd + 2;

// 将其转换为指定类型后才能使用
*(int*)pi = 2;
```
# 3 函数
- 函数定义在 `main()` 前才能使用
- 要想在 `main()` 后定义，需要在 `main()` 前声明
```c
void fun(void); // 声明
int main(void) {fun()};
void fun(void) {...} // 定义

// 声明也可以放在 main() 中调用前
int main(void) {void fun(void); fun();}
```
- 返回局部变量的地址的问题
```c
int* fun(void);
int main(void) 
{ 
	// 局部变量调用完后会释放，不能再使用
	int* p = fun();
}

int* fun(void) 
{
	int a[5] = {...};
	return &a;
}
```
## 3.1 传递数组参数
- 一维
```c
// fun1(int p[n]), fun1(int p[]) 都会被解析成 int* p
void fun1(int* p, int len);
int main(void)
{
	int a[5] = { 5, 7, 3, 8, 2 };
	fun1(a, 5);
}

// 也可以传数组指针
void fun2(int (*p)[5], int len);
int main(void)
{
	int a[5] = { 5, 7, 3, 8, 2 };
	fun2(&a, 5);
}
```
- 二维
```c
// fun1(int p[m][n]), fun1(int p[][n]) 都会被解析成 int (*p)[n]
// 靠近变量的 [] 会被解析成 *
void fun1(int(*p)[2][3], int row, int col)
{
	int i, j;
	for (i = 0; i < row; i++)
		for (j = 0; j < col; j++)
			printf("%d", (*p)[i][j]);
}
void fun2(int(*p)[3], int row, int col)
{
	int i, j;
	for (i = 0; i < row; i++)
		for (j = 0; j < col; j++)
			printf("%d", p[i][j]);
}
// 当作一维数组遍历
void fun3(int* p, int row, int col)
{
	int i;
	for (i = 0; i < row * col; i++)
		printf("%d", p[i]);
}

int main(void)
{
	int a[2][3] = { {2, 1, 3}, {4, 6, 5} };
	fun1(&a, 2, 3);
	fun2(a, 2, 3);
	fun3(&a[0][0], 2, 3);

}
```
- 引用传递
- 修改谁就传谁的地址
```c
void fun(int* a) 
{
	*a = 3;
}

fun(&a);
```
- 传二级指针修改指针指向
```c

```