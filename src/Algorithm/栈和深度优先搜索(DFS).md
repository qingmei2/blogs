# 栈和深度优先搜索(DFS)

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/blogs/algorithm/1/image.8oz8tmpzv0i.png)

与 `BFS` 类似，深度优先搜索`（DFS）`是用于在树/图中遍历/搜索的另一种重要算法。也可以在更抽象的场景中使用。

正如树的遍历中所提到的，我们可以用 `DFS` 进行 **前序遍历**，**中序遍历** 和 **后序遍历**。在这三个遍历顺序中有一个共同的特性：**除非我们到达最深的结点，否则我们永远不会回溯**。

这也是 `DFS` 和 `BFS` 之间最大的区别，**`BFS`永远不会深入探索，除非它已经在当前层级访问了所有结点**。

## 模版

### 递归模版

有两种实现 `DFS` 的方法。第一种方法是进行递归:

```java
boolean DFS(Node cur, Node target, Set<Node> visited) {
    return true if cur is target;
    for (next : each neighbor of cur) {
        if (next is not in visited) {
            add next to visted;
            return true if DFS(next, target, visited) == true;
        }
    }
    return false;
}
```

当我们递归地实现 `DFS` 时，似乎不需要使用任何栈。但实际上，我们使用的是由系统提供的隐式栈，也称为调用栈（Call Stack）。

### 显式栈模板

递归解决方案的优点是它更容易实现。 但是，存在一个很大的缺点：如果递归的深度太高，你将遭受堆栈溢出。 在这种情况下，您可能会希望使用 `BFS`，或使用 **显式栈** 实现 `DFS`。

```java
boolean DFS(int root, int target) {
    Set<Node> visited;
    Stack<Node> s;
    add root to s;
    while (s is not empty) {
        Node cur = the top element in s;
        return true if cur is target;
        for (Node next : the neighbors of cur) {
            if (next is not in visited) {
                add next to s;
                add next to visited;
            }
        }
        remove cur from s;
    }
    return false;
}
```

## 例题

### 1.岛屿数量

* 难度：`Medium`

#### 题目描述

给定一个由 `'1'`（陆地）和 `'0'`（水）组成的的二维网格，计算岛屿的数量。一个岛被水包围，并且它是通过水平方向或垂直方向上相邻的陆地连接而成的。你可以假设网格的四个边均被水包围。

示例 1:

> 输入:
11110  
11010  
11000  
00000  
> 输出: 1

示例 2:

> 输入:
11000  
11000  
00100  
00011  
输出: 3

#### 解题思路及实现

