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
- `short` 等价于 `signed short int`, 其他类似
- `x64` 下，`size_t` 定义为 `unsigned __int64`, `x86` 下定义为 `unsigned int` 
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
- `scanf()` 存在缓冲区溢出的问题，第一次读完剩下的字符在缓冲区中，第二次调用 `scanf()` 时就会影响结果
- `scanf_s()` 新增了一个参数，可以在读取时指定最大的读取长度
- `scanf_s()` 也要求字符串以 `'\0'` 结尾，否则返回一个错误码，不会存储任何字符到变量中
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
void fun(int** p) 
{
	*p = NULL;
}

int main(void)
{
	int* p;
	fun(&p);
	printf("%p", p);
}
```
## 3.2 函数指针
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
- 函数调用的本质是 函数地址 + 参数列表
```c
// 声明函数指针
void fun(int a);
void (*p) (int a) = fun;
void (*p) (int a) = &fun; // 两种方式等价

// fun(3) 等价于 (&fun)(3) 等价于 p(3), (*p)(3)
```
# 4 malloc() 和 free()
## 4.1 C 的内存空间
- C 语言中，内存分为 5 个区
- 也可分为 4 区，将常量区包含在全局区中
	- 栈区：编译器自动分配释放
		- 存放：局部变量，形参，返回值
	- 堆区：程序员分配内存和释放
	- 全局(静态)区
	- 常量区：字符串
	- 代码区：存放程序
- 程序编译后，生成 exe，未执行前分为两个区域
	- 代码区共享，只读
	- 全局区
	
![[c_mem.svg]]
### 4.1.1 栈
- 存局部变量，函数，函数调用时开辟栈区，函数结束时自动回收
- 从高地址向低地址增长
- 使用静态内存分配的变量有：全局变量和静态变量
### 4.1.2 堆
- 通过 `malloc(), realloc(), calloc()` 等开辟的内存
- 从低地址向高地址增长，手动分配
- 不会释放，需要手动回收
### 4.1.3 全局(静态)区
- 用于在编译期间就能确定存储大小的变量的存储区
- 作用于整个程序运行期间都可见的全局变量和静态变量
- 分为 `.bss(未初始化数据)` 段和 `.data` 段
	- `.bss` 
		- 存放未初始化的全局变量和静态变量
		- 存放初始化为 `0` 的全局变量和静态变量
		- 不占用可执行文件空间，内容由 `OS` 初始化
	- `.data`
		- 存放已初始化的全局变量
		- 存放已初始化的静态变量
		- 占用可执行文件空间，由程序初始化
### 4.1.4 常量区
- 存放字符串，数字等常量
- 存放 `const` 修饰的全局变量
- 程序运行期间，常量区的内容不能被修改
### 4.1.5 代码区
- 程序执行的二进制代码放在代码区，不能修改
## 4.2 malloc()
### 4.2.1 使用
- 引用头文件 `<malloc.h>` 
- 原型 `void* __cdecl malloc(size_t _Size)`
	- 返回 `void*`, 可以转换成任意类型使用
	- `_Size` 是要申请的字节数
```c
// 申请数组
// malloc() 会返回首地址
int* p = (int*)malloc(sizeof(int) * n);

