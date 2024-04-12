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
int n;
int m;
int a;
int out = Math.ceil(n / a) * Math.ceil(m / a)
```