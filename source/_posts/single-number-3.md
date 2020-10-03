---
title: single number 3
date: 2020-10-03 12:33:34
tags: leetcode, 
---

思路：

1. 先考虑是否遇到过相似的题目：
   1. 有：通过异或，删除出现过偶次数的number，保留只有奇次数的number 
2. 这道题的特殊点在于有2个奇次数的nubmer
3. 如何区分2个奇次数的number？首先必须找到它俩的特殊bit，用于区分


解题方法：

1. 创建第一个bitwise1, 0000x
   1. bitwise1 = bitwise1^x^y^a^a = bitwsie1^x^y
2. 创建第二个bitwise2,
   1. Bitwise2 = bitwise1^(-bitwise1+1)
   2. bitwise2中只有right most 1 bit of bitwise1
   3. bitwise2就含有x和y的特殊bit了
3. 创建第三个bitwise3, i : nums
   1. if bitwise3^i !=0, bitwise3 = bitwise3^i
   2. int the end, bitwise3= x (or y);
