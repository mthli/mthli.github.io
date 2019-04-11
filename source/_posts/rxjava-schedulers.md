---
title : RxJava 线程切换原理
date  : 2019-04-10
tag   : RxJava
---

从开始接触 [RxJava](https://github.com/ReactiveX/RxJava) 到现在也将近三年了，的确是平时开发中不可或缺的依赖。但最近深感自己对其实现原理缺乏了解，因此有必要学习一番。从 RxJava 最常被使用到的线程切换场景开始，简要分析 newThread(), single(), io() 这三个我最经常使用到的调度器 (Scheduler) 是如何实现的，最后再简单介绍一下 AndroidSchedulers.mainThread() 是如何实现的。

## Scheduler

[Scheduler](http://reactivex.io/RxJava/javadoc/io/reactivex/Scheduler.html) 是所有调度器实现的抽象父类，子类可以通过复写其 scheduleDirect() 来自行决定如何调度被分配到的任务；其实还有个 schedulePeriodicallyDirect() 方法，但大同小异；同时通过复写其 createWorker() 返回的 Scheduler.Worker 实例来执行具体的某个任务。此处的任务指的是通过 Runnable 封装的可执行代码块。

对于 Scheduler.scheduleDirect() 而言，默认的实现代码如下：

```Java
public abstract class Scheduler {
    ...

    @NonNull
    public Disposable scheduleDirect(
            @NonNull Runnable run, long delay, @NonNull TimeUnit unit) {

        // 每次调度都新建一个 Worker
        final Worker w = createWorker();

        final Runnable decoratedRun = RxJavaPlugins.onSchedule(run);
        DisposeTask task = new DisposeTask(decoratedRun, w);

        // 将任务提交给 Worker 执行
        w.schedule(task, delay, unit);

        return task;
    }
}
```

[Scheduler.Worker](http://reactivex.io/RxJava/javadoc/io/reactivex/Scheduler.Worker.html) 本质上与 JDK 原生提供的 [Executor](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/Executor.html) 做的是类似的事，子类可以通过复写其 schedule() 方法决定如何执行对应的任务。在下文中我们可以看到，实际上 RxJava 的线程切换也正是基于 Executor 等 JDK 提供的多线程 API 实现的。

## newThread()

[newThread()](http://reactivex.io/RxJava/2.x/javadoc/io/reactivex/schedulers/Schedulers.html#newThread--) 是上述几个调度器中最简单的实现，对于每一个分配到该调度器上的任务，都会新开启一个线程来执行，因此频繁使用可能会导致 OOM. 对应的调度器实现为 NewThreadScheduler, 它**并没有**复写 scheduleDirect() 的默认实现, 而仅仅是复写了 createWorker() 返回 NewThreadWorker:

```Java
public final class NewThreadScheduler extends Scheduler {
    final ThreadFactory threadFactory;

    ...

    public NewThreadScheduler(ThreadFactory threadFactory) {
        this.threadFactory = threadFactory;
    }

    @NonNull
    @Override
    public Worker createWorker() {
        return new NewThreadWorker(threadFactory);
    }
}
```

我们可以通过 NewThreadWorker 来简单了解一下一个 Scheduler.Worker 是如何实现的：

```Java
public class NewThreadWorker extends Scheduler.Worker implements Disposable {
    private final ScheduledExecutorService executor;

    ...

    public NewThreadWorker(ThreadFactory threadFactory) {
        // 每次都新建一个
        executor = SchedulerPoolFactory.create(threadFactory);
    }

    @NonNull
    @Override
    public Disposable schedule(
            @NonNull final Runnable action, long delayTime, @NonNull TimeUnit unit) {
        ...
        return scheduleActual(action, delayTime, unit, null);
    }

    @NonNull
    public ScheduledRunnable scheduleActual(
            final Runnable run, long delayTime,
            @NonNull TimeUnit unit, @Nullable DisposableContainer parent) {
        Runnable decoratedRun = RxJavaPlugins.onSchedule(run);
        ScheduledRunnable sr = new ScheduledRunnable(decoratedRun, parent);

        ...

        Future<?> f;
        try {
            if (delayTime <= 0) {
                // 直接提交
                f = executor.submit((Callable<Object>) sr);
            } else {
                // 延迟调度
                f = executor.schedule((Callable<Object>) sr, delayTime, unit);
            }

            ...
        } catch (RejectedExecutionException ex) {
            ...
        }

        return sr;
    }
}
```

上述代码中，executor 实际上是 corePoolSize 为 1 的 [ScheduledThreadPoolExecutor](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ScheduledThreadPoolExecutor.html) 实例。通过阅读 JDK 文档我们知道，该 executor 至多只会有 1 个线程在执行：

> While this class inherits from ThreadPoolExecutor, a few of the inherited tuning methods are not useful for it. In particular, because it acts as a fixed-sized pool using corePoolSize threads and an unbounded queue, adjustments to maximumPoolSize have no useful effect.

综合上文我们知道，由于 newThread() 没有复写 scheduleDirect(), 所以每次都会新建一个 NewThreadWorker, 继而每次都会创建一个容量为 1 的线程池，在这个线程池中执行新任务。所以的确是「每一个分配到该调度器上的任务，都会新开启一个线程来执行」啊。

## single()

了解了 newThread() 是如何实现线程切换之后，再来看 [single()](http://reactivex.io/RxJava/2.x/javadoc/io/reactivex/schedulers/Schedulers.html#single--) 对应的实现就很简单了：

```Java
public final class SingleScheduler extends Scheduler {
    final AtomicReference<ScheduledExecutorService> executor
        = new AtomicReference<ScheduledExecutorService>();

    ...

    @NonNull
    @Override
    public Worker createWorker() {
        return new ScheduledWorker(executor.get());
    }

    @NonNull
    @Override
    public Disposable scheduleDirect(
            @NonNull Runnable run, long delay, TimeUnit unit) {
        ScheduledDirectTask task
            = new ScheduledDirectTask(RxJavaPlugins.onSchedule(run));
        try {
            Future<?> f;
            if (delay <= 0L) {
                // 直接提交
                f = executor.get().submit(task);
            } else {
                // 延时调度
                f = executor.get().schedule(task, delay, unit);
            }

            task.setFuture(f);
            return task;
        } catch (RejectedExecutionException ex) {
            RxJavaPlugins.onError(ex);
            return EmptyDisposable.INSTANCE;
        }
    }
}
```

同样地，executor 是 corePoolSize 为 1 的 ScheduledThreadPoolExecutor 实例（的引用），只是 scheduleDirect() 被复写为直接把新任务调度到该 executor 上，而不是每次都新建一个，因此满足 single() 定义的单线程调度。我们通常可以在这个调度器上执行**强顺序关系**的任务。

## io()

[io()](http://reactivex.io/RxJava/2.x/javadoc/io/reactivex/schedulers/Schedulers.html#io--) 不像 newThread() 那样每次都会新建一个线程，相反地，如果有空闲线程，新任务就会被调度到这个空闲线程上；只有线程不够时，才会扩容。通常大多数异步操作，如网络请求，都可以切换到这个调度器上。

然而你会发现，io() 和 newThread() 一样，并没有复写 scheduleDirect(), 每次调度时，io() 都会新建一个 EventLoopWorker 实例：

```Java
public final class IoScheduler extends Scheduler {
    final AtomicReference<CachedWorkerPool> pool;

    ...

    @NonNull
    @Override
    public Worker createWorker() {
        return new EventLoopWorker(pool.get());
    }
}
```

对应的 EventLoopWorker 实现如下：

```Java
static final class EventLoopWorker extends Scheduler.Worker {
    private final ThreadWorker threadWorker;

    ...

    EventLoopWorker(CachedWorkerPool pool) {
        ...

        // 这一句是关键
        this.threadWorker = pool.get();
    }

    @NonNull
    @Override
    public Disposable schedule(
            @NonNull Runnable action, long delayTime, @NonNull TimeUnit unit) {
        ...

        // ThreadWorker 是 NewThreadWorker 的子类，
        // 其 scheduleActual() 就是 NewThreadWorker 的 scheduleActual()
        return threadWorker.scheduleActual(action, delayTime, unit, tasks);
    }
}
```

然后你又会发现，EventLoopWorker 的大体逻辑和 NewThreadWorker 差不多，所以它是怎么实现复用和扩容的呢？其实奥义就在 CachedWorkerPool 这个类：

```Java
static final class CachedWorkerPool implements Runnable {
    private final ConcurrentLinkedQueue<ThreadWorker> expiringWorkerQueue;
    final CompositeDisposable allWorkers;

    ...

    // 返回一个可以被复用的 Worker, 或一个新的 Worker
    ThreadWorker get() {
        ...

        // 如果有空闲 Worker 则使用之
        while (!expiringWorkerQueue.isEmpty()) {
            ThreadWorker threadWorker = expiringWorkerQueue.poll();
            if (threadWorker != null) {
                return threadWorker;
            }
        }

        // No cached worker found, so create a new one.
        ThreadWorker w = new ThreadWorker(threadFactory);
        allWorkers.add(w);
        return w;
    }
}
```

我们可以向上回顾一下 EventLoopWorker 的构造方法，每次 pool.get() 都会返回一个可以被复用的 Worker, 或一个新的 Worker, 从而实现了复用和扩容的逻辑。

## AndroidSchedulers.mainThread()

AndroidSchedulers.mainThread() 是 [RxAndroid](https://github.com/ReactiveX/RxAndroid) 提供的用于切换到 Android 主线程（通常是 UI 线程） 的调度器，我们可以在后台线程中执行一些耗时操作，再切换到主线程更新 UI.

RxAndroid 的封装非常简单，就是基于 [Handler](https://developer.android.com/reference/android/os/Handler) 的实现：

```Java
final class HandlerScheduler extends Scheduler {
    // handler 是与 Looper.getMainLooper() 绑定的 Handler 实例
    private final Handler handler;
    private final boolean async;

    ...

    @Override
    public Disposable scheduleDirect(Runnable run, long delay, TimeUnit unit) {
        ...

        run = RxJavaPlugins.onSchedule(run);
        ScheduledRunnable scheduled = new ScheduledRunnable(handler, run);

        // 延时调度
        handler.postDelayed(scheduled, unit.toMillis(delay));

        return scheduled;
    }

    @Override
    public Worker createWorker() {
        return new HandlerWorker(handler, async);
    }
}
```

同样地，我们再来看看 HandlerWorker, 也是基于 Handler 实现的：

```Java
private static final class HandlerWorker extends Worker {
    private final Handler handler;
    private final boolean async;

    ...

    // Async will only be true when the API is available to call.
    @Override
    @SuppressLint("NewApi")
    public Disposable schedule(Runnable run, long delay, TimeUnit unit) {
        ...

        ScheduledRunnable scheduled = new ScheduledRunnable(handler, run);
        Message message = Message.obtain(handler, scheduled);
        // Used as token for batch disposal of this worker's runnables.
        message.obj = this;

        if (async) {
            message.setAsynchronous(true);
        }

        // 延迟调度
        handler.sendMessageDelayed(message, unit.toMillis(delay));

        ...
    }
}
```

注意，上述代码中有一个 async 变量，设置为 true 时可以提升 UI 性能（如果你需要频繁调度到 Android 主线程的话），具体可以参见这篇文章 [RxAndroid’s New Async API](https://medium.com/@ZacSweers/rxandroids-new-async-api-4ab5b3ad3e93).