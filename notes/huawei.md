### 1 去除重复随机数
- 生成 N 个 1 到 5000 之间的随机整数，重复的只保留一个，从小到大排序
``` 
input:
第一行是随机整数的个数 N
接下来的 N 行是生成的随机数

output:
输出多行处理结果

example:
input:
3
2
2
1

output:
1
2
```
#### solution
```java
public static void main(String[] args) {  
  
    Scanner scanner = new Scanner(System.in);  
  
    while (scanner.hasNext()) {  
  
        // 输入数字个数  
        int num = scanner.nextInt();  
  
        ArrayList<Integer> list = new ArrayList<>();  
  
        for (int i = 0; i < num; i++) {  
  
            int in = scanner.nextInt();  
  
            if (!list.contains(in)) list.add(in);  
        }  
  
        Collections.sort(list);  
        for (Integer elem : list) {  
            System.out.println(elem);  
        }  
  
    }  
    scanner.close();  
}
```
### 2 字符串分隔
- 输入一个字符串，按长度 8 拆分每个输入字符并输出
- 长度不足的用 0 补足
```
example:
input:
abc

output:
abc00000
```
#### solution
```java
public static void splitStr(StringBuilder str) {  
    int lenDiv = str.length() / 8;  
  
    if (lenDiv == 0) {  
        str.append("0".repeat(8 - str.length()));  
        System.out.println(str);  
    } else {  
        System.out.println(str.substring(0, 8));  
        splitStr(new StringBuilder(str.substring(8)));  
    }  
}
```
### 3 十六进制转十进制
#### solution
```java
public static void main(String[] args) {  
  
    Scanner scanner = new Scanner(System.in);  
  
    String hex = scanner.next();  
  
    hex = hex.toUpperCase();  
  
    if (hex.startsWith("0X")) {  
        hex = hex.substring(2);  
    }  
  
    int digit = 0;  
    int res = 0;  
    for (int i = hex.length() - 1; i >= 0; i--, digit++) {  
        res += (int) ((hex.charAt(i) - 55) * Math.pow(16.0, digit));  
    }  
  
    System.out.println(res + "");  
}
```
### 4 质数因子
- 输入一个正整数，按照从小到大的顺序输出它的所有质因子(需要重复列举, 空格分割)
```
example:
input: 180
output: 2 2 3 3 5
```
#### solution
```java

```