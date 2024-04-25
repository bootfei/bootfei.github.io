---
title: 'leetcode:如何规划刷题'
date: 2020-12-28 10:59:50
tags:
---



## Sequence or substring

| 题目                                               | 相似题目                                   | difficulty | space and time complexity                |
| -------------------------------------------------- | ------------------------------------------ | ---------- | ---------------------------------------- |
| ~~5. Longest Palindromic Substring<br />~~         |                                            | 2          | Palindromic pivot with expand            |
| 128.Longest Consecutive Sequence                   |                                            | 2.5        |                                          |
| 300. Longest Increasing Subsequence(刷了3次)       | 516(刷了3次)、673（刷了2次）               | 2          | dp, subproblem by sub-length<br />O(n^2) |
| ~~647. Palindromic Substrings~~                    |                                            | 2          | Palindromic pivot with expand            |
| 1143. Longest Common Subsequence(刷了2次)          |                                            | 3.5        | dp                                       |
| 1048. Longest String Chain                         |                                            | 3          | dp: S(n),  O(n^2)                        |
| 1898. Maximum Number of Removable Characters       |                                            | 4.5        | Binary search + 2pointers                |
| ~~1930. Unique Length-3 Palindromic Subsequences~~ |                                            | 2.5        | Greedy                                   |
| 2207. Maximize Number of Subsequences in a String  |                                            | 4          |                                          |
| 2407. Longest Increasing Subsequence II            | 307、315、406、699、1649、2179、2213、2407 | 5          | ***Segment Tree problem***，O(n)         |



## Array

| 题目                                              | difficulty |                          |
| ------------------------------------------------- | ---------- | ------------------------ |
| 3. Longest Substring Without Repeating Characters | 3.5        | Sliding Window           |
| 2401. Longest Nice Subarray                       | 3.5        | Sliding Window + bitwise |
|                                                   |            |                          |
|                                                   |            |                          |
|                                                   |            |                          |

## Tree

| 题目                                         | 相似题目      | difficulty | Language | space and time complexity |
| -------------------------------------------- | ------------- | ---------- | -------- | ------------------------- |
| 144. Binary Tree Preorder Traversal          | 145、589、590 | 3/5        | Java     | traversal by recursing    |
| 987                                          | ~~429~~、1302 | 3/5        | java     | collect nodes             |
| 669. Trim a Binary Search Tree               | 1325          | 3.5/5      |          | prune tree                |
| 112. Path Sum                                | 437           | 3.5/5      | Java     | Path                      |
| 236. Lowest Common Ancestor of a Binary Tree | 235           |            |          | postOrder                 |
| 297                                          | 499           | 4/5        | Java     | constrcut tree            |
| 337                                          | 968,979       | 4.5/5      | Java     | Dp/memization             |





## DFS

| 题目 | 相似题目 | difficulty | Language | space and time complexity                                    |
| ---- | -------- | ---------- | -------- | ------------------------------------------------------------ |
| 133  | 138      | 3          | Java     | DFS: O(n + m)、T(n + m)、visitedMap/visitedSet<br />BFS: Queue+visitedMap |
| 200  | 695、827 | 2          |          | DFS + visitedMap                                             |



## BFS

| 题目                           | 相似题目 | difficulty | Language | space and time complexity |
| ------------------------------ | -------- | ---------- | -------- | ------------------------- |
| 4. Median of Two Sorted Arrays |          | 3.5        | Java\    | binary search             |
| 433. Minimum Genetic Mutation  |          | 3          |          | BFS                       |
| 456. 132 Pattern               |          |            |          | Balanced BFS<br />        |
|                                |          |            |          |                           |
|                                |          |            |          |                           |
|                                |          |            |          |                           |
|                                |          |            |          |                           |

## Graph

| 题目                                                   | 相似题目 | difficulty | Language | space and time complexity                                    |
| ------------------------------------------------------ | -------- | ---------- | -------- | ------------------------------------------------------------ |
| 133                                                    | 138      | 3          | Java     | DFS: O(n + m)、T(n + m)、visitedMap/visitedSet<br />BFS: Queue+visitedMap |
| 200                                                    | 695、827 | 2          |          | DFS + visitedMap                                             |
| ~~1061. Lexicographically Smallest Equivalent String~~ | 2477     | 3.5        |          |                                                              |
|                                                        |          |            |          |                                                              |
|                                                        |          |            |          |                                                              |
|                                                        |          |            |          |                                                              |
|                                                        |          |            |          |                                                              |
|                                                        |          |            |          |                                                              |
|                                                        |          |            |          |                                                              |

## Sliding Window

| 题目                                                         | 相似题目                                                     | difficulty | space and time complexity |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ---------- | ------------------------- |
| [2653. Sliding Subarray Beauty](https://zxi.mytechroad.com/blog/sliding-window/leetcode-2653-sliding-subarray-beauty/) | [LeetCode 2156. Find Substring With Given Hash Value](https://zxi.mytechroad.com/blog/sliding-window/leetcode-2156-find-substring-with-given-hash-value/) | 2.5        |                           |
| 3090.  Maximum Length Substring With Two Occurrences         |                                                              |            |                           |