int (*p1)[10] = (int(*)[10])malloc(sizeof(int) * 10);
```
## 4.3 free()
- 原型 `void __cdecl free(void *_Block)`
- 释放动态开辟的内存
```c
int* p = (int*)malloc(sizeof(int) * n);
free(p);
// free 后要将指针置为 NULL, 防止通过指针再次访问到无效内存
p = NULL;
```
## 4.4 \_msize()
- 原型 `size_t __cdecl _msize(void *Block)`
- 返回分配的字节大小
```c
int (*p2)[10] = (int(*)[10])malloc(sizeof(int) * 10);
// 用 zu 输出 size_t
printf("%zu", _msize(p2));
```
## 4.5 calloc() 和 realloc()
- `void* __cdecl calloc(size_t _Count, size_t _Size)`
- 与 `malloc()` 类似，区别在于 `calloc()` 会在返回起始地址之前，把在堆区申请的动态内存空间的每个字节都初始化为0
- `void* __cdecl realloc(void* _Block, size_t _Size)`
- 可以重新申请更大的空间
	- 返回首地址后，将原空间的数据依次复制进新空间
	- 自动释放之前的空间
# 5 String
- 以 `'\0'` 结尾的字符数组，`'\0'` 是 `ASCII` 表上第一个字符
- `0` 和 `NULL` 还有`'\0'`三者数值上一样
## 5.1 常量字符串
### 5.1.1 定义
- 定义在双引号中：`"string"`
- 本质就是字符数组，该字符串就是数组名: `"string"[0]`
- 自带 `'\0'` 结尾
	- 可访问 `"string"[6]`, `sizeof` 计算时也会计算 `'\0'` 的大小
- 只读
```c
char ch1[10] = {"hello"}; // 长度过长会出现乱码
char ch2[10] = "hello";
char ch3[] = "hello";
```
### 5.1.2 字符串指针
```c
char str = "hello"; 
str[2] = 'W'; // 复制了一份，可用修改
char *str = "hello c3"; // 直接指向地址，不能修改数组中的值
const char* str = "hello"; // 标砖写法加上 const 修饰
// 两者均从首地址开始，输出到 '\0' 结束
// 如果字符串中间有 '\0' 打印到中间就结束
printf("%s", str);
puts(str); // puts() 专门用于打印
```
### 5.1.3 字符串输出
- `%s`:遇到 `'\0'` 时才会停止
		- 如果是普通的字符数组, 如 `{'A', 'b', 'C'}`，就会一直向后输出，产生乱码
		- UTF-8 中用 `0xEFBFBD` 显示乱码，转 `GBK` 就会成为锟 `(0xEFBF)`, 斤 `(0xBDEF)`, 拷 `(0xBFBD)`
		- VC 中的乱码是烫 `(0xCC)` 和屯 `(0xCD)`
- `printf()` 也可以直接传入字符串输出 `printf(ch)`
- `puts()`
	- `int __cdecl puts(char const* _Buffer); `
	- 会添加 `'\n'`
```c
char ch[10] = "hell\0o world"; // 也可以通过 malloc 分配
printf(ch + 2) // 输出 ll
puts(ch + 1) // 输出 ell
printf(ch + 4) // 输出 o world
```
### 5.1.4 字符串输入
- `scanf_s()`
	- `%s` 会将空格作为分隔符，获取不到
	- 输入 `"hello world"` 只会输出 `hello`
- `gets_s()` 可以输入空格，字符串专用
	- `char* __cdecl gets_s(char* _Buffer, rsize_t _Size) `
	- 将传进来的字符保存在 `_Buffer` 中
### 5.1.5 字符串常用函数
- 引入 `<string.h>`  
```c
// 将字符串存储在 _Destination 中
char *__cdecl strcpy(char *_Destination, const char *_Source)

errno_t __cdecl strcpy_s(char *_Destination, rsize_t _SizeInBytes, const char *_Source)

// 拷贝字符串前 n 个字符
char *__cdecl strncpy(char *_Destination, const char *_Source, size_t _Count)

errno_t __cdecl strncpy_s(char *_Destination, rsize_t _SizeInBytes, const char *_Source, rsize_t _MaxCount)

strncpy(str, "hello world", 3) // 不会添加 '\0'，输出 hel烫烫...
strncpy(str, 5, "hello world", 3) // 添加 '\0'，正常输出

// 字符串拼接
char *__cdecl strcat(char *_Destination, const char *_Source)

errno_t __cdecl strcat_s(char *_Destination, rsize_t _SizeInBytes, const char *_Source)

// 字符串拼接 n 个 
// 将 src 中前 _Count 个字符拼接到 _Destination的尾部
char *__cdecl strncat(char *_Destination, const char *_Source, size_t _Count) // 自动加 '\0'

errno_t __cdecl strncat_s(char *_Destination, rsize_t _SizeInBytes, const char *_Source, rsize_t _MaxCount)

// 字符串比较
int __cdecl strcmp(const char *_Str1, const char *_Str2)

int __cdecl strncmp(const char *_Str1, const char *_Str2, size_t _MaxCount)

// 字符串长度
size_t __cdecl strlen(const char *_Str)

