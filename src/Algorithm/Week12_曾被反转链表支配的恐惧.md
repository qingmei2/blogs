# 每日一题 | 曾被反转链表支配的恐惧

> 前段时间看到掘金平台推出了每日算法打卡的活动，感觉挺好，好巧不巧打开`LeetCode`的每日一题刚好是 **反转链表**，这道去年年初面试字节跳动挂掉的算法题，趁这个机会把之前算法的记忆捡回来。

> Date: 2021-03-21  Week12

### 1、反转链表 II

* 难度：`Medium`

#### 题目描述

给你单链表的头指针 `head` 和两个整数 `left` 和 `right` ，其中 `left <= right` 。请你反转从位置 `left` 到位置 `right` 的链表节点，返回 **反转后的链表** 。

> 示例1  
>输入：head = [1,2,3,4,5], left = 2, right = 4  
>输出：[1,4,3,2,5]  

> 示例2  
>输入：head = [5], left = 1, right = 1  
>输出：[5]  

#### 解题思路及实现

链表这种题，真是下意识不想用指针，在纸上写写画画半天，还不一定能写对。

最简单的方式还是用栈，思路很简单，第一次迭代中先找到左边下一个就要反转的节点，该节点记录为左边界节点。

第二次从起始节点开始迭代，用栈`push`所有需要反转的节点，直到找到右边无需反转的节点，该节点记录为右边界节点。

最后一次迭代，将栈中所有节点依次`pop`然后首位连接起来，进行返回即可。

> 以示例1为例，1就是左边界节点，5是右边界节点，栈中依次存储[2,3,4]，之后依次弹出并进行连接，即输出[1,4,3,2,5]。

这道题来回理了2小时才写出来，重点有2：

* 1、边界问题，如何对左右2个边界进行准确的记录；
* 2、正常情况下，`head`应该还是第一个`Node`不变；但如果左边界为`null`，即第一个元素就是要被反转的节点，那么返回的`head`应该是右边界的`Node`。

```java
class Solution {
    public ListNode reverseBetween(ListNode head, int left, int right) {
        Stack<ListNode> stack = new Stack<>();

        int curr = 1;

        ListNode leftNode = null;
        ListNode currNode = head;

        // 1、找到第一个要反转的节点，并指向 currNode
        while (curr < left) {
            curr++;
            leftNode = currNode;
            if (currNode == null)
                break;
            else
                currNode = currNode.next;
        }

        // 这时 leftNode 就是左边界的节点，保持不动

        ListNode rightNode = currNode.next;

        // 2、找到最后一个要反转的节点，并指向 currNode
        while (curr <= right) {
            curr++;
            stack.push(currNode);
            currNode = rightNode;
            if (rightNode == null)
                break;
            else
                rightNode = rightNode.next;
        }
        // 这时 curr 就是右边界的节点
        ListNode start = leftNode == null ? null : head;
        while (!stack.isEmpty()) {
            ListNode node = stack.pop();
            if (leftNode == null) {
                leftNode = node;
                start = leftNode;
            } else {
                leftNode.next = node;
                leftNode = leftNode.next;
            }
        }

        leftNode.next = currNode;

        return start;
    }
}
```

### 2、设计停车系统

* 难度：`Easy`

#### 题目描述

请你给一个停车场设计一个停车系统。停车场总共有三种不同大小的车位：大，中和小，每种尺寸分别有固定数目的车位。

请你实现 `ParkingSystem` 类：

`ParkingSystem(int big, int medium, int small)` 初始化 `ParkingSystem` 类，三个参数分别对应每种停车位的数目。
`bool addCar(int carType)` 检查是否有 `carType` 对应的停车位。 `carType` 有三种类型：大，中，小，分别用数字 1， 2 和 3 表示。一辆车只能停在 `carType` 对应尺寸的停车位中。如果没有空车位，请返回 `false` ，否则将该车停入车位并返回 `true` 。

#### 解题思路及实现

这道题的难度连热身都不够，我甚至怀疑这是`leetcode`最简单的算法题没有之一，2分钟整出来，顺便复习一下反转链表。

```java
class ParkingSystem {

    private int[] arr;

    public ParkingSystem(int big, int medium, int small) {
        this.arr = new int[]{big, medium,small};
    }

    public boolean addCar(int carType) {
        if (this.arr[carType - 1] > 0) {
            this.arr[carType - 1] = this.arr[carType -1] -1;
            return true;
        }
        return false;
    }
}
```

### 2.1、反转链表

* 难度：`Easy`

#### 题目描述

反转一个单链表。

>示例:  
>输入: 1->2->3->4->5->NULL  
>输出: 5->4->3->2->1->NULL  

#### 解题思路及实现

反转链表的经典题，长时间不接触，不画图进行整理依旧做不出来，攻克这种题目类型的思路暂时只有一个：无他，惟手熟尔。

```java
public ListNode reverseList(ListNode head) {
    if (head == null) return head;

    // 这里还是使用比较熟悉的指针来解决
    ListNode curr = head;       // 指针 curr 代表即将反转的位置

    while (curr.next != null) {
        ListNode after = curr.next;  // 指针 after 是一个占位符，用于临时存储下一个的位置

        // 对当前节点进行反转
        curr.next = after.next;
        after.next = head;
        head = after;
    }
    return head;
}
```

### 3、逆波兰表达式求值

* 难度：`Medium`

#### 题目描述

根据逆波兰表示法，求表达式的值。

有效的算符包括 `+、-、*、/` 。每个运算对象可以是整数，也可以是另一个逆波兰表达式。

说明：
整数除法只保留整数部分。
给定逆波兰表达式总是有效的。换句话说，表达式总会得出有效数值且不存在除数为 0 的情况。
 
