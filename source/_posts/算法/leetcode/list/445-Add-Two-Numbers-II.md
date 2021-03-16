---
title: 445.Add Two Numbers II
date: 2020-12-30 18:34:50
tags: [leetcode]
---



ref:

1. 我觉得最好的解法
   https://leetcode.com/problems/add-two-numbers-ii/discuss/687339/Java-O(N)-solution-with-follow-up-question-no-recursion-no-stacks



```Java
/**
 * Definition for singly-linked list.
 * public class ListNode {
 *     int val;
 *     ListNode next;
 *     ListNode() {}
 *     ListNode(int val) { this.val = val; }
 *     ListNode(int val, ListNode next) { this.val = val; this.next = next; }
 * }
 */
class Solution {
    public ListNode addTwoNumbers(ListNode l1, ListNode l2) {
        int size1 = size(l1), size2 = size(l2);
        
        ListNode sumHead = null;
        while(l1!=null || l2!=null){
            
            if(size1>size2){
                ListNode buf = new ListNode(l1.val);
                buf.next = sumHead;
                sumHead = buf;
                
                l1 = l1.next;
                size1--;
            }else if(size1<size2){
                ListNode buf = new ListNode(l2.val);
                buf.next = sumHead;
                sumHead = buf;
                
                l2 = l2.next;
                size2--;
            }else{
                ListNode buf = new ListNode(l1.val+l2.val);
                buf.next = sumHead;
                sumHead = buf;
                
                l1 = l1.next;
                l2 = l2.next;
            }
        }
        
        ListNode pre = null;
        int carry = 0;
        while(sumHead!=null){
            ListNode buf = sumHead.next;
            
            int carry2 = (sumHead.val+carry)/10;
            sumHead.next = pre;
            sumHead.val = (sumHead.val+carry)%10;
            carry = carry2;
            pre = sumHead;
            sumHead = buf;

        }
        
        if(carry!=0){
            ListNode buf = new ListNode(1);
            buf.next = pre;
            pre = buf;
        }
        
        return pre;
        
    }
    
    private int size(final ListNode node){
        ListNode list = node;
        int size = 0;
        while(list!=null){
            size++;
            list = list.next;
        }
        return size;
    }
}
```

