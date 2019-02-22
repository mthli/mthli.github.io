---
title: 复习一下快排
date : 2019.02.22
---

快排基于这样一种思想：在需要排序的数组中，选取其中一个数锚点，一次迭代过后，使参考值左边的数总是小于等于它，使参考值右边的数总是大于等于它；经过多次迭代后，最终整个数组会变得有序。

举个最简单的例子，假设存在随机数组 [5, 4, 2], 选取 4 作为锚点，则根据上述原则，一次迭代后数组变为 [2, 4, 5]. 即使数组非常大，在不断迭代过程中也会被切分为一个一个这样的小数组进行排序，最终它们合起来的结果就是一个有序的大数组（可能你想到了归并排序）。

举个复杂点的例子，假设存在下列随机数组：

5 4 2 3 7 6 1 8 9

为方便起见，在一次迭代过程中，我们一般选取数组的第一个元素 5 作为锚点：

**5** 4 2 3 7 6 1 8 9

我们从右往左 (⇃) 遍历数组，找到第一个小于 5 的元素；在 5 之后从左往右 (⇂) 遍历数组，找到第一个大于 5 的元素；然后交换它俩的位置：

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;⇂&nbsp;&nbsp;&nbsp;⇃
**5** 4 2 3 7 6 1 8 9 ⇒ **5** 4 2 3 1 6 7 8 9

**继续**从右往左 (⇃) 找下一个小于 5 的元素；从左往右 (⇂) 找下一个大于 5 的元素。注意，此时在遍历过程中，从右往左在找到目标之前，会先遇到从左往右的指针，此时就意味着本次遍历结束了，应当交换锚点 5 和这个相遇位置上的元素：

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;↓
**5** 4 2 3 1 6 7 8 9 ⇒ 1 4 2 3 **5** 6 7 8 9

接着以锚点为分解，对锚点之前的数组和锚点之后的数组分别执行上述逻辑。在我们的例子中，锚点 5 后面的数组已经有序，就不展开讨论了；前面 [1, 4, 2, 3] 在后序迭代过程中，会逐步变为：

1 [4, 2, 3] → 1 [3, 2] 4 → 1 2 3 4

最后附上快排代码：

```kotlin
object QuickSort {
    fun sort(array: IntArray, low: Int, high: Int) {
        if (low >= high) {
            return
        }

        val part = part(array, low, high)
        sort(array, low, part - 1)
        sort(array, part + 1, high)
    }

    private fun part(array: IntArray, low: Int, high: Int): Int {
        val v = array[low]
        var i = low
        var j = high

        while (true) {
            // 从右往左找到小于锚点的数
            while (array[j] >= v) {
                if (j - 1 < low) {
                    break
                } else {
                    j--
                }
            }

            // 从左往右找到大于锚点的数
            while (array[i] <= v) {
                if (i + 1 > high) {
                    break
                } else {
                    i++
                }
            }

            if (i >= j) {
                break
            }

            // 交换发现的两个数
            val temp = array[i]
            array[i] = array[j]
            array[j] = temp
        }

        // 交换锚点与指针相遇的位置上的数
        val temp = array[low]
        array[low] = array[j]
        array[j] = temp

        // 返回指针相遇的位置
        return j
    }
}
```