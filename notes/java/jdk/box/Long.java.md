- 类定义
```java
public final class Long extends Number implements Comparable<Long> {...}
```
# 1 fields
```java
@Native public static final long MIN_VALUE = 0x8000000000000000L; // -2^63
@Native public static final long MAX_VALUE = 0x7fffffffffffffffL;

public static final Class<Long> TYPE = (Class<Long>) Class.getPrimitiveClass("long");

@Native public static final int SIZE = 64;
public static final int BYTES = SIZE / Byte.SIZE;

private final long value;

@Native private static final long serialVersionUID = 4290774380558885855L;
```
# 2 constructor
```java
public Long(long value) {  
    this.value = value;  
}

public Long(String s) throws NumberFormatException {  
    this.value = parseLong(s, 10);  
}
```
# 3 inner class
```java
private static class LongCache {  
    private LongCache(){}  
  
    static final Long cache[] = new Long[-(-128) + 127 + 1];  
  
    static {  
        for(int i = 0; i < cache.length; i++)  
            cache[i] = new Long(i - 128);  
    }  
}
```
# 4 methods
## 4.1 toString 类
### 4.1.1 getChars()
```java
static void getChars(long i, int index, char[] buf) {  
    long q;  
    int r;  
    int charPos = index;  
    char sign = 0;  
  
    if (i < 0) {  
        sign = '-';  
        i = -i;  
    }  
  
    // 首先循环取两位直到 int 的范围内 
    while (i > Integer.MAX_VALUE) {  
        q = i / 100;  
        // really: r = i - (q * 100);  
        r = (int)(i - ((q << 6) + (q << 5) + (q << 2)));  
        i = q;  
        buf[--charPos] = Integer.DigitOnes[r];  
        buf[--charPos] = Integer.DigitTens[r];  
    }  
  
    // 按 int 的 getChars 方法处理
    int q2;  
    int i2 = (int)i;  
    while (i2 >= 65536) {  
        q2 = i2 / 100;  
        // really: r = i2 - (q * 100);  
        r = i2 - ((q2 << 6) + (q2 << 5) + (q2 << 2));  
        i2 = q2;  
        buf[--charPos] = Integer.DigitOnes[r];  
        buf[--charPos] = Integer.DigitTens[r];  
    }  
  
    // Fall thru to fast mode for smaller numbers  
    // assert(i2 <= 65536, i2);    for (;;) {  
        q2 = (i2 * 52429) >>> (16+3);  
        r = i2 - ((q2 << 3) + (q2 << 1));  // r = i2-(q2*10) ...  
        buf[--charPos] = Integer.digits[r];  
        i2 = q2;  
        if (i2 == 0) break;  
    }  
    if (sign != 0) {  
        buf[--charPos] = sign;  
    }  
}
```
### 4.1.2 stringSize()
```java
static int stringSize(long x) {  
    long p = 10;  
    for (int i = 1; i < 19; i++) {  
        if (x < p)  
            return i;
		// p 循环乘 10 直到找到最相近的
        p = 10 * p;  
    }  
    // 否则返回 long 的最大位数 19
    return 19;  
}
```
### 4.1.3 toString()
```java
public String toString() {  
    return toString(value);  
}

public static String toString(long i) {  
    if (i == Long.MIN_VALUE)  
        return "-9223372036854775808";  
    int size = (i < 0) ? stringSize(-i) + 1 : stringSize(i);  
    char[] buf = new char[size];  
    getChars(i, size, buf);  
    return new String(buf, true);  
}

// 指定进制数
public static String toString(long i, int radix) {  
    if (radix < Character.MIN_RADIX || radix > Character.MAX_RADIX)  
        radix = 10;  
    if (radix == 10)  
        return toString(i);  
    char[] buf = new char[65];  
    int charPos = 64;  
    boolean negative = (i < 0);  
  
    if (!negative) {  
        i = -i;  
    }  
  
    while (i <= -radix) {  
        buf[charPos--] = Integer.digits[(int)(-(i % radix))];  
        i = i / radix;  
    }  
    buf[charPos] = Integer.digits[(int)(-i)];  
  
    if (negative) {  
        buf[--charPos] = '-';  
    }  
  
    return new String(buf, charPos, (65 - charPos));  
}
```
### 4.1.4 numberOfLeading(Trailing)Zeros()
```java
public static int numberOfLeadingZeros(long i) {  
    // HD, Figure 5-6  
     if (i == 0)  
        return 64;  
    int n = 1;  
    int x = (int)(i >>> 32);  
    if (x == 0) { n += 32; x = (int)i; }  
    if (x >>> 16 == 0) { n += 16; x <<= 16; }  
    if (x >>> 24 == 0) { n +=  8; x <<=  8; }  
    if (x >>> 28 == 0) { n +=  4; x <<=  4; }  
    if (x >>> 30 == 0) { n +=  2; x <<=  2; }  
    n -= x >>> 31;  
    return n;  
}

public static int numberOfTrailingZeros(long i) {  
    // HD, Figure 5-14  
    int x, y;  
    if (i == 0) return 64;  
    int n = 63;  
    y = (int)i; if (y != 0) { n = n -32; x = y; } else x = (int)(i>>>32);  
    y = x <<16; if (y != 0) { n = n -16; x = y; }  
    y = x << 8; if (y != 0) { n = n - 8; x = y; }  
    y = x << 4; if (y != 0) { n = n - 4; x = y; }  
    y = x << 2; if (y != 0) { n = n - 2; x = y; }  
    return n - ((x << 1) >>> 31);  
}
```
### 4.1.5 formatUnsignedLong()
```java
static int formatUnsignedLong(long val, int shift, char[] buf, int offset, int len) {  
    int charPos = len;  
    int radix = 1 << shift;  
    int mask = radix - 1;  
    do {  
        buf[offset + --charPos] = Integer.digits[((int) val) & mask];  
        val >>>= shift;  
    } while (val != 0 && charPos > 0);  
  
    return charPos;  
}
```
### 4.1.6 toUnsignedString0()
```java
static String toUnsignedString0(long val, int shift) {  
    // assert shift > 0 && shift <=5 : "Illegal shift value";  
    int mag = Long.SIZE - Long.numberOfLeadingZeros(val);  
    int chars = Math.max(((mag + (shift - 1)) / shift), 1);  
    char[] buf = new char[chars];  
  
    formatUnsignedLong(val, shift, buf, 0, chars);  
    return new String(buf, true);  
}
```
### 4.1.7 toBinary(Octal, Hex)String
```java
public static String toBinaryString(long i) {  
    return toUnsignedString0(i, 1);  
}

public static String toOctalString(long i) {  
    return toUnsignedString0(i, 3);  
}

public static String toHexString(long i) {  
    return toUnsignedString0(i, 4);  
}
```
### 4.1.8 toUnsignedBigInteger() [[BigInteger.java]]
```java
private static BigInteger toUnsignedBigInteger(long i) {  
    if (i >= 0L)  
        return BigInteger.valueOf(i);  
    else {  
        int upper = (int) (i >>> 32);  
        int lower = (int) i;  
  
        // return (upper << 32) + lower  
        return (BigInteger.valueOf(Integer.toUnsignedLong(upper))).shiftLeft(32).  
            add(BigInteger.valueOf(Integer.toUnsignedLong(lower)));  
    }  
}
```
### 4.1.9 toUnsignedString()
```java
public static String toUnsignedString(long i) {  
    return toUnsignedString(i, 10);  
}

public static String toUnsignedString(long i, int radix) {  
    if (i >= 0)  
        return toString(i, radix);  
    // 小于 0 
    else {  
        switch (radix) {  
        case 2:  
            return toBinaryString(i);  
  
        case 4:  
            return toUnsignedString0(i, 2);  
  
        case 8:  
            return toOctalString(i);  
  
        case 10:  
	        long quot = (i >>> 1) / 5; // 除以 10
            long rem = i - quot * 10;  
            return toString(quot) + rem; // 字符串拼接余数
  
        case 16:  
            return toHexString(i);  
  
        case 32:  
            return toUnsignedString0(i, 5);  
  
        default:  
            return toUnsignedBigInteger(i).toString(radix);  
        }  
    }  
}
```
## 4.2 parse 类
### 4.2.1 compareUnsigned()
```java
public static int compareUnsigned(long x, long y) {  
    return compare(x + MIN_VALUE, y + MIN_VALUE);  
}
```