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
### 2 
