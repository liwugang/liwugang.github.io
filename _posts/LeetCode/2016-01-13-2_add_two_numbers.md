---
layout: post

title:  2-Add Two Numbers @LeetCode
date:   2016-01-13 21:08:00 +0800
categories: LeetCode
tag: LeetCode
---

* content
{:toc}


题目
====================
![graph]({{"/assets/pictures/leetcode/2-1.png" | prepend:site.baseurl}})

思路
====================
## 题目中得到的信息有：

1. 这是两个非负数，每位分别保存在链表的一个结点上；
2. 逆序保存，从低位到高位依次。

一般整数的相加都是从低往高进行，和保存的顺序一致，因此一次遍历就可完成，可以看出这道题目不难。

## C算法
```c++
/**
 * Definition for singly-linked list. 
 * struct ListNode {
 *     int val;
 *     struct ListNode *next;
 * };
 */
struct ListNode* addTwoNumbers(struct ListNode* l1, struct ListNode* l2) {
    struct ListNode *ret, *p, *prev;
    p = ret = prev = NULL;
    int flag = 0;
	//先是两个整数相加
    while((l1 != NULL) && (l2 != NULL) ) { 
        p = (struct ListNode *)malloc(sizeof(struct ListNode));
        p->val = l1->val + l2->val + flag;
		flag = p->val / 10;
        p->val -= (flag > 0 ? 10 : 0);
        if (prev != NULL) prev->next = p;
        prev = p;
        if (ret == NULL) ret = p;
        l1 = l1->next;
        l2 = l2->next;
    }
	//若是l1还有结点，添加上去
    while(l1 != NULL) {
        p = (struct ListNode *)malloc(sizeof(struct ListNode));
         p->val = l1->val  + flag;
         flag = p->val / 10;
        p->val -= (flag > 0 ? 10 : 0);
        l1 = l1->next; 
         if (prev != NULL) prev->next = p;
        if (ret == NULL) ret = p;
        prev->next = p;
        prev = p;
    }
	//若是l2还有结点，添加上去
    while(l2 != NULL) {
        p = (struct ListNode *)malloc(sizeof(struct ListNode));
         p->val = l2->val  + flag;
         flag = p->val / 10;
        p->val -= (flag > 0 ? 10 : 0);
        l2 = l2->next; 
         if (prev != NULL) prev->next = p;
         if (ret == NULL) ret = p;
        prev->next = p;
        prev = p;
    }
	//最后可能会有进位，要考虑到
    if (flag != 0) {
        p = (struct ListNode *)malloc(sizeof(struct ListNode));
        p->val = flag;
        prev->next = p;
        prev = p;
        flag = 0;
    }
    prev->next = NULL;
    return ret;
}
```
## 结果
![graph]({{"/assets/pictures/leetcode/2-2.png" | prepend:site.baseurl}})
