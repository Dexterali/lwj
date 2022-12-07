# lowbit 运算

\\(lowbit(n)\\)定义为非负整数\\(n\\)在二进制表示下最低位的1和其后所有的0构成的数值。
> 例：\\(lowbit(10) = lowbit((1010)_2) = 2 = (10)_2\\)

## 推导公式

\begin{cases}
    lowbit(n) = n \\& (\sim n  + 1) \\\\
    \sim n = -1 - n \\\\
    lowbit(n) = n \\& (\sim n + 1) = n \\& -n
\end{cases}

## 实例：求整数二进制表示下所有是1的位

```cpp
int H[37];
for (int i = 0; i < 36; i++) {
    H[(1ll << i) % 37] = i;
}
while(n > 0) {
    cout << H[(n & -n) % 37] << " ";
    n -= n & -n;
}
cout << endl;
```
> 注解：
> 
> \\(H[2^k mod 37] = k\\)
> 
> \\(\forall k \in[0, 35], 2^k mod 37\\) 互不相等，且恰好取遍整数 \\(1\sim36\\) 
