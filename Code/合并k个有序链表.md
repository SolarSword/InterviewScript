# 合并k个有序链表
[力扣23](https://leetcode.cn/problems/merge-k-sorted-lists/description/)
升序还是降序就是一个大于小于号的问题。

```Golang
type ListNode struct {
    Val int
    Next *ListNode
}

func mergeLists(list1, list2 *ListNode) *ListNode {
    res := &ListNode{}
    cursor := res 
    cursor1 := list1
    cursor2 := list2
    for cursor1 != nil && cursor2 != nil {
        if cursor1.Val >= cursor2.Val {
            cursor.Next= cursor1
            cursor = cursor.Next
            cursor1 = cursor1.Next
        } else {
            cursor.Next= cursor2
            cursor = cursor.Next
            cursor2 = cursor2.Next
        }
    }
    if cursor1 == nil {
        cursor.Next = cursor2
    }
    if cursor2 == nil {
        cursor.Next = cursor1
    }
    return res
}

func mergeKListsProcessor(lists []*ListNode) *ListNode {
    n := len(lists)

    if n == 0 {
        return nil
    }
    if n == 1 {
        return lists[0]
    }
    if n == 2 {
        return mergeLists(lists[0], lists[1])
    }

    half := n/2
    remains := ListNode{mergeKListsProcessor(lists[:half]), mergeKListsProcessor(lists[half:]),}

    return mergeKListsProcessor(remains)
}
```