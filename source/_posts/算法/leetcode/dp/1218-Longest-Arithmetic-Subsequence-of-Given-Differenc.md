---
title: 1218.Longest Arithmetic Subsequence of Given Differenc
date: 2020-12-31 23:00:37
tags: [leetcode, dp]
---



方式1:  subsequence和dp结合，传统的2次iterative求解 ( Time Limit exceeded)

```golang
import "fmt"
func longestSubsequence(arr []int, difference int) int {
    n := len(arr)
    dp := make([]int, n);
    for i:=0; i<n; i++{
        dp[i] = 1;
    }
  
    var ans int = 1;
   
    for i:=1; i<n; i++ {
         var subAns int = 0;
        for j:=0; j<i; j++{
            var diff =arr[i] - arr[j];
            
            if(diff == difference){
                //fmt.Println("i=%d, j=%d",i,j)
                if(subAns>dp[j]+1 ){
                    dp[i] = subAns
                }else{
                    dp[i] = dp[j]+1
                }
            }
        }
        
        if(ans<dp[i]){
            ans = dp[i]
        }
    }
    
    return ans;
}
```