笔者曾经在 [这篇文章](https://github.com/qingmei2/blogs/issues/35) 中展示了如何使用 `BFS` 解决这道题，事实上该题使用 `DFS` 更简单，因为前者还需要一个队列维护 **广度优先搜索** 过程中搜索的层级信息。

使用 `DFS` 解题如下：

```java
public class B200NumIslands {

    public int numIslands(char[][] grid) {
        int nr = grid.length;
        if (nr == 0) return 0;
        int nc = grid[0].length;
        if (nc == 0) return 0;

        int result = 0;

        for (int r = 0; r < nr; r++) {
            for (int c = 0; c < nc; c++) {
                if (grid[r][c] == '1') {
                    result++;
                    dfs(grid, r, c);
                }
            }
        }

        return result;
    }

    private void dfs(char[][] grid, int r, int c) {
        int nr = grid.length;
        int nc = grid[0].length;

        // 排除边界外的情况
        if (r >= nr || c >= nc || r < 0 || c < 0) return;
        // 排除边界外指定位置为 '0' 的情况
        if (grid[r][c] == '0') return;

        // 该位置为一个岛，标记为已探索
        grid[r][c] = '0';

        dfs(grid, r - 1, c);  // top
        dfs(grid, r + 1, c);  // bottom
        dfs(grid, r, c - 1);  // left
        dfs(grid, r, c + 1);  // right
    }
}
```

### 2.克隆图

* 难度：`Medium`

#### 题目描述

给你无向 **连通** 图中一个节点的引用，请你返回该图的 **深拷贝（克隆）**。

图中的每个节点都包含它的值 `val（int）` 和其邻居的列表（`list[Node]`）。

```java
class Node {
    public int val;
    public List<Node> neighbors;
}
```

> 更详细的题目描述参考 [这里](https://leetcode-cn.com/problems/clone-graph/) :
> https://leetcode-cn.com/problems/clone-graph/

#### 解题思路及实现

题目比较难理解，需要注意的是：

1. 因为是 **深拷贝** ，因此所有节点都需要通过 `new` 进行实例化，即需要遍历图中的每个节点，因此解决方案就浮现而出了，使用 `DFS` 或者 `BFS` 即可；  
2. 对每个已经复制过的节点进行标记，避免无限循环导致堆栈的溢出。

用 `DFS` 实现代码如下：

```java
class Solution {
    public Node cloneGraph(Node node) {
        HashMap<Node,Node> map = new HashMap<>();
        return dfs(node, map);
    }

    private Node dfs(Node root, HashMap<Node,Node> map) {
        if (root == null) return null;
        if (map.containsKey(root)) return map.get(root);

        Node clone = new Node(root.val, new ArrayList());
        map.put(root, clone);
        for (Node nei: root.neighbors) {
            clone.neighbors.add(dfs(nei, map));
        }
        return clone;
    }
}
```

### 3.目标和

* 难度：`Medium`

#### 题目描述

给定一个非负整数数组，a1, a2, ..., an, 和一个目标数，S。现在你有两个符号 + 和 -。对于数组中的任意一个整数，你都可以从 `+` 或 `-` 中选择一个符号添加在前面。

返回可以使最终数组和为目标数 S 的所有添加符号的方法数。

示例 1:

```
输入: nums: [1, 1, 1, 1, 1], S: 3
输出: 5
解释:

>-1+1+1+1+1 = 3
+1-1+1+1+1 = 3
+1+1-1+1+1 = 3
+1+1+1-1+1 = 3
+1+1+1+1-1 = 3

一共有5种方法让最终目标和为3。
```

注意:

>1.数组非空，且长度不会超过20。
>2.初始的数组的和不会超过1000。
>3.保证返回的最终结果能被32位整数存下。

#### 解题思路及实现

说实话这道题真没想到使用 `DFS` 暴力解决，还是经验太少了，这道题暴力解法是完全可以的，而且不会超时，因为题目中说了数组长度不会超过20，20个数字的序列，组合方式撑死了也就 `2^20` 种组合:

```java
public class Solution {

    int count = 0;

    public int findTargetSumWays(int[] nums, int S) {
        dfs(nums, 0, 0, S);
        return count;
    }

    private void dfs(int[] nums, int index, int sum, int S) {
        if (index == nums.length) {
            if (sum == S) count++;
        } else {
            dfs(nums, index + 1, sum + nums[index], S);
            dfs(nums, index + 1, sum - nums[index], S);
        }
    }
}
```

### 4.二叉树的中序遍历

* 难度：`Medium`

#### 题目描述

给定一个二叉树，返回它的中序遍历。

**进阶**: 递归算法很简单，你可以通过迭代算法完成吗？

#### 解题思路及实现

二叉树相关真的是非常有趣的一个算法知识点（因为这道题非常具有代表性，我觉得面试考到的概率最高2333......），后续笔者会针对该知识点进行更详细的探究，本文列出两个解决方案。

#### 1.递归法

```java
public class Solution {

  // 1.递归法
  public List<Integer> inorderTraversal(TreeNode root) {
      List<Integer> list = new ArrayList<>();
      dfs(root, list);
      return list;
  }

  private void dfs(TreeNode node, List<Integer> list) {
      if (node == null) return;

      // 中序遍历：左中右
      if (node.left != null)
          dfs(node.left, list);

      list.add(node.val);

      if (node.right != null)
          dfs(node.right, list);
  }
}
```

#### 2.使用栈

```java
public class Solution {

  // 2.使用栈
  public List<Integer> inorderTraversal(TreeNode root) {
        List<Integer> list = new ArrayList<>();
        Stack<TreeNode> stack = new Stack<>();

        TreeNode curr = root;
        while (!stack.isEmpty() || curr != null) {
            while (curr != null) {
                stack.push(curr);
                curr = curr.left;
            }
            curr = stack.pop();
            list.add(curr.val);
            curr = curr.right;
        }
        return list;
    }
}
```

## 参考 & 感谢

文章绝大部分内容节选自`LeetCode`，**栈和深度优先搜索** 概述：

* https://leetcode-cn.com/explore/learn/card/queue-stack/219/stack-and-dfs/

例题：

* https://leetcode-cn.com/problems/number-of-islands
* https://leetcode-cn.com/problems/clone-graph/
* https://leetcode-cn.com/problems/target-sum/
* https://leetcode-cn.com/problems/binary-tree-inorder-traversal/

## 关于我

Hello，我是 [却把清梅嗅](https://github.com/qingmei2) ，如果您觉得文章对您有价值，欢迎 ❤️，也欢迎关注我的 [博客](https://juejin.im/user/588555ff1b69e600591e8462/posts) 或者 [GitHub](https://github.com/qingmei2)。

如果您觉得文章还差了那么点东西，也请通过**关注**督促我写出更好的文章——万一哪天我进步了呢？

* [我的Android学习体系](https://github.com/qingmei2/blogs)
* [关于文章纠错](https://github.com/qingmei2/blogs/blob/master/error_collection.md)
* [关于知识付费](https://github.com/qingmei2/blogs/blob/master/appreciation.md)
* [关于《反思》系列](https://github.com/qingmei2/blogs/blob/master/src/%E5%8F%8D%E6%80%9D%E7%B3%BB%E5%88%97/%E5%8F%8D%E6%80%9D%7C%E7%B3%BB%E5%88%97%E7%9B%AE%E5%BD%95.md)
