- 类定义
```java
public final class Short extends Number implements Comparable<Short> {...}
```
# 1 fields
```java
public static final short   MIN_VALUE = -32768;
public static final short   MAX_VALUE = 32767;

public static final Class<Short> TYPE = (Class<Short>) Class.getPrimitiveClass("short");

public static final int SIZE = 16;
public static final int BYTES = SIZE / Byte.SIZE;

private final short value;
private static final long serialVersionUID = 7515723908773894738L;
```
# 2 constructor
```java
public Short(short value) {  
    this.value = value;  
}

public Short(String s) throws NumberFormatException {  
    this.value = parseShort(s, 10);  
}
```
# 3 inner class
```java
private static class ShortCache {  
    private ShortCache(){}  
  
    static final Short cache[] = new Short[-(-128) + 127 + 1];  
  
    static {  
        for(int i = 0; i < cache.length; i++)  
            cache[i] = new Short((short)(i - 128));  
    }  
}
```
# 4 methods
## 4.1 parseShort()
```java
public static short parseShort(String s) throws NumberFormatException {  
    return parseShort(s, 10);  
}

// 调用 parseInt 后强转为 short
public static short parseShort(String s, int radix)  
    throws NumberFormatException {  
    int i = Integer.parseInt(s, radix);  
    if (i < MIN_VALUE || i > MAX_VALUE)  
        throw new NumberFormatException(  
            "Value out of range. Value:\"" + s + "\" Radix:" + radix);  
    return (short)i;  
}
```
## 4.2 valueOf()
```java
public static Short valueOf(String s) throws NumberFormatException {  
    return valueOf(s, 10);  
}

public static Short valueOf(String s, int radix)  
    throws NumberFormatException {  
    return valueOf(parseShort(s, radix));  
}

// [-128, 127]
public static Short valueOf(short s) {  
    final int offset = 128;  
    int sAsInt = s;  
    if (sAsInt >= -128 && sAsInt <= 127) { // must cache  
        return ShortCache.cache[sAsInt + offset];  
    }  
    return new Short(s);  
}
```
## 4.3 decode()
```java
public static Short decode(String nm) throws NumberFormatException {  
    int i = Integer.decode(nm);  
    if (i < MIN_VALUE || i > MAX_VALUE)  
        throw new NumberFormatException(  
                "Value " + i + " out of range from input " + nm);  
    return valueOf((short)i);  
}
```
## 4.4 toUnsignedInt()
```java
public static int toUnsignedInt(short x) {  
    return ((int) x) & 0xffff;  
}
```
## 4.5 toUnsignedLong()
```java
public static long toUnsignedLong(short x) {  
    return ((long) x) & 0xffffL;  
}
```
## 4.6 reverseBytes()
```java

```