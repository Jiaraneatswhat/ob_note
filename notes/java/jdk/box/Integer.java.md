- 类定义
```java
// since 1.0
public final class Integer extends Number implements Comparable<Integer>{...}
```
- `Number` 是数值型的抽象父类
```java
public abstract class Number implements java.io.Serializable {

	// 将数据转为 int 类型
	public abstract int intValue();

	... // 与之类似，还有 longValue(), floatValue() 等方法
}
```
# 1 fields
```java
// 转二进制是 1000 0000 0000 0000 0000 0000 0000 0000
// 高位 1 表示负数，-2^31
@Native public static final int   MIN_VALUE = 0x80000000;

// 0111 1111... 2^31 - 1
@Native public static final int   MAX_VALUE = 0x7fffffff;

// int 对应的 JVM 中的 Class 对象
public static final Class<Integer>  TYPE = (Class<Integer>) Class.getPrimitiveClass("int");

// 所有可以表示数字的字符
// 用于进制转换，int 支持二进制到 36 进制，因此包括 0-9 以及 a-z
final static char[] digits = {  
    '0' , '1' , '2' , '3' , '4' , '5' ,  
    '6' , '7' , '8' , '9' , 'a' , 'b' ,  
    'c' , 'd' , 'e' , 'f' , 'g' , 'h' ,  
    'i' , 'j' , 'k' , 'l' , 'm' , 'n' ,  
    'o' , 'p' , 'q' , 'r' , 's' , 't' ,  
    'u' , 'v' , 'w' , 'x' , 'y' , 'z'  
};

// 用于计算 Integer 转 String 后字符串长度
final static int [] sizeTable = { 9, 99, 999, 9999, 99999, 999999, 9999999, 99999999, 999999999, Integer.MAX_VALUE };

// DigitOnes 和 DigitTens 用于将数字转为 String
// 十位数
final static char [] DigitTens = {  
    '0', '0', '0', '0', '0', '0', '0', '0', '0', '0',  
    '1', '1', '1', '1', '1', '1', '1', '1', '1', '1',  
    '2', '2', '2', '2', '2', '2', '2', '2', '2', '2',  
    '3', '3', '3', '3', '3', '3', '3', '3', '3', '3',  
    '4', '4', '4', '4', '4', '4', '4', '4', '4', '4',  
    '5', '5', '5', '5', '5', '5', '5', '5', '5', '5',  
    '6', '6', '6', '6', '6', '6', '6', '6', '6', '6',  
    '7', '7', '7', '7', '7', '7', '7', '7', '7', '7',  
    '8', '8', '8', '8', '8', '8', '8', '8', '8', '8',  
    '9', '9', '9', '9', '9', '9', '9', '9', '9', '9',  
    } ;  

// 个位数
final static char [] DigitOnes = {  
    '0', '1', '2', '3', '4', '5', '6', '7', '8', '9',  
    '0', '1', '2', '3', '4', '5', '6', '7', '8', '9',  
    '0', '1', '2', '3', '4', '5', '6', '7', '8', '9',  
    '0', '1', '2', '3', '4', '5', '6', '7', '8', '9',  
    '0', '1', '2', '3', '4', '5', '6', '7', '8', '9',  
    '0', '1', '2', '3', '4', '5', '6', '7', '8', '9',  
    '0', '1', '2', '3', '4', '5', '6', '7', '8', '9',  
    '0', '1', '2', '3', '4', '5', '6', '7', '8', '9',  
    '0', '1', '2', '3', '4', '5', '6', '7', '8', '9',  
    '0', '1', '2', '3', '4', '5', '6', '7', '8', '9',  
    } ;

// 存放数字
private final int value;

// int 的字节数
@Native public static final int SIZE = 32;

public static final int BYTES = SIZE / Byte.SIZE;
```
# 2 constructor
```java
public Integer(int value) {  
    this.value = value;  
}

public Integer(String s) throws NumberFormatException {  
    this.value = parseInt(s, 10);  
}
```
# 3 methods

![[integer_methods.jpg]]
## 3.1 toString 类
### 3.1.1 getChars()
```java
// 将数字转为字符存在 buf 中
static void getChars(int i, int index, char[] buf) {  
    int q, r;  
    int charPos = index;  
    char sign = 0;  
  
    if (i < 0) {  
        sign = '-';  
        i = -i;  
    }  
  
    // Generate two digits per iteration  
    // 每次循环取两位
    while (i >= 65536) {  
        q = i / 100;  
    // really: r = i - (q * 100);  
        r = i - ((q << 6) + (q << 5) + (q << 2));  
        i = q;  
        buf [--charPos] = DigitOnes[r];  
        buf [--charPos] = DigitTens[r];  
    }  

	// 每次循环取一位
    // Fall thru to fast mode for smaller numbers  
    // assert(i <= 65536, i);    
    for (;;) {  
	    // 54249 >>> 19 = 0.1xxxxxx
        q = (i * 52429) >>> (16+3);  
        r = i - ((q << 3) + (q << 1));  // r = i-(q*10) ...  
        buf [--charPos] = digits [r];  
        i = q;  
        if (i == 0) break;  
    }  
    if (sign != 0) {  
        buf [--charPos] = sign;  
    }  
}
```
### 3.1.2 stringSize()
```java
static int stringSize(int x) {  
    for (int i=0; ; i++) 
	    // 循环比较 
        if (x <= sizeTable[i])  
            return i+1;  
}
```
### 3.1.3 toString()
```java
public String toString() {  
    return toString(value);  
}

public static String toString(int i) {  
    if (i == Integer.MIN_VALUE)  
        return "-2147483648";  
    int size = (i < 0) ? stringSize(-i) + 1 : stringSize(i);  
    char[] buf = new char[size];  
    getChars(i, size, buf);  
    return new String(buf, true);  
}


```