// strlen 不计算 '\0', sizeof 会计算 '\0'
printf("%d %d", strlen("abc"), sizeof("abc")); // 3 4

// strlen 遇到 '\0' 终止
printf("%d %d", strlen("abc\0def"), sizeof("abc\0def")); // 3 8(两个'\0')
```
# 6 其他类型
## 6.1 结构体
```c
// 定义在 main() 外
struct structName
{
	char field1;
	int field2;
	double field3;
	...
};

int main(void) 
{
	// c 需要写 struct 关键字
	struct structName struct1;
	struct structName struct2 = {"name", 10, 20.0};
	struct structName *p = &struct2;
	struct structName *p2 = (struct structName*)
						malloc(sizeof(struct structName));
}
```
- 访问结构体元素
	- 普通变量用 `'.'`
	- 指针变量用 `'->'`
```c
struct Stu stu = { "lisi", 20, 70.0 };
struct Stu* stu_p = &stu;

printf("%s %d %lf\n", stu.name, stu.age, stu.score);
printf("%s %d %lf\n", stu_p->name, stu_p->age, stu_p->score);
// 相当于
printf("%s %d %lf\n", (&stu)->name, (*stu_p).age, stu.score);

// 赋值
strcpy_s(stu.name, 20, "zhangsan"); // str 必须使用 strcpy_s() 赋值
stu_p->age = 20;
// 相同结构体之间可以直接赋值
stu1 = stu2;
*stu_p1 = stu2;
// 使用复合文字赋值
stu3 = (struct Stu){"", , }
```
- 匿名结构体
```c
// 此时变量只能紧跟在结构体后定义
struct
{...
} s;
```
- 特殊结构体成员
```c
// 嵌套结构体
struct Stu stu = { "lisi", 20,  (struct Teacher) {"wang", 30 }};

// 指针成员
struct Stu
{
	char name[5];
	int age;
	struct Stu* p;
};

// 函数指针成员
void eat(void) {}
struct Stu
{
	char name[5];
	int age;
	void (*p)(void);
};
// 通过 stu.p() 调用 eat()
```
- 结构体大小计算
	- 字节对齐
	- 将变量存储的首地址按 1, 2, 4, 8 字节对齐
		- 整体对齐：以最大类型的字节数为对齐字节数，成员按顺序填充
		- 局部对齐：填充时，与前面已分配好的成员，最大字节对齐
		- 结尾补齐：补齐最大字节数，最终为最大字节的整数倍
	- `#pragma pack(n)` 关键字
		- 按照 n 字节对齐
		- 括号内不填恢复默认对齐方式
## 6.2 联合体
- 所有成员共用一块空间，起始地址一样
```c
union Uni
{
	int a;
	short b;
	char c;
};

union Uni u = {233}; // 只能初始化第一个数据
// 修改一个数据，其他数据也会发生变化
printf("%d %d %c", u.a, u.b, u.c); // 233 233 ？
// 数值超出可表示范围时取余数
```
## 6.3 枚举
- 一组有名字的 int 类型数据的类型
```c
enum color {yellow, red, black, white, pink, blue};
int main(void) 
{
	// 默认从 0 开始对应
	printf("%d", yellow); // 0
	// 也可以给枚举变量赋值
	enum color co = black;
}

// 指定枚举变量的值
enum color {yellow=3, red, black, white=20, pink, blue};
// 没有值的接前边的进行增加 red -> 4, pink -> 21
```
# 7 I/O
- 打开文件：
	- `fopen()`
	- `fopen_s()`
- 读文件：
	- `fget()`
	- `fgets()`
	- `fscanf()`
	- `fread()`
- 写文件:
	- `fputc()`
	- `fputs()`
	- `fprintf()`
	- `fwrite()`
- 文件指针:
	- `fseek()`
	- `rewind()`
	- `ftell()`