> 示例 1：
>
> 输入：tokens = ["2","1","+","3","*"]  
> 输出：9  
> 解释：该算式转化为常见的中缀算术表达式为：((2 + 1) * 3) = 9  

> 示例 2：  
>
> 输入：tokens = ["4","13","5","/","+"]  
> 输出：6  
> 解释：该算式转化为常见的中缀算术表达式为：(4 + (13 / 5)) = 6  

**逆波兰表达式**：

逆波兰表达式是一种后缀表达式，所谓后缀就是指算符写在后面。

平常使用的算式则是一种中缀表达式，如 `( 1 + 2 ) * ( 3 + 4 )` 。
该算式的逆波兰表达式写法为 `( ( 1 2 + ) ( 3 4 + ) * )` 。
逆波兰表达式主要有以下两个优点：

去掉括号后表达式无歧义，上式即便写成 `1 2 + 3 4 + *` 也可以依据次序计算出正确结果。
**适合用栈操作运算**：遇到数字则入栈；遇到算符则取出栈顶两个数字进行计算，并将结果压入栈中。

#### 解题思路及实现

题目还没看完，就被题干中一句 **适合用栈操作运算** 给震惊了，好家伙等于直接给答案，感觉题目难度瞬间下降成了简单难度。

```java
class Solution {
    public int evalRPN(String[] tokens) {
        Stack<Integer> stack = new Stack<>();
        for (int index = 0; index < tokens.length; index++) {
            String elem = tokens[index];
            if (elem.equals("+")) {
                int num1 = stack.pop();
                int num2 = stack.pop();
                stack.push(num2 + num1);
            } else if (elem.equals("-")) {
                int num1 = stack.pop();
                int num2 = stack.pop();
                stack.push(num2 - num1);
            } else if (elem.equals("*")) {
                int num1 = stack.pop();
                int num2 = stack.pop();
                stack.push(num2 * num1);
            } else if (elem.equals("/")) {
                int num1 = stack.pop();
                int num2 = stack.pop();
                stack.push( num2 / num1);
            } else {
                // 数字，直接入栈
                stack.push(Integer.valueOf(elem));
            }
        }
        return stack.pop();
    }
}
```

### 4、矩阵置零

* 难度：`Medium`

#### 题目描述

给定一个 m x n 的矩阵，如果一个元素为 0 ，则将其所在行和列的所有元素都设为 0 。请使用**原地算法**。

进阶：

一个直观的解决方案是使用  O(mn) 的额外空间，但这并不是一个好的解决方案。  
一个简单的改进方案是使用 O(m + n) 的额外空间，但这仍然不是最好的解决方案。  
你能想出一个仅使用常量空间的解决方案吗？  

#### 解题思路及实现

**原地算法**就是直接对入参的数组进行更新，然后返回，这在`Android`中也很常见（比如的`Rect`类）。

思路1，直接原地复制一个数组进行记录，时间和空间复杂度都是`O(mn)`。

思路2，我们可以用两个标记数组分别记录每一行和每一列是否有零出现。

具体地，我们首先遍历该数组一次，如果某个元素为`0`，那么就将该元素所在的行和列所对应标记数组的位置置为 `true`。最后我们再次遍历该数组，用标记数组更新原数组即可。

```java
class Solution {
    public void setZeroes(int[][] matrix) {
        int row = matrix.length;
        int col = matrix[0].length;

        boolean[] rowArray = new boolean[row];
        boolean[] colArray = new boolean[col];

        for (int i = 0; i < row; i++) {
            for (int j = 0; j < col; j++) {
                if (matrix[i][j] == 0) {
                    rowArray[i] = true;
                    colArray[j] = true;
                }
            }
        }

        for (int i = 0; i < row; i++) {
            for (int j = 0; j < col; j++) {
                if (rowArray[i] || colArray[j]) {
                    matrix[i][j] = 0;
                }
            }
        }
    }
}
```

官方题解下面给出了空间复杂度为`O(1)`的方案，我看了看，和思路2差不多，但用于记录的2个1为数组，通过使用入参中的`matrix[0][]`和`matrix[][0]`来记录，最后进行还原，但代码可读性并不佳，客户端实战应用中一般无需这么吝啬，就忽略了，有兴趣的读者可以研究下。

## 参考 & 感谢

文章绝大部分内容节选自`LeetCode`，例题：

* https://leetcode-cn.com/problems/reverse-linked-list-ii/
* https://leetcode-cn.com/problems/design-parking-system/
* https://leetcode-cn.com/problems/reverse-linked-list/
* https://leetcode-cn.com/problems/evaluate-reverse-polish-notation/
* https://leetcode-cn.com/problems/set-matrix-zeroes/solution/ju-zhen-zhi-ling-by-leetcode-solution-9ll7/

## 关于我

Hello，我是 [却把清梅嗅](https://github.com/qingmei2) ，如果您觉得文章对您有价值，欢迎 ❤️，也欢迎关注我的 [博客](https://blog.csdn.net/mq2553299) 或者 [GitHub](https://github.com/qingmei2)。

如果您觉得文章还差了那么点东西，也请通过 **关注** 督促我写出更好的文章——万一哪天我进步了呢？

* [我的Android学习体系](https://github.com/qingmei2/blogs)
* [关于文章纠错](https://github.com/qingmei2/blogs/blob/master/error_collection.md)
* [关于知识付费](https://github.com/qingmei2/blogs/blob/master/appreciation.md)
* [关于《反思》系列](https://github.com/qingmei2/blogs/blob/master/src/%E5%8F%8D%E6%80%9D%E7%B3%BB%E5%88%97/thinking_in_android_index.md)
