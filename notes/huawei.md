### HJ3 去除重复随机数
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
### HJ4 字符串分隔
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
### HJ5 十六进制转十进制
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
### HJ6 质数因子
- 输入一个正整数，按照从小到大的顺序输出它的所有质因子(需要重复列举, 空格分割)
```
example:
input: 180
output: 2 2 3 3 5
```
#### solution
```java
public static void main(String[] args) {  
  
    ArrayList<String> resList = new ArrayList<>();  
  
    Scanner scanner = new Scanner(System.in);  
  
    long num = scanner.nextLong();  
  
    if (num > 2) {  
  
        while (num % 2 == 0) {  
            num /= 2;  
            resList.add(String.valueOf(2)); // 添加 2        }  
  
        for (int i = 3; i <= num; i++) {  
            if (num % i == 0) {  
                num /= i;  
                resList.add(String.valueOf(i));  
                i--; // 判断 i 的平方数  
            }  
        }  
  
        String res = "";  
  
        for(String str: resList) {  
            res += str + " ";  
        }  
        System.out.println(res);  
    } else {  
        System.out.println(num);  
    }  
}
```
### HJ13 句子逆序
- 调整单词的顺序而非字母
- 句子不确定单词个数
```
example:
input: I am a boy
output: boy a am I
```
#### solution
```java
public static void main(String[] args) {  
  
    Scanner scanner = new Scanner(System.in);  
  
    String sentence = scanner.nextLine();  
  
    LinkedList<String> words = new LinkedList<>();  
  
    for (String word : sentence.split(" ")) {  
        words.addFirst(word);  
    }  
  
    System.out.println(String.join(" ", words));  
  
}
```
### HJ15 十进制转二进制
- 输出 1 的个数
#### solution
```java
public static void main(String[] args) {  
  
	Scanner scanner = new Scanner(System.in);  
  
	int num = scanner.nextInt();  
  
    int cnt = num & 1;  
  
	// String bs = Integer.toBinaryString(num);   
	// System.out.println(bs);  
	// 8 位二进制, 2 次幂不停右移，最后有一个 1 
	for (int i = 0; i < 7; i++) {  
		cnt += (num >>>= 1) & 1;  
	}  
	System.out.println(cnt);  
    }
```
### HJ16 购物单
- 附件必须随主件购买，每件物品只能购买一次
- 每个主件可以有 $[0, 2]$ 个附件
- 花费不超过 N 元达到最大满意度

| 主件 | 附件           |
| ---- | -------------- |
| 电脑 | 打印机，扫描仪 |
| 书柜 | 图书           |
| 书桌 | 台灯，文具     |
| 工作椅     | 无               |
```
input:
第 1 行： N(总钱数) m(可购买物品个数)
第 2 至 m+1 行：第 j 行给出编号为 j-1 的物品的基本数据:
	v: value
	p: 重要程度(1 ~ 5)
	q: 0 -> 主件 / x(x>0) -> 附件所属主件编号
	
example:
input:
1000 5
800 2 0
400 5 1
300 5 1
400 3 0
500 2 0

output: 2200
```