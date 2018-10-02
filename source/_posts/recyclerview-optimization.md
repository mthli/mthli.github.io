---
title: RecyclerView 性能优化
date : 2017.04.06
---

老文一篇，参考 Google 在 Medium 上发表的文章 [RecyclerView Prefetch](https://medium.com/google-developers/recyclerview-prefetch-c2f269075710). RV 是非常成熟的一个框架，GitHub 上有很多基于 RV 的库，大多数都属于 bootstrap 类型，也就是方便管理和使用 RV ；但要提到性能，和裸写并没有什么区别。相反在编码时注意一些事项更能优化性能。

RV 滑动展示数据的过程，简单可以概括为输入、动画、布局、渲染四个过程，如下图所示，在 Android 5.0 以上具体的渲染过程会交付给 RenderThread（包括阴影的绘制等都由这个线程绘制，而不是 UI Thread ，这也是 5.0 开始图形性能大幅提升的重要原因）：

![](/images/2017-04-06-normal.png)

我们都知道的事实是，滑动展示过程单帧少于 16ms 是 OK 的，超过 16ms 是卡顿的，下图形象地展示了繁重的输入过程会导致卡顿：

![](/images/2017-04-06-block.png)

输入过重的一个重要原因是，展示当前列表项时，同时也在创建下一个新的、没有被缓存的列表项：

![](/images/2017-04-06-create.png)

针对这个问题，官方提供的解决方案也很简洁，一次展示过程分别在 UI Thread 和 RenderThread 中完成，两者之间存在顺序关系；由于是不同的线程，线程等待的过程中自然可以利用空闲时间来先把下一次展示过程可以做的先做了，比如创建新的列表项：

![](/images/2017-04-06-solution.png)

开启这样的设置也很简单，只需要升级到 RecyclerView v25.1.0 以上即可，默认提供的 LayoutManager 均提供 prefetch 功能。如果是你自己实现的 LayoutManager 就要自己实现了，具体请参考[这篇博客](https://medium.com/google-developers/recyclerview-prefetch-c2f269075710)。

当然想要单纯依靠 prefetch 来保证 RV 滑动不掉帧是不可能的，我们仍然需要在自己的代码中将耗时的部分剔除。到这部分和 ListView 的优化就有很多重合的地方了，简化图层结构这些都没什么好说的，推荐大家开始使用 [ConstraintLayout](https://developer.android.com/training/constraint-layout/index.html) ；如果存在大量 findViewById（以当前视图为根深度优先遍历）操作，谨慎配合 [DataBinding](https://developer.android.com/topic/libraries/data-binding/index.html) 食用可显著提升获取目标视图的效率。

一个常见的问题是在 ViewHolder 中进行数据处理：

```Java
// 移除字符串中的 HTML 标签，但保留其中的文字
mTextView.setText(Html.fromHtml(data).toString());
```

Html.fromHtml() 这个方法是比较耗时的，对于不同的字符串，几毫秒到十几毫秒不等。就算一次几毫秒，几个 TV 就是十几毫秒，掉帧是十分严重的。所以我们真的有必要在 VH 中存在数据处理的逻辑吗？

RV 只是一个视图层级的库，严格意义上来说管理的只是做视图展示的工作；Adapter 和 VH 严格意义上来说做的**只是数据与视图的绑定逻辑，而不应该同时做数据的处理逻辑。**想象一些我们使用 RV 的一般过程，即从网络异步获取数据到 Adapter notify change ，数据的处理逻辑应该是在 notify change 之前就做好了的，这样在 VH 里就可以简单无压力地做数据与视图的绑定逻辑；同时数据的处理逻辑可以与网络异步线程放在一起，站在用户角度，最多就是网络刷新时间稍长一点。君不见 Twitter 数据结构与视图接口如此复杂，滚动列表时依然流畅，我想一定是做了这方面的处理吧。