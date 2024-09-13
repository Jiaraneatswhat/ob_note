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
@Native public static final int MIN_VALUE = 0x80000000;

// 0111 1111... 2^31 - 1
@Native public static final int MAX_VALUE = 0x7fffffff;

// int 对应的 JVM 中的 Class 对象
public static final Class<Integer> TYPE = (Class<Integer>) Class.getPrimitiveClass("int");

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

// 存放对应的 int 值
private final int value;

// int 的 bit 数
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
# 3 inner class
```java
private static class IntegerCache {  
    static final int low = -128;  
    static final int high;  
    static final Integer cache[];  
  
    static {  
        // high value may be configured by property  
        int h = 127;  
        String integerCacheHighPropValue = sun.misc.VM.getSavedProperty("java.lang.Integer.IntegerCache.high");  
        if (integerCacheHighPropValue != null) {  
        // 如果从配置中读取到了值
            try {  
                int i = parseInt(integerCacheHighPropValue);  
                // 保证上限 h 不低于 127
                i = Math.max(i, 127);  
                // Maximum array size is Integer.MAX_VALUE  
                // 不超过最大数组大小
                h = Math.min(i, Integer.MAX_VALUE - (-low) -1);  
            } catch( NumberFormatException nfe) {  
                // If the property cannot be parsed into an int, ignore it.  
            }  
        }  
        high = h;  
        cache = new Integer[(high - low) + 1];  
        int j = low;  
        for(int k = 0; k < cache.length; k++)  
            cache[k] = new Integer(j++);  
  
        // range [-128, 127] must be interned (JLS7 5.1.7)  
        assert IntegerCache.high >= 127;  
    }  
  
    private IntegerCache() {}  
}
```
# 4 methods