- 关闭文件：`fclose()`
## 7.1 fopen() & fopen_s()
- `fopen()` 返回文件指针，`fopen_s()` 返回操作是否成功的状态码
- `FILE *__cdecl fopen(const char *_FileName, const char *_Mode)`
- `errno_t __cdecl fopen_s(FILE **_Stream, const char *_FileName, const char *_Mode)`
- `FILE *` 就是文件指针，打开文件的本质就是将文件内容放进缓冲区，`FILE *` 可以理解为文件缓冲区首地址
```c

FILE* pFile = fopen("hello.txt", "r");

FILE* pf = NULL;
// fopen_s 的第一个参数是二级指针
// corecrt.h -> typedef int errno_t
errno_t e = fopen_s(&pf, "hello.txt", "r");
printf("%d", e);
```
- 打开方式
- 文本模式
	- `"r/rt"` 只读
	- `r+` 可读可写，必须存在
	- `w/wt` 只写，不存在时会创建文件
	- `w+` 可读可写，不存在时创建
	- `a/at` 追加写，不存在时创建，指向尾字节
	- `a+` 可读可写，指向尾字节
- 二进制模式
	- `rb` 对应 `r/rt`
	- `rb+` 对应 `r+`
	- `wb` 对应 `w/wt`
	- `wb+` 对应 `w+`
	- `ab` 对应 `a/at`
	- `ab+` 对应 `a+`
- 区别
	- `win` 下行结尾是 `\r\n`, 文本模式读的是 `\n`, 二进制模式读的是 `\r\n`
	- `linux` 下没区别
## 7.2 fputc() & fgetc()
- `fputc()` 一次写入一个字符，返回字符的 `ascii` 码
- `fgetc()` 一次读取一个字符，返回字符的 `ascii` 码
- `int __cdecl fputc(int _Character, FILE *_Stream)`
- `int __cdecl fgetc(FILE *_Stream)`
```c
FILE* pf = NULL;
// 判断打开是否成功
// "w" 打开时就会清理掉之前的内容
if (fopen_s(&pf, "hello.txt", "w") != 0)
	return 0;
fputc('d', pf);
fputc('a', pf); 
fputc('\n', pf); 
fputc('d', pf);  // 输出 da /n d 
fclose(pf);

printf("%c", fgetc(pf)); // h
printf("%c", fgetc(pf)); // e
printf("%c", fgetc(pf)); // l
printf("%c", fgetc(pf)); // l
fclose(pf);
```
- 循环读
```c
while (1)
{
	int a = fgetc(pf);
	if (feof(pf))
		break;
	putchar(a);
}
```
## 7.3 fputs() & fgets()
- 读取或写入多个字符
- `int __cdecl fputs(const char *_Buffer, FILE *_Stream)`
- `char *__cdecl fgets(char *_Buffer, int _MaxCount, FILE *_Stream)`
- `fgets()` 会读 `_MaxCount - 1 ` 个字符，拼接 `'\0'` 输出
- `fputs()` 用于写入数据
```c
while (1)
{	
	fgets(str, 20, pf);
	printf(str);
	if (feof(pf))
		break;
}

fputs("hello scala", pf);
```
## 7.4 fprintf() & fscanf()
- 格式化读写
- `inline int __cdecl fprintf(FILE *const _Stream, const char *const _Format, ...)`
- `inline int __cdecl fscanf(FILE *const _Stream, const char *const _Format, ...)`
- `inline int __cdecl fscanf_s(FILE *const _Stream, const char *const _Format, ...)`
```c
// 格式化写
fprintf(pf, "a:%d,b%lf,s%s", 12, 34.5, "hello rr");
// 格式化读
fscanf_s(pf, "a:%d,b%lf,s%s", &a, &b, str, 20);
```
## 7.5 fread() & fwrite()
- 以二进制读写，不用转换类型，比前面几组效率高
- `size_t __cdecl fread(void *_Buffer, size_t _ElementSize, size_t _ElementCount, FILE *_Stream)`
- `size_t __cdecl fwrite(const void *_Buffer, size_t _ElementSize, size_t _ElementCount, FILE *_Stream)`
```c
// 可以直接写入结构体
struct Node n = { 12, "hello", 34.5 }, p;
int a = fwrite(&n, sizeof n, 1, pf);
// 读取
fread(&p, sizeof p, 1, pf);
// 写入的是转换后的数据
fprintf(pf, "%d,%lf,%s", n.a, n.b, n.str);
```
## 7.6 rewind() & ftell() & fseek()
- `void __cdecl rewind(FILE *_Stream)`
- `long __cdecl ftell(FILE *_Stream)`
- `int __cdecl fseek(FILE *_Stream, long _Offset, int _Origin)`
- 
```c
// rewind() 将文件指针指向文件首
putchar(fgetc(pf)); // a
putchar(fgetc(pf)); // b
rewind(pf);
putchar(fgetc(pf)); // a
putchar(fgetc(pf)); // b

// ftell() 返回文件指针指向的文件中的字节下标
putchar(fgetc(pf)); // a
printf("%ld", ftell(pf)); // 1
putchar(fgetc(pf)); // b
printf("%ld", ftell(pf)); // 2
rewind(pf); 
printf("%ld", ftell(pf)); // 0

// fseek() 设置文件指针指向哪个字节
// 指向 _Offset + _Origin 的位置
// SEEK_SET(首), SEEK_END, SEEK_CUR
putchar(fgetc(pf)); // a
putchar(fgetc(pf));// b
fseek(pf, 4, SEEK_SET); // 跳一个字符
putchar(fgetc(pf)); // e

putchar(fgetc(pf));
putchar(fgetc(pf));
fseek(pf, -1, SEEK_END); // 以 EOF 为0，向左移动到倒数第一个字符
putchar(fgetc(pf));

// 如果换行结尾，SEEK_END向回读时，行尾是'\r\n'
```
# 8 宏与存储类说明符
- `#include "xxx" <xxx>` 的区别
	- 标准库文件使用 `<>`, 自定义头文件使用 `""`
	- 查找的起始路径不一样
		- `""` 会先去工程文件所在路径查找，再去默认路径找
		- `<>` 会直接在默认路径找
	-  直接写头文件绝对路径的编译效率高 
