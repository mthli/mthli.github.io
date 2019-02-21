---
title: LruCache
date : 2019.02.21
---

LruCache 是实现缓存的常见方案。当然啦，官方有 [LruCache](https://developer.android.com/reference/android/util/LruCache) 的工具类；Jake Wharton 已经帮我们实现了一套简单易用的磁盘缓存策略 [DiskLruCache](https://github.com/JakeWharton/DiskLruCache). 鉴于其是面试中的常见问题，我们还是有必要学习一个。

我们一般使用 Map 来作为缓存的数据结构，其时间复杂度是 O(1). 但是缓存的空间是有限的，不可能无限制地扩展 Map, 因此需要有策略地从 Map 中丢弃掉一些命中率低的缓存数据。LRU(Least Recently Used) 即是这样一种策略，它假定一定时间内没有被使用的数据，在未来被使用的概率也不大，因此可以直接丢弃。

最简单的用来统计是否被使用的方法，自然是在每个节点上记录命中次数；当缓存达到最大值，遍历找到命中次数最少的那个节点，将其删除之。但是遍历的时间复杂度是 O(N), 且 N 可能比较大。每次删除节点都要遍历一次，性能上可能是不可接受的。

成熟的 LruCache 在 Map 的基础上，配合使用了**双向链表**来连接 Map 中的各个节点。当有节点被命中时，将这个节点移动到链表尾部；当缓存达到最大值，将链表头部的节点删除之，同时也将其从 Map 中删除。如此循环往复，被命中的节点总是倾向于远离链表头部而免于被删除，而没有被命中的节点则越来越靠近链表头部最终被删除。之所以使用双向链表，是因为对某个节点进行插入删除操作时，不需要像单链表那样需要通过遍历知道其前序节点和后序节点，时间复杂度 O(1). 假设一个已经达到缓存最大值的结构如下：

A ⇆ B ⇆ C ⇆ D ⇆ E ⇆ F ⇆ G

此时命中了 C, 则将 C 移动到链表尾部：

A ⇆ B ⇆ D ⇆ E ⇆ F ⇆ G ⇆ C

如果有一个新的数据 H 入队，由于已经达到缓存最大值，则 A 被出队：

B ⇆ D ⇆ E ⇆ F ⇆ G ⇆ C ⇆ H

因此，不论是读缓存，还是写缓存，时间复杂度都是 O(1) 了。

其实我们可以很简单地通过复写 [LinkedHashMap](https://docs.oracle.com/javase/8/docs/api/java/util/LinkedHashMap.html) 实现一个最简单的 LruCache:

```kotlin
class LruCache<K, V>(initialCapacity: Int) : LinkedHashMap<K, V>(initialCapacity) {
    // 可以是你自定义的大小
    override fun removeEldestEntry(e: MutableMap.MutableEntry<K, V>?) = size > 7
}
```

不过为了明确思想，我们还是自己实现一套逻辑比较好：

```kotlin
data class Node(
        val key: String, var value: String?,
        var before: Node?, var after: Node?)

class LruCache(private val initialCapacity: Int) {
    private val map = HashMap<String, Node>(initialCapacity)

    private var head: Node? = null
    private var tail: Node? = null
    private var size: Int = 0

    fun get(key: String): String? {
        val node = map[key] ?: return null

        // 将 node 从链表原位置删除
        node.before?.apply { after = node.after }
        node.after?.apply { before = node.before }

        // 将 node 衔接到尾节点上
        node.before = tail
        tail = node

        return node.value
    }

    fun put(key: String, value: String?) {
        var node = map[key]

        if (node == null) {
            node = Node(key, value, null, null)
        } else {
            node.value = value
        }

        // 记录头节点
        if (head == null) {
            head = node
        }

        // 记录尾节点
        if (tail == null) {
            tail = node
        } else {
            node.before?.apply { after = node.after }
            node.after?.apply { before = node.before }
            node.before = tail
            tail = node
        }

        // 如果超出限制，则删除当前头节点
        if (size.inc() > initialCapacity) {
            head = head?.apply { map.remove(this.key) }?.after
            size--
        }

        map[key] = node
        size++
    }
}
```