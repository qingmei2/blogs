# 每日一题 |

> Date: 2021-03-22  Week13

### 1、位1的个数

* 难度：`Easy`

#### 题目描述

编写一个函数，输入是一个无符号整数（以二进制串的形式），返回其二进制表达式中数字位数为 '1' 的个数（也被称为汉明重量）。

提示：

请注意，在某些语言（如 Java）中，没有无符号整数类型。在这种情况下，输入和输出都将被指定为有符号整数类型，并且不应影响您的实现，因为无论整数是有符号的还是无符号的，其内部的二进制表示形式都是相同的。
在 Java 中，编译器使用二进制补码记法来表示有符号整数。因此，在上面的 示例 3 中，输入表示有符号整数 -3。
 

> 示例 1：  
> 输入：00000000000000000000000000001011  
> 输出：3  
> 解释：输入的二进制串 00000000000000000000000000001011 中，共有三位为 '1'。  

#### 解题思路及实现

标准的位运算。

```java
public class Solution {
    public int hammingWeight(int n) {
        int count = 0;
        int num = 1;

        while (num <= 32) {
            if ((n & (1 << num)) != 0) {
                count++;
            }
            num ++;
        }
        return count;
    }
}
```

因为位运算的算法接触的比较少，官方还提供了一个优化的解法更有意思一些：

观察这个运算：`n & (n−1)`，其预算结果恰为把 `n` 的二进制位中的最低位的 1 变为 0 之后的结果。

这样我们可以利用这个位运算的性质加速我们的检查过程，在实际代码中，我们不断让当前的 `n 与 n - 1` 做与运算，直到 `n` 变为 `0` 即可。因为每次运算会使得 `n` 的最低位的 `1` 被翻转，因此运算次数就等于 `n` 的二进制位中 `1` 的个数。

```java
public class Solution {
    public int hammingWeight(int n) {
        int count = 0;

        while (n != 0) {
            count++;
            n = n & (n - 1);
        }
        return count;
    }
}
```

个人觉得这种解法比较巧妙，但实战中价值不高，同时这种解法需要注意是`（n != 0）`而非`（n > 0）`, 因为`n`可能是负值（即32位为1）。

### 1.1 环形链表

* 难度：`Easy`

#### 题目描述

给定一个链表，判断链表中是否有环。

如果链表中有某个节点，可以通过连续跟踪 `next` 指针再次到达，则链表中存在环。 为了表示给定链表中的环，我们使用整数 `pos` 来表示链表尾连接到链表中的位置（索引从 `0` 开始）。 如果 `pos` 是 `-1`，则在该链表中没有环。注意：`pos` 不作为参数进行传递，仅仅是为了标识链表的实际情况。

如果链表中存在环，则返回 `true` 。 否则，返回 `false` 。

#### 解题思路及实现

经典的环形链表题目，作为回顾链表类题型的开篇，这里不使用常规的`HashMap`进行解答，直接使用弗洛伊德判断算法。

```java
public class Solution {
    public boolean hasCycle(ListNode head) {
        if (head == null || head.next == null) {
            return false;
        }
        ListNode slow = head;
        ListNode fast = head.next;

        while (fast != null && fast.next != null) {
            if (slow == fast) {
                return true;
            }
            slow = slow.next;
            fast = fast.next.next;
        }

        return false;
    }
}
```

最麻烦的还是边界条件的判断，写了5遍才过，心累。

## 参考 & 感谢

文章绝大部分内容节选自`LeetCode`，例题：

* https://leetcode-cn.com/problems/number-of-1-bits/
* https://leetcode-cn.com/problems/linked-list-cycle/

## 关于我

Hello，我是 [却把清梅嗅](https://github.com/qingmei2) ，如果您觉得文章对您有价值，欢迎 ❤️，也欢迎关注我的 [博客](https://blog.csdn.net/mq2553299) 或者 [GitHub](https://github.com/qingmei2)。

如果您觉得文章还差了那么点东西，也请通过 **关注** 督促我写出更好的文章——万一哪天我进步了呢？

* [我的Android学习体系](https://github.com/qingmei2/blogs)
* [关于文章纠错](https://github.com/qingmei2/blogs/blob/master/error_collection.md)
* [关于知识付费](https://github.com/qingmei2/blogs/blob/master/appreciation.md)
* [关于《反思》系列](https://github.com/qingmei2/blogs/blob/master/src/%E5%8F%8D%E6%80%9D%E7%B3%BB%E5%88%97/thinking_in_android_index.md)
