- 类定义
```java
public final class Byte extends Number implements Comparable<Byte> {...}
```
# 1 fields
```java
public static final byte MIN_VALUE = -128;
public static final byte   MAX_VALUE = 127;

public static final Class<Byte> TYPE = (Class<Byte>) Class.getPrimitiveClass("byte");

public static final int SIZE = 8;
public static final int BYTES = SIZE / Byte.SIZE;

private final byte value;

private static final long serialVersionUID = -7183698231559129828L;
```
# 2 constructor
```java
public Byte(byte value) {  
    this.value = value;  
}

public Byte(String s) throws NumberFormatException {  
    this.value = parseByte(s, 10);  
}
```
# 3 inner class
```java
private static class ByteCache {  
    private ByteCache(){}  
  
    static final Byte cache[] = new Byte[-(-128) + 127 + 1];  
  
    static {  
        for(int i = 0; i < cache.length; i++)  
            cache[i] = new Byte((byte)(i - 128));  
    }  
}
```
# 4 methods
## 4.1 parseByte()
```java
public static byte parseByte(String s) throws NumberFormatException {  
    return parseByte(s, 10);  
}

public static byte parseByte(String s, int radix)  
    throws NumberFormatException {  
    int i = Integer.parseInt(s, radix);  
    if (i < MIN_VALUE || i > MAX_VALUE)  
        throw new NumberFormatException(  
            "Value out of range. Value:\"" + s + "\" Radix:" + radix);  
    return (byte)i;  
}
```
## 4.2 valueOf()
```java
public static Byte valueOf(String s) throws NumberFormatException {  
    return valueOf(s, 10);  
}

public static Byte valueOf(String s, int radix)  
    throws NumberFormatException {  
    return valueOf(parseByte(s, radix));  
}

// byte 肯定在 Cache 的范围内，直接从缓存获取
public static Byte valueOf(byte b) {  
    final int offset = 128;  
    return ByteCache.cache[(int)b + offset];  
}
```
## 4.3 decode()
```java
public static Byte decode(String nm) throws NumberFormatException {  
    int i = Integer.decode(nm);  
    if (i < MIN_VALUE || i > MAX_VALUE)  
        throw new NumberFormatException(  
                "Value " + i + " out of range from input " + nm);  
    return valueOf((byte)i);  
}
```
## 4.4 toUnsignedInt()
```java
public static int toUnsignedInt(byte x) {  
    return ((int) x) & 0xff;  
}
```
## 4.5 toUnsignedLong()
```java
public static long toUnsignedLong(byte x) {  
    return ((long) x) & 0xffL;  
}
```