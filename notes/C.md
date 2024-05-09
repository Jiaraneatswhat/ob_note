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