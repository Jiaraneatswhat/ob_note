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
### 1.1.5 bool
```cpp
bool f1 = true;
bool f2 = false;
cout << f1 << endl; // 1
cout << f2 << endl; // 0
cout << sizeof f1 << endl // 1
```
### 1.1.6 cin
```cpp
int a = 0;
cout << "input a number: " << endl;
cin >> a;
cout << a << endl;
```
# 2 面向对象
## 2.1 new 操作符
- `c++` 利用 `new` 在堆中开辟数据，返回一个指针
- 删除使用 `delete`
```cpp
int *arr = new int[10];
delete[] arr // 释放数组需要[]
```
- new 和 malloc()
	- `new` 是关键字，`malloc()` 是函数
	- `new` 申请内存无需指定大小，`new` 还会调用构造函数
		- `malloc()` 需要显式指出所需内存
	- `new` 返回的是对象类型指针，无需转换
		- `malloc()` 返回的是 `void*`，需要强转
	- `new` 会调用 `operator new` 函数，可以重载，`malloc()` 不能重载
	- `new` 在自由存储区分配内存，`c++` 默认使用堆来实现自由存储
	- `new` 的效率比 `malloc()` 高
## 2.2 引用
- 给变量起别名
```cpp
int a = 10;
int &b = a;
b = 20;
cout << b << endl; // 给内存起的别名
-------------
int &d; // 引用必须初始化
int c = 20;
b = c; // 引用初始化后再赋值不会更改引用位置
```
- 引用做函数参数
```cpp
// 简化指针，用引用实现地址传递
void swap(int &a, int &b)
{
	int tmp = a;
	a = b;
	b = tmp;
}

int main()
{
	swap(a, b) // a的别名就是 swap() 中的 a  
}
```
- 引用做函数返回值
```cpp
int &test01()
{
	int a = 10;
	return a;
}
int& test02()
{
	static int a = 10;
	return a;
}

int main()
{
	int &ref = test01();
	cout << ref << endl;
	// 不要返回局部变量的引用
	cout << ref << endl; // 第二次返回时结果会出错
	system("pause");

	int &ref = test02();
	cout << ref << endl;
	cout << ref << endl; // 加了 static 后可以多次返回
	test02() = 1000; // 函数的调用可以做为左值
	cout << ref << endl; // 1000
}
```
- 引用的本质 - 指针常量
```cpp
void func(int& ref)
{
	ref = 100;
}

int main()
{
	int a = 10;
	int &ref = a; // 编译器会转为 int* const ref = &a; 
	ref = 20; // *ref = 20;
	func(a);
}
```
- 常量引用
```cpp
// 加上 const 后编译器会自动创建大小为 10 的元素
const int &ref = 10;
ref = 30; // 报错，加了 const 只读

// 加上 const 防止数据被修改
void func(const int &val) ...
```
## 2.3 函数重载
- 函数的占位参数
```cpp
void func(int a , int) ...

// 占位参数默认值
void func(int a, double = 1.1) ...
```
- 重载基本条件与 `java` 相同
- 引用作为重载
```cpp
void func(int &a)...
void func(const int &a)...

int main()
{
	int a = 10;
	func(a); // a 是变量，调用 func(int &a)
	func(20); // 直接传常量调用 func(const int &a)
}
```
- 默认参数不能重载
```cpp
void func(int a)...
void func(int a, int b = 10)...

int main()
{
	func(10) // 调用时有歧义 编译错误
}
```
## 2.4 类和对象
### 2.4.1 封装
```cpp
// 定义一个 Circle 类
class Circle
{
public:
	int radius;
	double get_perimeter()
	{
		return 2 * PI * radius;
	}
};

int main()
{
	Circle c1;
	c1.radius = 10.0;
	cout << "perimeter = " << c1.get_perimeter() << endl;
	system("pause");
}
```
- `c++` 有三种访问权限
	- `public` 
	- `protected` 和 `private` 类内可以访问，类外不可以访问
	- 子类可以访问父类的 `protected` 成员，`private` 不行
- `struct` 和 `class` 的区别
	- `struct` 默认的访问权限是 `public`
	- `class` 默认的访问权限是 `private`
- 