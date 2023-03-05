# 算法题

## LeetCode

### 1.两数之和

### 2.两数相加

描述：

- 给出两个**非空**的链表用来表示两个非负的整数。其中，它们各自的位数是按照**逆序**的方式存储的，并且它们的每个节点只能存储**一位**数字。
- 如果，我们将这两个数相加起来，则会返回一个新的链表来表示它们的和。
- 您可以假设除了数字 0 之外，这两个数都不会以 0 开

示例：

```
输入：(2 -> 4 -> 3) + (5 -> 6 -> 4)
输出：7 -> 0 -> 8
原因：342 + 465 = 807
```

解答：

```java
/*
	1.cur、pre都是辅助指针
	2.pre是自己单独定义的一个头结点，返回他的next那就是他的下一个节点就是我们要的链表了， curr是为了操作pre的next而存在的，如果直接移动pre节点，那最后就拿不到返回值了
	3.使用预先指针的目的在于链表初始化时无可用节点值，而且链表构造过程需要指针移动，进而会导致头指针丢失，无法返回结果
*/
class Solution {
    public ListNode addTwoNumbers(ListNode l1, ListNode l2) {
        ListNode pre = new ListNode(0);//初始化预先指针，值为0
        ListNode cur = pre;//当前指针指向预先指针
        int carry = 0;//进位值初始化为0
        while(l1 != null || l2 != null) {//两个节点只要有一个不为空就能继续往下走
            //为空补0，否则赋值
            int x = l1 == null ? 0 : l1.val;
            int y = l2 == null ? 0 : l2.val;
            //和为两个链表当前节点值+进位值
            int sum = x + y + carry;
            
            carry = sum / 10;//carry = x+y>=10?1:0
            sum = sum % 10;
            
            cur.next = new ListNode(sum);//当前指针的下一个节点值为sum(有点懵)

            cur = cur.next;//cur指向后一个节点
            //l1、l2指向后一个节点
            if(l1 != null)
                l1 = l1.next;
            if(l2 != null)
                l2 = l2.next;
        }
        //链表节点都遍历完后进位值仍为1，那么增加一个节点，值为1（两边节点遍历完了，不会有求和了，只有carry）
        if(carry == 1) {
            cur.next = new ListNode(carry);
        }
        return pre.next;//返回真正的链表节点
    }
}

```

