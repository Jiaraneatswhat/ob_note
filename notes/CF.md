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