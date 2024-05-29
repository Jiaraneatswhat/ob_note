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
### 2.4.2 对象的初始化和清理
#### 2.4.2.1 构造函数和析构函数
- 析构函数：主要作用于在对象销毁前系统自动调用，执行清理工作
- 构造函数 `class_name(){}`
	- 无返回值，不写 `void`
	- 可以重载
	- 无需手动调用，只调用一次
- 析构函数 `~class_name(){}`
	- 没有返回值，不写 `void`
	- 没有参数，不能重载
	- 无需手动调用，只调用一次
```cpp
Circle(double r) 
{
	radius = r;
}
~Circle() 
{
	cout << "析构函数被调用" << endl;
}
```
#### 2.4.2.2 构造函数分类和调用
- 构造函数的分类
	- 有参/无参构造
	- 普通/拷贝构造
- 构造函数的调用方式
	- 括号
	- 显式
	- 隐式转换
```cpp
// 拷贝构造函数
// 需要加 const，传引用, 复制生成一个相同属性的对象
Circle(const Circle &c)
{
	radius = c.radius;
}

int main()
{   
	// 括号法
	Circle c1; // 无参构造不能加 (), 编译器会认为是函数声明
	Circle c2(10.0); // 有参构造
	Circle c3(c2); // 拷贝构造

	// 显式法类似 java
	Circle c = Circle(10.0);

	// 不能用拷贝构造函数初始化匿名对象，会报重定义
	Circle(c1);

	// 隐式转换法
	Circle c4 = 10.0;
}
```
- 拷贝构造函数调用时机
	- 使用一个创建完毕的对象初始化一个新对象
	- 值传递的方式给函数参数传值
	- 以值方式返回局部对象
```cpp
double get_area(Circle c)
{	
	return PI * c.radius * c.radius;
}

Circle get_circle() 
{
	Circle c;
	return c;
}

void test()
{
	Circle c;
	// c 作为形参传入时会调用拷贝构造函数
	get_area(c);
	// get_circle 返回值调用拷贝构造函数
	Circle c_ = get_circle();
}
```
- 定义了有参构造，`c++` 会提供默认拷贝构造
- 定义拷贝构造，`c++` 不提供其他构造函数
#### 2.4.2.3 深拷贝与浅拷贝
- 浅拷贝：简单的赋值拷贝
- 深拷贝：堆区重新申请空间进行拷贝
```cpp
// 浅拷贝会导致堆区内存重复释放
// 在 Circle 类中新增一个属性 int *number
Circle(double r, int num) {
	radius = r;
	// 将 number 声明在堆区
	number = new int(num);
}
// 更新析构函数内容
~Circle() 
{	
	if (number)
	{
		delete number;
		number = NULL;
	}
}
// 若不自定义拷贝构造函数，调用系统的做浅拷贝操作
// 调用析构函数时释放内存，另一个对象在释放时就会报错
// 自己实现一个拷贝构造函数，进行深拷贝
Circle(const Circle& c)
{
	radius = c.radius;
	// 开辟一个新空间
	number = new int(*c.number);
}
```
#### 2.4.2.4 初始化列表
- 用于初始化属性
```cpp
class Circle
{
public:
	double radius;
	int number;
	string color;

	Circle(...): radius(10.0), number(20), color("red") {...}
};
```
#### 2.4.2.5 静态成员
- 在成员变量和成员函数前加上 `static`
- 静态成员变量
	- 所有对象共享统一份数据
	- 在编译阶段分配内存
	- 类内声明，类外初始化
- 静态成员函数
	- 所有对象共享同一个函数
	- 静态成员函数不能访问非静态成员变量
```cpp
class Person
{
public:
	static int age;
	// 静态函数
	static void func()
	{
		cout << "static func..." << endl;
	}
private:
	// 静态变量也有访问权限
	static int num;
};

// 类外初始化
int Person::age = 10;
int Person::num = 20;

void test()
{
	Person p;
	cout << p.age << endl;
	cout << p.num << endl; // 访问不到

	Person p1;
	p1.age = 20; // p1 修改数据后，p 也会变
	// 两种访问方式
	cout << p.age << endl;
	cout << Person::age << endl;

	p.func();
	Person::func();
}
```
### 2.4.3 对象模型
#### 2.4.3.1 成员变量和成员函数的存储
- 类内成员变量和成员函数分开存储
- 只有非静态成员变量才属于类的对象
```cpp
// 空对象占一个字节
// 添加了非静态属性后，按属性的大小来算
class Person 
{
	int a;
	static b; // 不属于对象
	void func1() {...} // 不属于对象
	static void func2() {...} // 不属于对象
}
```
#### 2.4.3.2 this 指针
- 本质是指针常量，不能修改指向 type* const p
```cpp
class Person
{
public:
	int age;
	Person(int age)
	{
		this->age = age;
	}
	// 返回引用避免值传递
	Person& add_age(Person p)
	{
		this->age += p.age;
		// 返回对象本身用 *this
		return *this;
	}
	void show_class()
	{
		cout << "Person" << endl;
	}
	void show_age()
	{
		cout << age << endl;
	}
};
// 空指针可以访问成员，确保 this 不为 NULL 即可
void test()
{
	Person *p = NULL;
	p->show_class(); // 正常运行
	p->show_age(); // age 前默认有 this，会报空指针
}
```
#### 2.4.3.3 const 修饰成员函数
- 常函数
	- 常函数内不可修改成员属性
	- 成员属性声明时加 `mutable` 关键字，在常函数中可以修改
- 常对象只能调用常函数
```cpp
class Person
{
public:
	int age;
	mutable int num;

	void show_person() const
	{	
		// 函数不加 const 时，this 指针相当于 Person * const this
		// 函数加了 const 后，相当于 const Person * const this
		// 指针指向的值也不能修改
		age = 20; // 不能修改
		num = 20; // 加了 mutable 后可以修改
		cout << age << endl;
	}
	
};
```