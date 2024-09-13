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

// 如果两个
private static final int KARATSUBA_THRESHOLD = 80;
```