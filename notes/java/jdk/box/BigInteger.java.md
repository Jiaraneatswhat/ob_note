- 类定义
```java
public class BigInteger extends Number implements Comparable<BigInteger> {...}
```
# 1 fields
```java
final int signum;
// 以 32 位为一组存放大数, mag[0], mag[1] ,...
final int[] mag;

final static long LONG_MASK = 0xffffffffL;

private static final int MAX_MAG_LENGTH = Integer.MAX_VALUE / Integer.SIZE + 1; // (1 << 26)

// 超出这个长度的大数在 searchLen 计算时会溢出
private static final int PRIME_SEARCH_BIT_LENGTH_LIMIT = 500000000;

// 如果参与计算的两个 mag 数组的长度都超过此阈值，则使用 karatsuba 乘法
private static final int KARATSUBA_THRESHOLD = 80;

// 两个 mag 数组长度都远大于 karatsuba 阈值，且至少有一个远大于 toom_cook阈值，则使用 toom_cook算法
private static final int TOOM_COOK_THRESHOLD = 240;

// 使用两种开方算法的阈值
private static final int KARATSUBA_SQUARE_THRESHOLD = 128;
private static final int TOOM_COOK_SQUARE_THRESHOLD = 216;

// 使用 Burnikel-Ziegler 除法的除数位数阈值
static final int BURNIKEL_ZIEGLER_THRESHOLD = 80;
// Burnikel-Ziegler 的 offset
// 除数位数超过了 Burnikel-Ziegler 阈值，被除数位数超过了除数位数 + offset，使用 Burnikel-Ziegler 除法
static final int BURNIKEL_ZIEGLER_OFFSET = 40;
```