# 算法模版

## 回文串

[最长回文子串 - LPS](http://notebook.desgard.com:8888/notebooks/gua_interview_note/ch01_data_structure_and_algorithm_1.ipynb#1%EF%BC%89%E6%9C%80%E9%95%BF%E5%9B%9E%E6%96%87%E5%AD%90%E4%B8%B2---LPS)

```python
# LPS DP O(n^2)
def lps_dp(s):
    lens = len(s)
    if lens <= 1:
        return s
    dp = [[False for i in range(lens)] for j in range(lens)]
    for i in range(lens):
        dp[i][i] = True
        dp[i][i - 1] = True
    l, r = 0, 0
    for k in range(2, lens + 1):
        for i in range(0, lens - k + 1):
            if s[i] == s[i + k - 1] and dp[i + 1][i + k - 2]:
                dp[i][i + k - 1] = True
                if r - l < k:
                    l = i
                    r = i + k - 1
    return (l, r, s[l:r + 1])
```

## [Two Sum](https://leetcode.com/problems/two-sum/description/)

这里使用 hash table 方法来处理，保证 `O(n)` 复杂度完成。

```python
class Solution:
    def twoSum(self, nums, target):
        ht = {}
        for index, num in enumerate(nums):
            aim = target - num
            if aim in ht:
                return [min(index, ht[aim]), max(index, ht[aim])]
            else:
                ht[num] = index
        return 
```

## [3 Sum](https://leetcode.com/problems/3sum/description/)


```python
class Solution:
    def threeSum(self, num):
        num.sort()
        res = []
        for i in range(len(num) - 2):
            if i == 0 or num[i] > num[i - 1]:
                l, r = i + 1, len(num) - 1
                while l < r:
                    if num[l] + num[r] == -num[i]:
                        res.append([num[i], num[l], num[r]])
                        l += 1; r -= 1;
                        while l < r and num[l] == num[l - 1]: 
                            l += 1
                        while l < r and num[r] == num[r + 1]: 
                            r -= 1
                    elif num[l] + num[r] < -num[i]:
                        while l < r:
                            l += 1 
                            if num[l] > num[l - 1]: break
                    else:
                        while l < r:
                            r -= 1            
                            if num[r] < num[r + 1]: break    
        return res
```

## 快速幂取模

```c
double quickPow(int base, int n, int mod) {
    double muti = base;
    int res = 1;
    while (n) {
        if (n & 1) {
            res *= muti % mod;
        }
        muti *= muti;
        n >>= 1;
    }
    return res;
}
```

## 快排

```c
vector<int> a;

void quick_sort(int l, int r) {
    if (l >= r) return;
    int i = l, j = r, t = a[l];
    while (i < j) {
        while (i < j && a[j] >= t) j --;
        if (i < j) a[i ++] = a[j];
        while (i < j && a[i] <  t) i ++;
        if (i < j) a[j --] = a[i];
    }
    a[i] = t;
    quick_sort(l, i - 1);
    quick_sort(i + 1, r);
}
```

