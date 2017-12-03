# 每 K 个一组翻转链表



这道题是 [LeetCode#25](https://leetcode.com/problems/reverse-nodes-in-k-group/description/) ，要求是给定一个链表，每逢 K 个节点就将其翻转。原题描述以及例子如下：

```
Given a linked list, reverse the nodes of a linked list k at a time and return its modified list.

k is a positive integer and is less than or equal to the length of the linked list. If the number of nodes is not a multiple of k then left-out nodes in the end should remain as it is.

You may not alter the values in the nodes, only nodes itself may be changed.

Only constant memory is allowed.

For example,
Given this linked list: 1->2->3->4->5

For k = 2, you should return: 2->1->4->3->5

For k = 3, you should return: 3->2->1->4->5
```





链表的题目总是容易让人联想到**递归**，这道题也不例外。总体思想为找到第 K + 1个节点，将从头结点到第 K 个节点进行翻转，而 K 之后的节点顺序交给下一层递归方法。



```java
public ListNode reverseKGroup(ListNode head, int k) {
        ListNode cur = head;
        int count = 0;
    
        // 找到 k + 1 个节点
        while (cur != null && count != k) {
            cur = cur.next;
            count++;
        }
        
        if (count == k) {
            // 递归
            cur = reverseKGroup(cur, k);
            
            // 翻转 head-> K 之间的节点
            while (count-- > 0) {
                ListNode tmp = head.next;
                head.next = cur;
                cur = head;
                head = tmp;
            }
            
            head = cur;
        }
        
        return head;
        
    }
```



这段代码中最重要的部分当然是 翻转节点这一部分，我们一起来看一下这段程序是如何运行的。

假设链表 ***head = [1, 2, 3, 4, 5], k = 3***



直接看 *while* 循环体

此时的链表是这样的： 

```
 1 -> 2 -> 3 -> 4 -> 5
 ↑              ↑
head           cur
```

此时 count = 3，会执行三次循环体，第一次之后，链表是这样的：

```
          cur       
tmp        1
 ↓         ↓
 2 -> 3 -> 4 -> 5
 ↑         
head     
```

count = 2， 再次执行循环体，之后链表是这样的：

```
          cur       
tmp   1 <- 2
 ↓    ↓
 3 -> 4 -> 5
 ↑         
head    
```

最后一遍执行循环体：

```
    tmp        
     ↓         
5 <- 4 <- 1 <- 2 <- 3 
     ↑              ↑  
    head           cur
```



之后将 cur 赋值给 head，将 head 返回。