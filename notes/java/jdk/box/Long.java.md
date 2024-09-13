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
        p = 10 * p;  
    }  
    return 19;  
}
```