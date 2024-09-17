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

private static final int SMALL_PRIME_THRESHOLD = 95;  
  
private static final int DEFAULT_PRIME_CERTAINTY = 100;

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

// 使用 Schoenhage recursive base conversion 的阈值
private static final int SCHOENHAGE_BASE_CONVERSION_THRESHOLD = 20;

private static final int MULTIPLY_SQUARE_THRESHOLD = 20;

// Montgomery 乘法的阈值
private static final int MONTGOMERY_INTRINSIC_THRESHOLD = 512;

static int[] bnExpModThreshTable = {7, 25, 81, 241, 673, 1793, Integer.MAX_VALUE};

private static long bitsPerDigit[] = { 0, 0,  
    1024, 1624, 2048, 2378, 2648, 2875, 3072, 3247, 3402, 3543, 3672,  
    3790, 3899, 4001, 4096, 4186, 4271, 4350, 4426, 4498, 4567, 4633,  
    4696, 4756, 4814, 4870, 4923, 4975, 5025, 5074, 5120, 5166, 5210,  
    5253, 5295};

private static int digitsPerLong[] = {0, 0, 62, 39, 31, 27, 24, 22, 20, 19, 
	18, 18, 17, 17, 16, 16, 15, 15, 15, 14, 14, 14, 14, 13, 13, 13, 13, 13, 
	13, 12, 12, 12, 12, 12, 12, 12, 12};

private static BigInteger longRadix[] = {null, null,  
    valueOf(0x4000000000000000L), valueOf(0x383d9170b85ff80bL),  
    valueOf(0x4000000000000000L), valueOf(0x6765c793fa10079dL),  
    valueOf(0x41c21cb8e1000000L), valueOf(0x3642798750226111L),  
    valueOf(0x1000000000000000L), valueOf(0x12bf307ae81ffd59L),  
    valueOf( 0xde0b6b3a7640000L), valueOf(0x4d28cb56c33fa539L),  
    valueOf(0x1eca170c00000000L), valueOf(0x780c7372621bd74dL),  
    valueOf(0x1e39a5057d810000L), valueOf(0x5b27ac993df97701L),  
    valueOf(0x1000000000000000L), valueOf(0x27b95e997e21d9f1L),  
    valueOf(0x5da0e1e53c5c8000L), valueOf( 0xb16a458ef403f19L),  
    valueOf(0x16bcc41e90000000L), valueOf(0x2d04b7fdd9c0ef49L),  
    valueOf(0x5658597bcaa24000L), valueOf( 0x6feb266931a75b7L),  
    valueOf( 0xc29e98000000000L), valueOf(0x14adf4b7320334b9L),  
    valueOf(0x226ed36478bfa000L), valueOf(0x383d9170b85ff80bL),  
    valueOf(0x5a3c23e39c000000L), valueOf( 0x4e900abb53e6b71L),  
    valueOf( 0x7600ec618141000L), valueOf( 0xaee5720ee830681L),  
    valueOf(0x1000000000000000L), valueOf(0x172588ad4f5f0981L),  
    valueOf(0x211e44f7d02c1000L), valueOf(0x2ee56725f06e5c71L),  
    valueOf(0x41c21cb8e1000000L)};

private static int digitsPerInt[] = {0, 0, 30, 19, 15, 13, 11,  
    11, 10, 9, 9, 8, 8, 8, 8, 7, 7, 7, 7, 7, 7, 7, 6, 6, 6, 6,  
    6, 6, 6, 6, 6, 6, 6, 6, 6, 6, 5};

private static int intRadix[] = {0, 0,  
    0x40000000, 0x4546b3db, 0x40000000, 0x48c27395, 0x159fd800,  
    0x75db9c97, 0x40000000, 0x17179149, 0x3b9aca00, 0xcc6db61,  
    0x19a10000, 0x309f1021, 0x57f6c100, 0xa2f1b6f,  0x10000000,  
    0x18754571, 0x247dbc80, 0x3547667b, 0x4c4b4000, 0x6b5a6e1d,  
    0x6c20a40,  0x8d2d931,  0xb640000,  0xe8d4a51,  0x1269ae40,  
    0x17179149, 0x1cb91000, 0x23744899, 0x2b73a840, 0x34e63b41,  
    0x40000000, 0x4cfa3cc1, 0x5c13d840, 0x6d91b519, 0x39aa400  
};

private final static int MAX_CONSTANT = 16;  
private static BigInteger posConst[] = new BigInteger[MAX_CONSTANT+1];  
private static BigInteger negConst[] = new BigInteger[MAX_CONSTANT+1];

private static volatile BigInteger[][] powerCache;
private static final double[] logCache;

private static final BigInteger SMALL_PRIME_PRODUCT  
	= valueOf(3L * 5 * 7 * 11 * 13 * 17 * 19 * 23 * 29 * 31 * 37 * 41);

// 常数
private static final double LOG_TWO = Math.log(2.0);
public static final BigInteger ZERO = new BigInteger(new int[0], 0);
public static final BigInteger ONE = valueOf(1);
private static final BigInteger TWO = valueOf(2);
private static final BigInteger NEGATIVE_ONE = valueOf(-1);
public static final BigInteger TEN = valueOf(10);

private static String zeros[] = new String[64];  
// zeros[i] = "000..." i 个 0
static {  
    zeros[63] = "000000000000000000000000000000000000000000000000000000000000000";  
    for (int i= 0; i < 63; i++)  
        zeros[i] = zeros[63].substring(0, i);  
}

private static final long serialVersionUID = -8287574255936472291L;

private static final ObjectStreamField[] serialPersistentFields = {  
    new ObjectStreamField("signum", Integer.TYPE),  
    new ObjectStreamField("magnitude", byte[].class),  
    new ObjectStreamField("bitCount", Integer.TYPE),  
    new ObjectStreamField("bitLength", Integer.TYPE),  
    new ObjectStreamField("firstNonzeroByteNum", Integer.TYPE),  
    new ObjectStreamField("lowestSetBit", Integer.TYPE)  
    };
```
# 2 constructor
## 2.1 前置方法
### 2.1.1 stripLeadingZeroBytes()
```java
private static int[] stripLeadingZeroBytes(byte a[]) {  
    int byteLength = a.length;  
    int keep;  
    // 找到第一个非 0 的索引 keep
    for (keep = 0; keep < byteLength && a[keep] == 0; keep++);  
    // 根据 keep 和字节数组长度计算出对应的 int 数组长度
    int intLength = ((byteLength - keep) + 3) >>> 2;  
    int[] result = new int[intLength];  
    int b = byteLength - 1;  
    for (int i = intLength - 1; i >= 0; i--) {  
        result[i] = a[b--] & 0xff;  
        int bytesRemaining = b - keep + 1;  
        int bytesToTransfer = Math.min(3, bytesRemaining);  
        for (int j = 8; j <= (bytesToTransfer << 3); j += 8)  
            result[i] |= ((a[b--] & 0xff) << j);  
    }  
    return result;  
}
```