- 防止头文件重复包含
	- `1.h` 中声明一个结构体 `Node`
	- `2.h` 引入 `1.h`
	- `main.c` 中引入 `1.h, 2.h` 时就会出现结构体重定义
		- 在 `1.h` 中加入 `#pragma once` 只编译一次(新标准)
		- 旧标准使用 `#ifndef + 标识`
			- 标识一般是 `_ + 头文件名大写`，将其中的 `.` 换成 `_`，例如 `_STDIO_H`
			- `#ifndef -> #define -> #else -> #endif`
## 8.1 宏
- 常量宏: 不参与计算，在预编译阶段进行替换
	- `#define N 12`
	- `#define M printf`
- 参数宏
	- `#define N(x, y) x + y`
## 8.2 存储类说明符
- `typedef` 类型重命名
	- `typedef int my_int;`
	- `typedef int* p_my_int;`
	- `typedef int arr[n];`
	- `typedef int(*parr)[3];`
	- `typedef int(*pfun)(int, double);`
- 全局变量 `extern`
	- 作用域是整个工程，只能定义在源文件中
	- 默认初始化为 `0`
	- 其他源文件需要使用时，先声明 `extern type name`，不能初始化，否则就会重定义
	- 全局变量与局部变量同名时，局部变量起作用
- `static`
	- 在全局变量前加 `static`
		- 作用域是所在文件，与程序共存亡
		- 不同文件定义重名的只在各自文件中使用的全局变量，就声明为 `static`
	- 在局部变量前加 `static`
		- 作用域是所在大括号，与程序共存亡
```c
void fun(void) 
{
	static int a = 1;
	a++;
}

fun() // a = 2
fun() // a = 3, 不会重新赋值
```
- `auto` 局部变量
	- 声明周期在大括号内
	- 局部变量自带 `auto` 关键字，可省略
	- 自动指空间自动申请释放，由 `OS` 管理
- `register`
	- 向编译器建议将数据存在寄存器中
		- 取决于 `OS` 的调度算法
		- 一般不存
	- 不能取地址，不能为静态
	- 用于提高程序的执行速度
- `const`
	- const 修饰全局变量放在全局区中
	- const 修饰局部变量
```c
int a = 12, b = 56;
const int* p1 = &a; // 不能修改值，可以更改指向位置
int* const p2 = &a; // 可以修改值，不能更改指向位置
const int* const p3 = &a; // 都不能修改
```
	