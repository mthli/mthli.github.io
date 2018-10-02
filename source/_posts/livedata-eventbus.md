---
title: 使用 LiveData 实现 EventBus
date : 2017.05.25
---

Google IO 2017 推出了 [Android Architecture Components](https://developer.android.com/topic/libraries/architecture/index.html) ，着重处理 Android 开发中的生命周期问题。其中一个有意思的组件叫做 [LiveData](https://developer.android.com/topic/libraries/architecture/livedata.html) ，它在把数据的更新行为变为可观测的对象的同时，与 Activity/Fragment 的生命周期绑定，使得我们不再需要关心与生命周期相关的回调问题。

如果你经常使用 RxBus 配合 [RxLifecycle](https://github.com/trello/RxLifecycle) ，你一定知道我在说什么。有了 LiveData ，我们可以用更「原生」的方式实现 EventBus ，我们可以把这样的实现称为 LiveBus 。下面来看看 LiveBus 的实现吧：

```Java
import android.arch.lifecycle.MutableLiveData;

public final class LiveBus extends MutableLiveData<Object> {
    private static final class LiveBusHolder {
        private static final LiveBus INSTANCE = new LiveBus();
    }

    private LiveBus() {}

    @NonNull
    public static LiveBus getInstance() {
        return LiveBusHolder.INSTANCE;
    }
}
```

然后你就可以在任何地方开始使用 LiveBus 啦：

```Java
// 在**非主线程**中使用 postValue()
LiveBus.getInstance().postValue(event);

// 在**主线程**中使用 setValue()
LiveBus.getInstance().setValue(event);

// 在 LifecycleActivity/LifecycleFragment 中订阅事件，订阅事件的回调都将在**主线程**中执行；
// observe() 仅仅在 onStart() 与 onPause() 之间时执行回调；
// observeForever() 在任何生命周期都可以回调，所以**务必**手动移除回调
LiveBus.getInstance().observe(this, new Observer<Object>() {
    @Override
    public void onChanged(@Nullable Object event) {
        // Event 通常都是继承自 Object 的，
        // 要想订阅某个指定的 Event 类型，你还需要自己 cast 一下
        // ...
    }
});
```

现阶段必须配合 LifecycleActivity/LifecycleFragment 才能使用 LiveBus 。Google 官方表示，在相关实现稳定后，就会合并到 support 库的实现中，届时我们就可以很方便地使用 LiveBus 啦。