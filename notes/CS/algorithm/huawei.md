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
- 一些例子：

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
50 5
20 3 5
20 3 5
10 3 0
10 2 0
10 1 0

output: 130
```
#### solution
```java
/*
 * example 有三件主件：
 *     1 M1 10 3
 *     2 M2 10 2
 *     3 M3 10 1
 * 主件 M3 有两个附件:
 *     S1 20 3
 *     S2 20 3
 * 
 *      0     1      2      3       4       5
 *  0   0     M1     M1     M1      M1      M1 
 *  1   0     M1    M1M2   M1M2    M1M2    M1M2
 *  2   0     M1    M1M2  M3S1S2  M3S1S2  M3S1S2 
 *  
 *  每个主件对应四种情况: M, MS1, MS2, MS1S2
 *  dp[i][j] = max(dp[i-1][j], dp[i-1][j-w[j]] + v[j])  
 */

public static void main(String[] args) {  
  
    Scanner sc = new Scanner(System.in);  
    String s = sc.nextLine();  
  
    String[] ins = s.split(" ");  
    int total = Integer.parseInt(ins[0]) / 10;  
    int goodsCnt = Integer.parseInt(ins[1]);  
  
    ArrayList<Item> items = new ArrayList<>();  
    items.add(new Item(0, 0, 0));  
  
    for (int i = 0; i < goodsCnt; i++) {  
        String str = sc.nextLine();  
        String[] good = str.split(" ");  
        Item item = new Item(Integer.parseInt(good[0]) / 10,  
                Integer.parseInt(good[1]),  
                Integer.parseInt(good[2]));  
        item.satisfied = item.v * item.p;  
        items.add(item);  
    }  
  
    // 将附件添加到 sub 列表中  
    for (int i = 1; i < items.size(); i++) {  
        Item curr = items.get(i);  
        if (curr.q != 0) {  
            items.get(curr.q).sub.add(curr);  
        }  
    }  
  
    int[][] dp = new int[items.size()][total + 1];  
  
    for (int i = 1; i < items.size(); i++) {  
        for (int j = 1; j < total + 1; j++) {  
            Item curr = items.get(i);  
            if (curr.q == 0) {  
                dp[i][j] = dp[i - 1][j];  
                if (j - curr.v >= 0) { // 只买主件  
                    dp[i][j] = Integer.max(dp[i][j], dp[i - 1][j - curr.v] + curr.satisfied);  
                }  
  
                if (curr.sub.size() >= 1) { // 至少有一个附件  
                    Item sub1 = curr.sub.get(0);  
                    if (j - curr.v - sub1.v >= 0) {  
                        dp[i][j] = Integer.max(dp[i][j],  
                                dp[i - 1][j - curr.v - sub1.v] + curr.satisfied + sub1.satisfied);  
                    }  
                }  
  
                if (curr.sub.size() > 1) { // 添加第二个附件  
                    Item sub2 = curr.sub.get(0);  
                    if (j - curr.v - sub2.v >= 0) {  
                        // 此处的dp[i][j]是上一步买一个附件的满意度  
                        dp[i][j] = Integer.max(dp[i][j],  
                                dp[i - 1][j - curr.v - sub2.v] + curr.satisfied + sub2.satisfied);  
                    }  
                }  
  
                if (curr.sub.size() > 1) { // 同时购买  
                    Item sub1 = curr.sub.get(0);  
                    Item sub2 = curr.sub.get(1);  
                    if (j - curr.v - sub1.v - sub2.v >= 0) {  
                        dp[i][j] = Integer.max(dp[i][j],  
                                dp[i - 1][j - curr.v - sub1.v - sub2.v] + curr.satisfied + sub1.satisfied + sub2.satisfied);  
  
                    }  
                }  
            } else {  
                dp[i][j] = dp[i - 1][j];  
            }  
        }  
    }  
    System.out.println(dp[items.size() - 1][total] * 10);  
}
```
### HJ17 坐标移动
- 从(0, 0)开始，从字符串读取坐标进行移动，WASD 表示方向
- 合法坐标为 W(ASD)  + 2 位以内数字
- 坐标之间以 ';' 分隔，" "不影响
- 输出以 ',' 分隔的坐标
```
example:

input: A10;S20;W10;D30;X;A1A;B10A11;;A10;

output:10,-10
```
#### solution
```java
public static void main(String[] args) {  
  
    Scanner scanner = new Scanner(System.in);  
    String str = scanner.nextLine();  
    String[] move = str.split(";");  
  
    HashMap<String, Integer> points = new HashMap<>();  
    for (String s : move) {  
        if (s.length() <= 3 && (  
                s.startsWith("W") ||  
                        s.startsWith("A") ||  
                        s.startsWith("S") ||  
                        s.startsWith("D"))) {  
            boolean legal = true;  
            for (int i = 1; i < s.length(); i++) {  
                if (s.charAt(i) < '0' || s.charAt(i) > '9') {  
                    legal = false;  
                    break;  
                }  
            }  
            if (legal) {  
                int distance = Integer.parseInt(s.substring(1));  
                String direction = String.valueOf(s.charAt(0));  
                Integer val = points.get(direction);  
                if (val == null) {  
                    points.put(direction, distance);  
                } else {  
                    val += distance;  
                    points.put(direction, val);  
                }  
            }  
        }  
    }  
  
    int x = 0;  
    int y = 0;  
    for (Map.Entry<String, Integer> entry : points.entrySet()) {  
        switch (entry.getKey()) {  
            case "W":  
                y += entry.getValue();  
                break;  
            case "A":  
                x -= entry.getValue();  
                break;  
            case "S":  
                y -= entry.getValue();  
                break;  
            case "D":  
                x += entry.getValue();  
        }  
    }  
    System.out.println(x + "," + y);  
}
```
### HJ18 识别有效 IP 地址和掩码
- IP 地址分为 5 类
	- A：1.0.0.0 -> 126.255.255.255
	- B: 128.0.0.0 -> 191.255.255.255
	- C: 192.0.0.0 -> 223.255.255.255
	- D: 224.0.0.0 -> 239.255.255.255
	- E: 240.0.0.0 -> 255.255.255.255
- 私网 IP 范围是：
	- 10.0.0.0 -> 10.255.255.255
	- 172.16.0.0 -> 172.31.255.255
	- 192.168.0.0 -> 192.168.255.255
- 二进制子网掩码前面是连续的 1 后面是连续的 0，全 0 或全 1 均为非法子网掩码
- 输入多行字符串，每行一个 IP 地址和掩码，通过~隔开
- 输出统计 ABCDE，错误 IP 地址或错误掩码，私有 IP 个数，空格分隔
- 192.\*.\*.\*不属于不合法，忽略
```
example:
input:
10.70.44.68~255.254.255.0
1.0.0.1~255.0.0.0
192.168.0.2~255.255.255.0
19..0.~255.255.255.0

output: 1 0 1 0 0 2 1


```