### 1A. Theatre Square

Theatre Square in the capital city of Berland has a rectangular shape with the size $n \times m$ meters. On the occasion of the city's anniversary, a decision was taken to pave the Square with square granite flagstones. Each flagstone is of the size $a \times a$.

What is the least number of flagstones needed to pave the Square? It's allowed to cover the surface larger than the Theatre Square, but the Square has to be covered. It's not allowed to break the flagstones. The sides of flagstones should be parallel to the sides of the Square.

`input: ` $n, m, a,(1 \leq n, m, a \leq 10^9)$
`output: ` Write the needed number of flagstones
```
example:
input:
6 6 4
output:
4
```

`solution:`
```java
	Scanner scanner = new Scanner(System.in);  
	System.out.println("input n: ");  
	int n = scanner.nextInt();  
	System.out.println("input m: ");  
	int m = scanner.nextInt();  
	System.out.println("input a: ");  
	int a = scanner.nextInt();  
	int out = (int) (Math.ceil(n / a) * Math.ceil(m / a));  
	System.out.println(out == 0 ? 1 : out);
```

### 1B. Spreadsheet

In the popular spreadsheets systems (for example, in Excel) the following numeration of columns is used. The first column has number A, the second — number B, etc. till column 26 that is marked by Z. Then there are two-letter numbers: column 27 has number AA, 28 — AB, column 52 is marked by AZ. After ZZ there follow three-letter numbers, etc.

The rows are marked by integer numbers starting with 1. The cell name is the concatenation of the column and the row numbers. For example, BC23 is the name for the cell that is in column 55, row 23.

Sometimes another numeration system is used: RXCY, where X and Y are integer numbers, showing the column and the row numbers respectfully. For instance, R23C55 is the cell from the previous example.

Your task is to write a program that reads the given sequence of cell coordinates and produce each item written according to the rules of another numeration system.

`input:` 
The first line of the input contains integer number $n\:(1 \leq n \leq 10^5)$, the number of coordinates in the test. Then there follow $n$ lines, each of them contains coordinates. All the coordinates are correct, there are no cells with the column and/or the row numbers larger than $10^6$ .
`output:`
Write $n$ lines, each line should contain a cell coordinates in the other numeration system.
```
example:
input:
2
R23C55
BC23

output:
BC23
R23C55
```

`solution:`
```java
    Scanner s = new Scanner(System.in);  
    int n = s.nextInt();  
    String a;  
    // 字母表  
    char[] word = new char[27];  
    word[1] = 'A';  
    for (int i = 2; i <= 26; i++)  
        word[i] = (char) (word[i - 1] + 1);  
    for (int i = 0; i < n; i++) {  
        a = s.next();  
        int l = a.length();  
        int flag = 0;  
        for (int j = 0; j < l - 1; j++)  
        // RXCY类型 数字 + 字母  
            // 符合条件将 flag 置为 1            
            if (a.charAt(j) >= '0' && a.charAt(j) <= '9'  
                    && a.charAt(j + 1) >= 'A' && a.charAt(j + 1) <= 'Z') {  
                flag = 1;  
            }  
        if (flag == 1) {  
            int n1 = 0, n2 = 0;  
            int k = 1;  
            // RXCY 中的 X 转换为十进制  
            while (a.charAt(k) >= '0' && a.charAt(k) <= '9') {  
                k++;  
            }  
            n1 = Integer.parseInt(a.substring(1, k));  
            // Y 转为十进制  
            n2 = Integer.parseInt(a.substring(k + 1));  
            int x[] = new int[10000];  
            int t = 0;  
            while (n2 > 0) {  
                if (n2 % 26 == 0) {  
                    x[t++] = 26;  
                    n2 = n2 / 26 - 1; // 26 / 26 - 1  
                } else {  
                    x[t++] = n2 % 26;  
                    n2 /= 26;  
                }  
            }  
            for (int j = t - 1; j >= 0; j--)  
                System.out.print(word[x[j]]);  
            System.out.println(n1);  
        }  
        /** 情况2：BC23类型 */  
        else {  
            int n1 = 0, n2 = 0;  
            int k = 0;  
            /*记录字母出现个数*/  
            while (a.charAt(k) >= 'A' && a.charAt(k) <= 'Z') {  
                n1 = n1 * 26 + a.charAt(k) - 'A' + 1; // 'A' - 'A' + 1 = 1  
                k++;  
            }  
            /*记录数字出现个数*/  
            for (; k < l; k++)  
                n2 = n2 * 10 + a.charAt(k) - '0';  
            System.out.println("R" + n2 + "C" + n1);  
  
        }  
    }  
```
### 1C. Ancient Berland Circus

Nowadays all circuses in Berland have a round arena with diameter 13 meters, but in the past things were different.

In Ancient Berland arenas in circuses were shaped as a regular (equiangular) polygon(多边形), the size and the number of angles could vary from one circus to another. In each corner of the arena there was a special pillar, and the rope strung between the pillars marked the arena edges.

Recently the scientists from Berland have discovered the remains of the ancient circus arena. They found only three pillars, the others were destroyed by the time.

You are given the coordinates of these three pillars. Find out what is the smallest area that the arena could have.

`input:`
The input file consists of three lines, each of them contains a pair of numbers –– coordinates of the pillar. Any coordinate doesn't exceed 1000 by absolute value, and is given with at most six digits after decimal point
`output:`
Output the smallest possible area of the ancient arena. This number should be accurate to at least 6 digits after the decimal point. It's guaranteed that the number of angles in the optimal polygon is not larger than 100.
```
example:
input:
0.000000 0.000000  
1.000000 1.000000  
0.000000 1.000000

output:
1.00000000
```
`solution:`
海伦公式：
	三角形的三条边 $a,b,c$，其面积为 $\sqrt{p(p-a)(p-b)(p-c)},p=$
