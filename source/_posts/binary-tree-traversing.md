---
title: 二叉树的各种遍历
data : 2019.02.21
---

最近在准备面试，不得不说业务写多了，基础方面还是疏漏了很多的。因此希望通过写博客的方式把自己复习的内容记录下来。今天就来复习一下二叉树的各种遍历。

对于一个二叉树而言，通常有两类遍历方式，一类是前序、中序、后序遍历，另一类则是深度遍历和广度遍历。对于前者而言，我们主要使用**递归**的思想；对于后者则是**栈与队列**。

如果用 D, L, R 分别代码访问根节点、访问左子树、访问右子树的话，前序遍历即 DLR, 中序遍历即 LDR, 后序遍历即 LRD. 你也发现了，所谓的「序」就是访问根节点的顺序。下面是一个二叉树的实例：

![](/images/2019-02-21-binary-tree.png)

DLR: 2 → 7 → 2 → 6 → 5 → 11 → 5 → 9 → 4

LDR: 2 → 7 → 5 → 6 → 11 → 2 → 5 → 4 → 9

LRD: 2 → 5 → 11 → 6 → 7 → 4 → 9 → 5 → 2

注意了，无论是前序、中序还是后序，在遍历的时候一定要走到**叶子节点**后才开始输出，之后都是不断回溯的过程。因此它们都是深度遍历的一种情况，而递归的本质正是不断压栈出栈的过程。在这里选取前序遍历作为例子，来看看递归的实现和迭代的实现：

```kotlin
import java.util.*

object BinaryTree {

    data class Node(val value: Int, val left: Node?, val right: Node?)

    // 前序遍历的递归实现
    fun recursiveDLR(root: Node?) {
         root?.let {
             println(it.value)
             recursiveDLR(root.left)
             recursiveDLR(root.right)
         }
    }

    // 前序遍历的迭代实现
    fun iterateDLR(root: Node) {
        val stack = Stack<Node>()
        var node: Node? = root

        while (node != null || stack.isNotEmpty()) {
            node = node?.let {
                println(it.value)
                stack.push(it)
                it.left
            } ?: stack.pop().right
        }
    }
}
```

再来看看广度遍历。所谓的广度遍历，你也可以理解为层次遍历，先遍历第一层，再遍历第二层... 直到遍历完所有层为止。因此上述二叉树实例的输出结果为：2 → 7 → 5 → 2 → 6 → 9 → 5 → 11 → 4

对应地广度搜索的队列实现如下：

```kotlin
fun bfs(root: Node) {
    val queue = LinkedList<Node>()
    queue.offer(root)

    while (queue.isNotEmpty()) {
        val node = queue.poll()
        println(node.value)
        node.left?.let { queue.offer(it) }
        node.right?.let { queue.offer(it) }
    }
}
```