![[integer_methods.jpg]]
## 4.1 toString 类
### 4.1.1 getChars()
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
        // 99 为例, DigitOnes 和 DigitTens 中的第 99 个元素都是 9
        buf [--charPos] = DigitOnes[r];  
        buf [--charPos] = DigitTens[r];  
    }  

	// 每次循环取一位
    // Fall thru to fast mode for smaller numbers  
    assert(i <= 65536, i);    
    for (;;) {  
	    /**
         * 6554 / 2^16 = 0.100006103515625
         * 13108 / 2^17 = 0.1000006103515625
         * 26215 / 2^18 = 0.100000228881835938
         * 54249 / 2^19 = 0.10000038146972656 -> 从精度上考虑 19 最合适
         * 104858 / 2^20 = 0.1000003815
	    */
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
### 4.1.2 stringSize()
```java
static int stringSize(int x) {  
    for (int i=0; ; i++) 
	    // 循环比较 
        if (x <= sizeTable[i])  
            return i+1;  
}
```
### 4.1.3 toString()
```java
public String toString() {
	// 调用自身静态的 toString 方法
    return toString(value);  
}

public static String toString(int i) {  
	// 等于最小值时直接返回对应的字符串
    if (i == Integer.MIN_VALUE)  
        return "-2147483648";
    // 计算 size 创建 buf  
    int size = (i < 0) ? stringSize(-i) + 1 : stringSize(i);  
    char[] buf = new char[size];  
    getChars(i, size, buf);
    // 填充 buf 后调构造器创建字符串  
    return new String(buf, true);  
}

// 指定进制数时
public static String toString(int i, int radix) {  
	// 判断 radix 是否属于 [2, 32]
    if (radix < Character.MIN_RADIX || radix > Character.MAX_RADIX)  
        radix = 10;  
  
    /* Use the faster version */  
    // 十进制下调普通的 toString
    if (radix == 10) {  
        return toString(i);  
    }  
	// 32 位 + 符号位
    char buf[] = new char[33];  
    boolean negative = (i < 0);  
    int charPos = 32;  

	// 转为负数防止溢出
    if (!negative) {  
        i = -i;  
    }  
  
    while (i <= -radix) {  
	    // 从 digits 数组中取出对应字符
        buf[charPos--] = digits[-(i % radix)];  
        i = i / radix;  
    }  
    buf[charPos] = digits[-i];  
  
    if (negative) {  
        buf[--charPos] = '-';  
    }  
  
    return new String(buf, charPos, (33 - charPos));  
}
```
### 4.1.4 numberOfLeading(Trailing)Zeros()
```java
// 计算最高位 1 前 0 的个数
public static int numberOfLeadingZeros(int i) {  
    // HD, Figure 5-6  
    // 全部为 0 返回 32
    if (i == 0)  
        return 32; 
    // 至少有 1 个 0 
    int n = 1;  
    /**
	  分组计算 0 的个数:
	  
      00000000 00000000 00000000 00000001 
      右移 16 位为 0, 说明最高两个字节全为 0，让 n += 16，左移 16 位
      00000000 00000001 00000000 00000000
      右移 24 位为 0，说明第三个字节也为 0，让 n += 8，左移 8 位
      00000001 00000000 00000000 00000000
      右移 28 位为 0，说明最后一个字节的高四位也为 0，让 n += 4，左移 4 位
      00010000 00000000 00000000 00000000
      右移 30 位为 0，n += 2, 左移两位
      01000000 00000000 00000000 00000000
      
      剩下的两位有三种情况: 01, 10, 11
      01 的情况下，右移 31 位等于 0, 此时 n 不变
      1x 的情况下, 右移 31 位得到 1，n -= 1
    */
    if (i >>> 16 == 0) { n += 16; i <<= 16; }  
    if (i >>> 24 == 0) { n +=  8; i <<=  8; }  
    if (i >>> 28 == 0) { n +=  4; i <<=  4; }  
    if (i >>> 30 == 0) { n +=  2; i <<=  2; }  
    n -= i >>> 31;  
    return n;  
}

// 同上
public static int numberOfTrailingZeros(int i) {  
    // HD, Figure 5-14  
    int y;  
    if (i == 0) return 32;  
    int n = 31;  
    y = i <<16; if (y != 0) { n = n - 16; i = y; }  
    y = i << 8; if (y != 0) { n = n - 8; i = y; }  
    y = i << 4; if (y != 0) { n = n - 4; i = y; }  
    y = i << 2; if (y != 0) { n = n - 2; i = y; }  
    return n - ((i << 1) >>> 31);  
}
```
### 4.1.5 formatUnsignedInt()
```java
// shift 是进制数以 2 为底的对数
static int formatUnsignedInt(int val, int shift, char[] buf, int offset, int len) {  
    int charPos = len;  
    int radix = 1 << shift;  
    int mask = radix - 1;  
    do {  
	    // 通过与 mask 进行与运算，将 val 按 shift 进行分组，每次取出对应的字符后移位，处理下一组
        buf[offset + --charPos] = Integer.digits[val & mask];  
        val >>>= shift;  
    } while (val != 0 && charPos > 0);  
  
    return charPos;  
}
```
### 4.1.6 toUnsignegString0()
```java
private static String toUnsignedString0(int val, int shift) {  
    // assert shift > 0 && shift <=5 : "Illegal shift value";  
    // 32 - 0 的个数得到有效位数 mag
    int mag = Integer.SIZE - Integer.numberOfLeadingZeros(val);  
    int chars = Math.max(((mag + (shift - 1)) / shift), 1);  
    char[] buf = new char[chars];  
  
    formatUnsignedInt(val, shift, buf, 0, chars);  
  
    // Use special constructor which takes over "buf".  
    return new String(buf, true);  
}
```
### 4.1.7 toBinary(Octal, Hex)String
```java
// 仅传入的 shift 不同
public static String toBinaryString(int i) {  
    return toUnsignedString0(i, 1);  
}

public static String toOctalString(int i) {  
    return toUnsignedString0(i, 3);  
}

public static String toHexString(int i) {  
    return toUnsignedString0(i, 4);  
}
```
### 4.1.8 toUnsignedString()
```java
public static String toUnsignedString(int i) {  
    return Long.toString(toUnsignedLong(i));  
}

public static String toUnsignedString(int i, int radix) {  
    return Long.toUnsignedString(toUnsignedLong(i), radix);  
}

public static long toUnsignedLong(int x) {  
    return ((long) x) & 0xffffffffL;  
}
```
## 4.2 parse 类
### 4.2.1 parseInt()
```java
public static int parseInt(String s) throws NumberFormatException {  
	// 不传进制数，默认 10
    return parseInt(s,10);  
}

public static int parseInt(String s, int radix) throws NumberFormatException  
{  
    // 判空
    if (s == null) {...}  
	// 进制数合法性
    if (radix < Character.MIN_RADIX) {...}  
    if (radix > Character.MAX_RADIX) {...}  
  
    int result = 0;  
    boolean negative = false;  
    int i = 0, len = s.length();  
    int limit = -Integer.MAX_VALUE;  
    int multmin;  
    int digit;  
  
    if (len > 0) {  
        char firstChar = s.charAt(0);  
        // '+' 对应 ascii 43, '-' 对应 45, '0' 对应 48
        // 判断是否为 '+', ‘-’
        if (firstChar < '0') { // Possible leading "+" or "-"  
            if (firstChar == '-') {  
                negative = true;  
                limit = Integer.MIN_VALUE;  
            } else if (firstChar != '+')  
                throw NumberFormatException.forInputString(s);  
  
            if (len == 1) // 不能仅为一个正负号
                throw NumberFormatException.forInputString(s); 
            // 开始处理第一个字符 
            i++;  
        }  
        multmin = limit / radix;  
        while (i < len) {  
            // Accumulating negatively avoids surprises near MAX_VALUE  
            digit = Character.digit(s.charAt(i++),radix);  
            // 结果是否超出范围
            if (digit < 0) {...}  
            if (result < multmin) {...} 
            // 百位数 * 10 * 10 + 十位数 * 10 
            result *= radix;  
            if (result < limit + digit) {...}  
            // 个位数
            result -= digit;  
        }  
    } else {// 长度不合法抛异常
    }  
    return negative ? result : -result;  
}
```
### 4.2.2 parseUnsignedInt()
```java
public static int parseUnsignedInt(String s) throws NumberFormatException {  
    return parseUnsignedInt(s, 10);  
}

public static int parseUnsignedInt(String s, int radix)  
            throws NumberFormatException {  
    if (s == null)  {...}  
  
    int len = s.length();  
    if (len > 0) {  
        char firstChar = s.charAt(0);  
        if (firstChar == '-') {  
	        // 无符号数不带符号，抛异常     
        } else {  
	        // 在有符号 int 的范围内，调有符号的 parse 方法
            if (len <= 5 || // Integer.MAX_VALUE in Character.MAX_RADIX is 6 digits  
                (radix == 10 && len <= 9) ) { // Integer.MAX_VALUE in base 10 is 10 digits  
                return parseInt(s, radix);  
            } else {  
	            // 作为 long 进行解析
                long ell = Long.parseLong(s, radix);  
                if ((ell & 0xffff_ffff_0000_0000L) == 0) {  
	                // 不超过 uint 的最大值时强转为 int 返回
                    return (int) ell;  
                } else {  
                    throw new ...}  
            }  
        }  
    } else {...}  
}
```
## 4.3 位运算类
### 4.3.1 bitCount()
- 对两位的二进制数来说，用自身减去最高位的数后得到一的个数
	 ${\color{Red} 0} 0\to  00-00=00$ 没有 1
	 ${\color{Red} 0} 1\to  01-00=01$ 一个 1
	 ${\color{Red} 1} 0\to  10-01=01$ 一个 1
	 ${\color{Red} 1} 1\to  11-01=10$ 两个 1
```java
public static int bitCount(int i) {  
	/**
	  统计二进制中 1 的个数
	  0x55555555 -> 0101 0101 0101 0101 0101 0101 0101 0101
	  0x33333333 -> 0011 0011 0011 0011 0011 0011 0011 0011
	  0x0f0f0f0f -> 0000 1111 0000 1111 0000 1111 0000 1111
	  0x3f -> 0000 0000 0000 1111 0000 1111 0000 1111 
	*/
    // HD, Figure 5-2  
    // 分组计算 1 的个数，通过与运算屏蔽掉移位后的高位数字
    i = i - ((i >>> 1) & 0x55555555);  
    i = (i & 0x33333333) + ((i >>> 2) & 0x33333333);  
    i = (i + (i >>> 4)) & 0x0f0f0f0f;  
    i = i + (i >>> 8);  
    i = i + (i >>> 16);  
    return i & 0x3f;  
}
```
### 4.3.2 signum()
```java
public static int signum(int i) {  
    // HD, Section 2-7  
    // 位运算操作的是补码，负数取反加一得到补码
    // 负数右移补 1
    return (i >> 31) | (-i >>> 31);  
}
```
### 4.3.3 reverse()
```java
// 将第 i 位和第 33 - i 位进行互换
public static int reverse(int i) {  
    // HD, Figure 7-1  
    // 先交换相邻两位
    i = (i & 0x55555555) << 1 | (i >>> 1) & 0x55555555;  
    // 相邻四位
    i = (i & 0x33333333) << 2 | (i >>> 2) & 0x33333333;  
    // 相邻八位
    i = (i & 0x0f0f0f0f) << 4 | (i >>> 4) & 0x0f0f0f0f;  
    // 0xff00 -> 00000000 00000000 11111111 00000000
    i = (i << 24) | ((i & 0xff00) << 8) |  
        ((i >>> 8) & 0xff00) | (i >>> 24);  
    return i;  
}
```
### 4.3.4 reverseBytes()
```java
// 以字节为单位进行反转
public static int reverseBytes(int i) {  
    return ((i >>> 24)) |  
           ((i >> 8) & 0xFF00) |  
           ((i << 8) & 0xFF0000) |  
           ((i << 24));  
}
```
### 4.3.5 highestOneBit()
```java
public static int highestOneBit(int i) {  
	// 返回最高位的 1, 其他全为 0 的值 
    // HD, Figure 3-1  
    /**
	    1001 0000 | 0100 1000 -> 1101 1000
	    1101 1000 | 0011 0110 -> 1111 1110
	    1111 1110 | 0000 1111 -> 1111 1111
	    ...
	    1111 1111 - 0111 1111 = 1000 0000
    */
    i |= (i >>  1);  
    i |= (i >>  2);  
    i |= (i >>  4);  
    i |= (i >>  8);  
    i |= (i >> 16);  
    return i - (i >>> 1);  
}
```
### 4.3.6 lowestOneBit()
```java
public static int lowestOneBit(int i) {  
    // HD, Section 2-1  
    /**
		0011 & 11111111 11111111 11111111 00001101 = 0...0001
    */
    return i & -i;  
}
```
### 4.3.7 rotateLeft(Right)()
```java
// 循环移位
public static int rotateLeft(int i, int distance) {  
	// >>> -a = >>> 32 - a
    return (i << distance) | (i >>> -distance);  
}

public static int rotateRight(int i, int distance) {  
    return (i >>> distance) | (i << -distance);  
}
```
## 4.4 获取 Integer 相关
### 4.4.1 valueOf()
```java
public static Integer valueOf(String s, int radix) throws NumberFormatException {  
    return Integer.valueOf(parseInt(s,radix));  
}

public static Integer valueOf(String s) throws NumberFormatException {  
    return Integer.valueOf(parseInt(s, 10));  
}

// 对于不超出范围的数直接从 Cache 中取出对应对象, 否则创建一个新的对象
public static Integer valueOf(int i) {  
    if (i >= IntegerCache.low && i <= IntegerCache.high)  
        return IntegerCache.cache[i + (-IntegerCache.low)];  
    return new Integer(i);  
}
```
### 4.4.2 decode()
```java
// 将二、八、十六进制字符串解析为 Integer
public static Integer decode(String nm) throws NumberFormatException {  
    int radix = 10;  
    int index = 0;  
    boolean negative = false;  
    Integer result;  
  
    if (nm.length() == 0)  
        throw new NumberFormatException("Zero length string");  
    char firstChar = nm.charAt(0);  
    // Handle sign, if present  
    if (firstChar == '-') {  
        negative = true;  
        index++;  
    } else if (firstChar == '+')  
        index++;  
  
    // Handle radix specifier, if present  
    // 十六进制
    if (nm.startsWith("0x", index) || nm.startsWith("0X", index)) {  
        index += 2;  
        radix = 16;  
    }  
    else if (nm.startsWith("#", index)) {  
        index ++;  
        radix = 16;  
    } 
    // 以 0 开头且不是一个单独的 0, 说明是八进制
    else if (nm.startsWith("0", index) && nm.length() > 1 + index) {  
        index ++;  
        radix = 8;  
    }  
  
    if (nm.startsWith("-", index) || nm.startsWith("+", index))  
        throw new NumberFormatException("Sign character in wrong position");  
  
    try {  
        result = Integer.valueOf(nm.substring(index), radix);  
        result = negative ? Integer.valueOf(-result.intValue()) : result;  
    } catch (NumberFormatException e) {  
        // If number is Integer.MIN_VALUE, we'll end up here. The next line  
        // handles this case, and causes any genuine format error to be        // rethrown.        
        String constant = negative ? ("-" + nm.substring(index))  
                                   : nm.substring(index);  
        result = Integer.valueOf(constant, radix);  
    }  
    return result;  
}
```
### 4.4.3 getInteger()
```java
// 获取系统属性对应的 Integer 的值
public static Integer getInteger(String nm) {  
    return getInteger(nm, null);  
}

public static Integer getInteger(String nm, int val) {  
    Integer result = getInteger(nm, null);  
    return (result == null) ? Integer.valueOf(val) : result;  
}

public static Integer getInteger(String nm, Integer val) {  
    String v = null;  
    try {  
        v = System.getProperty(nm);  
    } catch (IllegalArgumentException | NullPointerException e) {  
    }  
    if (v != null) {  
        try {  
            return Integer.decode(v);  
        } catch (NumberFormatException e) {  
        }  
    }  
    return val;  
}
```