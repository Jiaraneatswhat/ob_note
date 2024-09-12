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
### 4.1.4 numberOfLeadingZeros()
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
static int formatUnsignedInt(int val, int shift, char[] buf, int offset, int len) {  
    int charPos = len;  
    int radix = 1 << shift;  
    int mask = radix - 1;  
    do {  
        buf[offset + --charPos] = Integer.digits[val & mask];  
        val >>>= shift;  
    } while (val != 0 && charPos > 0);  
  
    return charPos;  
}
```