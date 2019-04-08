---
title: Kotlin 中 Extensions 和 Utils 的区别
date : 2018.10.01
tag  : Kotlin
---

Extensions 看起来类似 JS 中的 prototype（但实际上是两种不同的概念）：你可以直接给一个类添加它原来所没有的方法，不论这个类是抽象类，还是不可继承的 final 的类。

现在存在一个名为 ContextExtensions.kt 的文件，它简单给 Context 扩展了 dp 转 px 的方法：

```Java
import android.content.Context
import android.support.annotation.FloatRange
import android.util.TypedValue

fun Context.dp2px(@FloatRange(from = 0.0) dp: Float): Int {
    return TypedValue.applyDimension(TypedValue.COMPLEX_UNIT_DIP, dp, resources.displayMetrics).toInt()
}

// 使用时，
// context.dp2px(...)
```

我们注意到，对于 dp2px 这个方法来说，本来应该是写在 Utils 里面的，但是现在却通过 Extensions 实现了，那么是不是所有 Utils 都可以用 Extensions 实现呢？

代码层面上确实可以做到，但是我们不建议这样做。而只是应该在适当的场景下，才使用 Extensions 实现 Utils 的功能。假设现在有另外一个依赖 Context 的功能，比如通过 Chrome Custion Tabs 打开一个网页，用 Utils 来写是这样的：

```Java
object CustomTabsUtils {
    fun openUrl(context: Context, url: String) {
        val builder = CustomTabsIntent.Builder()
        builder.build().launchUrl(context, Uri.parse(url))
    }
}
```

用 Extensions 来写是这样的：

```Java
fun Context.openUrl(url: String) {
    val builder = CustomTabsIntent.Builder()
    builder.build().launchUrl(this, Uri.parse(url)) // this 就是 context
}
```

看起来和 dp2px 差不多，都是依赖 Context 来做自己想做的事情，但是我们**按照场景**仔细想想：

dp2px 本质上是类似 getString 的 Context 自身已有的方法的分类；而 openUrl 则是凭空创造了一种 Context 的使用场景，不够纯粹；当一个 Extension 的代码不再关注这个类本身，而仅仅是使用 this 来作为另一种场景的依赖时，就不能作为 Extension 存在了，你应该把他转换为 Util 的代码。

**永远记住：Extensions 只是用来扩展类本身的属性，而不是作为另一种场景的依赖而存在。**