---
title: 470用Rand7()实现Rand10()
categories: 
- 算法
- LeetCode
---

题目地址：https://leetcode-cn.com/problems/implement-rand10-using-rand7/

(rand(7)-1)*7 + rand(7) 可以等概率生成[1,49]的数

```java
class Solution extends SolBase {
    public int rand10() {
        int num = (rand7() - 1) * 7 + rand7();
        // 只要它还大于40，就不断生成
        while (num > 40) {
            num = (rand7() - 1) * 7 + rand7();
        }
        // 返回结果，+1是为了解决 40%10为0的情况
        return 1 + num % 10;
    }
}
```

