## 0、前言
如果不是很了解 `LiveData` 和 `Lifecycle` 的同学可以先看一下我之前的文章
[基于 Android Architecture Components 的 MVVM 浅析](https://juejin.im/post/5cba9b2ef265da038145d51a)。同时安利下自己平时用的工具库
[LiveDataUtils](https://github.com/funnywolfdadada/LiveDataUtils)，还在不断完善中，欢迎大家 star、fork、关注和批评指正。
## 1、为什么要把 `LiveData` 当作事件传递用
利用观察者模式打造的事件总线的优点不必多说（当然也有很多缺点），如 `EventBus` 和 `RxBus` 用的好的话能起到很好的解耦作用，使整个程序架构更加清晰，不至于到处传递各种 `Callback`。但是他们都缺少了对 `View` 层（`Activity`、`Fragment` 等）的生命周期的感知能力，需要在生命周期结束时手动解除观察者，手动管理生命周期十分繁琐且很容易出错。
而 Google 推出的 `Lifecycle` 库就是为了解决这一问题，其中的 `LiveData` 就是一个具有生命周期感知能力的观察者，如果用它来打造一个**能够感知生命周期的事件总线**，岂不美哉！
![](https://user-gold-cdn.xitu.io/2019/5/18/16acb0973f58104d?w=255&h=255&f=jpeg&s=10003)
## 2、`LiveData` 当作事件传递用的那些坑
在随着对 `LiveData` 的运用和理解的逐渐深入，特别是对它的「生命周期感知能力」有了更深的理解，慢慢发现这样用的一些坑，借此机会就跟大家分享探讨一下。而且我平时也有把 `LiveData` 纯粹当作事件传递来用，特别是列表操作（比如涉及 IO 的增删操作，`View` 层需要知道哪个数据改动了，以及操作是否成功等，只能以事件的形式传递）。
### 2.1、`postValue` 数据丢失的问题
在我的前一篇文章中也提到过，大家也可以直接看源码。`postValue` 只是把传进来的数据先存到 `mPendingData`，然后往主线程抛一个 `Runnable`，在这个 `Runnable` 里面再调用 `setValue` 来把存起来的值真正设置上去，并回调观察者们。而如果在这个 `Runnable` 执行前多次 `postValue`，其实只是改变暂存的值 `mPendingData`，并不会再次抛另一个 `Runnable`。这就会出现后设置的值把前面的值覆盖掉的问题，会导致事件丢失。
``` java
protected void postValue(T value) {
    boolean postTask;
    synchronized (mDataLock) {
        postTask = mPendingData == NOT_SET;
        // 这里先把数据暂存起来，后来的数据会覆盖前面的
        mPendingData = value;
    }
    // 这里保证只抛一个 mPostValueRunnable，#-.-
    if (!postTask) {
        return;
    }
    ArchTaskExecutor.getInstance().postToMainThread(mPostValueRunnable);
}
```
### 2.2、`setValue` 不回调观察者
`LiveData` 的生命周期感知能力就体现在这里，它不会回调处于「非激活状态」（即 `onStart` 之后到 `onPause` 之前）的观察者，因为这时更新 `View` 没有意义，而且比较危险，它会等到观察者激活之后再把新的值回调给他。
但是如果我传了多个数据（假设都是用 `setValue` 保证不会被覆盖），那些处于非激活状态的观察者是毫不知情的，他们在激活的时候只会收到最后一个数据。这对于事件传递来说，就表现为事件丢失，中间传的任何数据都无法收到，那也就失去了事件传递的意义。
``` java
// LiveData 通知观察者时调的函数
private void considerNotify(ObserverWrapper observer) {
    // 非激活就直接返回了
    if (!observer.mActive) {
        return;
    }
    // Check latest state b4 dispatch. Maybe it changed state but we didn't get the event yet.
    //
    // we still first check observer.active to keep it as the entrance for events. So even if
    // the observer moved to an active state, if we've not received that event, we better not
    // notify for a more predictable notification order.
    if (!observer.shouldBeActive()) {
        observer.activeStateChanged(false);
        return;
    }
    // 每次更新数据都会版本加一，这里保证不会回调之前发送过的数据
    if (observer.mLastVersion >= mVersion) {
        return;
    }
    observer.mLastVersion = mVersion;
    //noinspection unchecked
    observer.mObserver.onChanged((T) mData);
}
```
### 2.3、`LiveData` 就不是为传递事件准备的
从上面两点也可以看出，`LiveData` （或者说它的观察者） 在观察者激活之前并不关心中间经历过多少次数据变更，它只会在某个观察者激活时，传递给他最新的值，中间的值都不会起作用。
当然 `LiveData` 的设计也不是为了传递事件的，它是为了反应 `View` 当前状态的，`View` 层只需要根据当前 `LiveData` 的值去渲染数据就行，非激活状态时 `View` 都不可见，就算更新了也没意义。
我最开始也是觉得 `LiveData` 用到了观察者模式，而且可以进行一些不同 `Fragment` 之间数据通讯，就想到了事件总线，现在想想当时还是 too young too naive。
![](https://user-gold-cdn.xitu.io/2019/5/18/16acb1356114935b?w=289&h=225&f=jpeg&s=8403)
当然，也不是说 `LiveData` 也不是没救了，有谷歌给我们提供了强大的 `Lifecycle` 库，我们完全可以自己造一个不会丢事件的 `LiveData`。
## 3、打造一个不会丢事件的 `LiveData`
`LiveData` 的其他功能做的很完善，只是会丢事件，我们要改造就要就针对上面的问题逐个击破。
## 3.1、`postValue` 的问题
对于 `postValue` 的问题，既然它最后也是调用的 `setValue`，丢数据是因为只抛了一次 `Runable`，那我们就自己每次都往主线程抛一个 `Runable` 就能解决这个问题
具体实现可以参考之前提到的 [LiveDataUtils](https://github.com/funnywolfdadada/LiveDataUtils)。
``` java
/**
 * LiveData 相关的工具类，简化 LiveData 操作
 *
 * @author funnywolf
 * @since 2019-04-22
 */
public class LiveDataUtils {
    private static Handler sMainHandler;
    /**
     * 用 setValue 更新 MutableLiveData 的数据，如果在子线程，就切换到主线程
     */
    public static <T> void setValue(MutableLiveData<T> mld, T d) {
        if (mld == null) {
            return;
        }
        if (Thread.currentThread() == Looper.getMainLooper().getThread()) {
            mld.setValue(d);
        } else {
            postSetValue(mld, d);
        }
    }

    /**
     * 向主线程的 handler 抛 SetValueRunnable
     */
    public static <T> void postSetValue(MutableLiveData<T> mld, T d) {
        if (sMainHandler == null) {
            sMainHandler = new Handler(Looper.getMainLooper());
        }
        sMainHandler.post(SetValueRunnable.create(mld, d));
    }

    private static class SetValueRunnable<T> implements Runnable {
        private final MutableLiveData<T> liveData;
        private final T data;

        private SetValueRunnable(@NonNull MutableLiveData<T> liveData, T data) {
            this.liveData = liveData;
            this.data = data;
        }

        @Override
        public void run() {
            liveData.setValue(data);
        }

        public static <T> SetValueRunnable<T> create(@NonNull MutableLiveData<T> liveData, T data) {
            return new SetValueRunnable<>(liveData, data);
        }
    }
}
```
## 3.2、非激活状态的问题
其实我觉得这个问题的主要「责任」并不在 `LiveData`，而是在它的观察者，「是你告诉我你非激活的呀，那我怎么给你发数据呢，我发给你，万一你出问题了呢，那到底谁负责？」。
我们常用的观察者其实是 `LifecycleBoundObserver`，在调用 `public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<? super T> observer)` 会自动帮我们封装一个这样的观察者，而它会根据 `LifecycleOwner` 的生命周期呈现出「激活」和「非激活」状态。
``` java
// LifecycleBoundObserver
boolean shouldBeActive() {
    return mOwner.getLifecycle().getCurrentState().isAtLeast(STARTED);
}
```
`LiveData` 默认的还有另外一种观察者 `AlwaysActiveObserver`，它是我们在调用 `public void observeForever(@NonNull Observer<? super T> observer)` 时生成的，顾名思义它会一直处于激活状态，`LiveData` 当然也不会替我们管理这样观察者的生命周期，我们需要在不使用时手动调用 `public void removeObserver(@NonNull final Observer<? super T> observer)` 移除观察者，否则可能会内存泄漏。
``` java
// AlwaysActiveObserver
boolean shouldBeActive() {
    return true;
}
```
这个 `AlwaysActiveObserver` 看样子能够解决我们的问题，他一直处于激活状态，那所有的事件都会回调给他，但是需要我们自己管理生命周期。这不是开历史倒车吗？好不容易有生命周期感知了，现在又要自己手动搞？
### 3.3、造一个生命周期感知的还不丢事件的观察者
手动管理生命周期是绝对不能忍的，`AlwaysActiveObserver` 可以解决刚才说的问题，那我们就造一个新的观察者来管理 `observeForever` 和 `removeObserver` 的问题。既然要造，那就造个好用的，首先事件一定不能丢，要不就没意义了；而且生命周期要观察者自己管理，不能只是简单的 `observeForever` 和 `removeObserver`，非激活状态之类的也要考虑进去。
既然要管理生命周期，那就不得不用到 `LifecycleOwner`、`Lifecycle`，然后自己观察 `LifecycleOwner` 的 `Lifecycle`。
``` java
/**
 * Marks a class as a LifecycleObserver. It does not have any methods, instead, relies on
 * {@link OnLifecycleEvent} annotated methods.
 * <p>
 * @see Lifecycle Lifecycle - for samples and usage patterns.
 */
@SuppressWarnings("WeakerAccess")
public interface LifecycleObserver {

}
```
`Lifecycle` 对外只给了这个接口，并不含有任何回调，我们需要用注释里说的 `OnLifecycleEvent` 注解来标记相应的函数，`Lifecycle` 会通过反射拿到标记的函数，然后生成对应的适配器，感兴趣的可以看下 `Lifecycling.getCallback` 函数。比如我们可以这样用
``` java
public class Observer implements LifecycleObserver {
    @OnLifecycleEvent(Lifecycle.Event.ON_START)
    private void onStart() {
        doSomethingOnStart();
    }
}
```
拿到生命周期后，我们就可以在一开始 `observeForever`，在 `Lifecycle.Event.ON_DESTROY` 时 `removeObserver`。
接下来就要考虑激活和非激活的状态了，既然用了 `observeForever`，那每次事件都会有回调，这时候如果 `Lifecycle` 处于激活状态，那可以直接把事件发出去。但如果非激活，不能直接把事件发出去，又不能丢，那我们就需要先把事件存起来，然后在 `Lifecycle` 变为激活状态时再把存起来的事件发送出去。简单画了下流程图。
![](https://user-gold-cdn.xitu.io/2019/5/18/16acb5ae094ce6d7?w=680&h=535&f=png&s=56364)
``` java

/**
 * LiveData 用作事件总线时的观察者
 * 保证所有事件不丢失，保存非激活状态的事件，并能够在激活状态回调，且没有内存泄漏
 *
 * @see AsEventBus
 *
 * @author funnywolf
 * @since 2019-05-18
 */
public class LiveEventObserver<T> implements LifecycleObserver, Observer<T> {
    private LiveData<T> mLiveData;
    private LifecycleOwner mOwner;
    private Observer<? super T> mObserver;

    private List<T> mPendingData = new ArrayList<>();

    public LiveEventObserver(LiveData<T> liveData, LifecycleOwner owner, Observer<? super T> observer) {
        mLiveData = liveData;
        mOwner = owner;
        mObserver = observer;
        mOwner.getLifecycle().addObserver(this);
        mLiveData.observeForever(this);
    }

    /**
     * 在生命周期结束前的任何时候都可能会调用
     */
    @Override
    public void onChanged(@Nullable T t) {
        if (isActive()) {
            // 如果是激活状态，就直接更新
            mObserver.onChanged(t);
        } else {
            // 非激活状态先把数据存起来
            mPendingData.add(t);
        }
    }

    /**
     * @return 是否是激活状态，即 onStart 之后到 onPause 之前
     */
    private boolean isActive() {
        return mOwner.getLifecycle().getCurrentState()
                .isAtLeast(Lifecycle.State.STARTED);
    }

    /**
     * onStart 之后就是激活状态了，如果之前存的有数据，就发送出去
     */
    @OnLifecycleEvent(Lifecycle.Event.ON_START)
    private void onStart() {
        for (int i = 0; i < mPendingData.size(); i++) {
            mObserver.onChanged(mPendingData.get(i));
        }
        mPendingData.clear();
    }

    /**
     * onDestroy 时解除各方的观察和绑定，并清空数据
     */
    @OnLifecycleEvent(Lifecycle.Event.ON_DESTROY)
    private void onDestroy() {
        mLiveData.removeObserver(this);
        mLiveData = null;

        mOwner.getLifecycle().removeObserver(this);
        mOwner = null;

        mPendingData.clear();
        mPendingData = null;

        mObserver = null;
    }


    public static <T> void bind(@NonNull LiveData<T> liveData,
                                @NonNull LifecycleOwner owner,
                                @NonNull Observer<? super T> observer) {
        if (owner.getLifecycle().getCurrentState() == Lifecycle.State.DESTROYED) {
            return;
        }
        new LiveEventObserver<>(liveData, owner, observer);
    }
}
```
### 3.4、保证 `LiveData` 的事件更新
3.1 也说过要自己处理 `postValue`，其次要保证用我们自己定义的观察者，需要重写 `public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<? super T> observer)`。
``` java
/**
 * 用作事件总线的 {@link MutableLiveData}
 *
 * @see AsEventBus
 *
 * @author funnywolf
 * @since 2019-05-18
 */
public class EventMutableLiveData<T> extends MutableLiveData<T> {
    @Override
    public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<? super T> observer) {
        LiveEventObserver.bind(this, owner, observer);
    }

    @Override
    public void postValue(T value) {
        LiveDataUtils.setValue(this, value);
    }
}

/**
 * 用作事件总线的 {@link MediatorLiveData}
 *
 * @see AsEventBus
 *
 * @author funnywolf
 * @since 2019-05-18
 */
public class EventMediatorLiveData<T> extends MediatorLiveData<T> {
    @Override
    public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<? super T> observer) {
        LiveEventObserver.bind(this, owner, observer);
    }

    @Override
    public void postValue(T value) {
        LiveDataUtils.setValue(this, value);
    }
}

/**
 * 该注解只用于 {@link androidx.lifecycle.LiveData}，用于表示 LiveData 是当成事件总线用的，需要注意：
 * - 观察者在非激活状态（onStart 之后，onPause 之前）时不会产生回调，会丢失事件
 * - postValue 可能会被覆盖，只能用 setValue 来更新值
 * - LiveData 的事件都是黏性的，不使用时手动抛出一个 null 事件，以防下次绑定时会发送存在之前的旧数据；
 *
 * @see LiveDataUtils
 * @see LiveEventObserver
 * @see EventMutableLiveData
 * @see EventMediatorLiveData
 *
 * @author funnywolf
 * @since 2019-05-06
 */
@Target(ElementType.FIELD)
public @interface AsEventBus {
}
```
## 4、`LiveDataUtils` 中其他工具简介
这个工具包里还有其他一些我平时常用的小工具，这里简单分享下：
 - `StateData` 是含有状态和错误信息的数据包装类，因为 `LiveData` 只有一个 `onChanged` 回调，无法知道数据状态，所以搞了这个类
 - `RxLiveData` 继承自 `MutableLiveData`，实现了 `Disposable` 和 `Observer` 接口，主要为了数据从 `RxJava` 到 `LiveData` 的转换
 - `LiveDataBus`，一个基于 `LiveData` 的事件总线，但是不推荐用。事件总线这玩意尽量不要用，除非是不用不行的场合，写的时候很香，之后维护起来很麻烦

## 5、总结
不得不说谷歌的生命周期库真的很强大，不仅给我们提供了现成的工具，还给予了我们方便 DIY 的能力，一个不到五百行的 `LiveData` 都能玩出很多花样。以上也只是自己的一些经验总结，难免会有不足，欢迎各位批评指正。