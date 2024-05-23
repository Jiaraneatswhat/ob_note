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

	... 

}
```