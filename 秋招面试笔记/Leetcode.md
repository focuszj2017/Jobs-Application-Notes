# Leetcode总结

## 经验

1. 对数组头尾增加两个哨兵，应该这么写：

   ```java
           //头尾增加两个哨兵
           int[] tmp = new int[height.length + 2];
           System.arraycopy(height, 0, tmp, 1, height.length);
   ```

2. 背包问题中，完全背包问题的内外循环可以互换（不考虑顺序的情况下）

   > **组合问题公式**	
   > dp[i] += dp[i-num]
   >
   > **True、False问题公式**	
   > dp[i] = dp[i] or dp[i-num]
   >
   > **最大最小问题公式**	
   > dp[i] = min(dp[i], dp[i-num]+1)或者dp[i] = max(dp[i], dp[i-num]+1)
   >
   > **背包问题技巧**：
   > 1.如果是0-1背包，即数组中的元素不可重复使用，nums放在外循环，target在内循环，且内循环倒序；
   >
   >
   > for num in nums:
   >     for i in range(target, nums-1, -1):
   > 2.如果是完全背包，即数组中的元素可重复使用，nums放在外循环，target在内循环。且内循环正序。
   >
   >
   > for num in nums:
   >     for i in range(nums, target+1):
   > 3.如果组合问题需考虑元素之间的顺序，需将target放在外循环，将nums放在内循环。
   >
   > for i in range(1, target+1):
   >     for num in nums:

   