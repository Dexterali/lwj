# 第 k 小整数

## 题目描述

现有 $n$ 个正整数，要求出这 $n$ 个正整数中的第 $k$ 个最小整数。

## 输入格式

第一行为 $n$ 和 $k$; 第二行开始为 $n$ 个正整数的值，整数间用空格隔开。

## 输出格式

第$k$个最小整数的值；若无解，则输出 `NO RESULT`。

## 样例 #1

### 样例输入 #1

```
10 3
1 3 3 7 2 5 1 2 4 6
```

### 样例输出 #1

```
3
```

## 提示

$n \leq 10000$，$k \leq 1000$，正整数均小于 $30000$。

```c++
#include <iostream>

using namespace std;

const int N = 1e5 + 10;

int n, k, q[N];


int quick_sort(int l, int r, int k) {
    if (l == r) return q[l];
    
    int i = l - 1, j = r + 1, x = q[l];
    while(i < j) {
        do ++i; while (q[i] < x);
        do --j; while (q[j] > x);
        if(i < j) swap(q[i], q[j]);
    }
    
    int sl = j - l + 1;
    if(k <= sl) quick_sort(l, j, k);
    else quick_sort(j + 1, r, k - sl);
}

int main() {
    
    cin >> n >> k;
    
    for (int i = 0; i < n; i++) cin >> q[i];
    
    cout << quick_sort(0, n - 1, k) << endl;
    
    return 0;
}
```