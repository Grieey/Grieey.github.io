---
title: "Go语言 Merge two sorted lists 题目实现"
date: 2018-11-07T21:18:46+08:00
tags: 
    - 算法
    - Go
categories: 
    - 算法
---

# 题目描述

Merge two sorted linked lists and return it as a new list. The new list should be made by splicing together the nodes of the first two lists.

Example:

	Input: 1->2->4, 1->3->4
	Output: 1->1->2->3->4->4
    
给出两个已经排序了的链表数组，将两个数组合并为一个

# 解决思路

我想的是将第一个数组作为最后返回的数组，那么只需要将这个数组遍历，在遍历得时候和另一个数组的值进行比较。遍历过程中，将第二个数组的值按序插入到第一个数组中。

# 代码实现

```go
/**
 * Definition for singly-linked list.
 * type ListNode struct {
 *     Val int
 *     Next *ListNode
 * }
 */
func mergeTwoLists(l1 *ListNode, l2 *ListNode) *ListNode {
	// 如果第一个数组为空，则直接返回第二个数组
    if l1 == nil {
        return l2
    }
    
    // 第一个数组的游标
    var cur *ListNode = new (ListNode)
    cur = l1  // 指向头结点
    
    for cur != nil && l2 != nil {
        var temp *ListNode = new (ListNode)
        // 将l1的值和l2的进行比较
        // 如果l1当前节点值大于了l2当前节点值，
        // 将l2当前节点值插入到l1当前节点前，
        if cur.Val >= l2.Val {
            temp.Val = cur.Val
            temp.Next = cur.Next
            cur.Val = l2.Val
            cur.Next = temp
            // 同时游标和l2向后移
            cur = cur.Next
            l2 = l2.Next
        }else{
        	// 如果l2当前节点值大于了l1，
            // 那么游标向后移动，直到找到大于l2的当前节点值得节点
            
            if cur.Next == nil {
            	// l1找不到比l2当前节点值大的节点
                // 直接将l2后续节点整个连接到l1上并返回
                cur.Next = l2
                return l1
            }else{
                cur = cur.Next
            }
        }
    }
    return l1
}
```