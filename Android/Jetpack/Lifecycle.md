## Lifecycle

帮助我们轻松的管理UI组件的生命周期，当组件的生存状态改变时，可以避免内存泄漏的问题，并且能轻松的将数据重新加载到UI中。


## 一 、生命周期的绑定过程

### 1.1 LifecycleOwner.getLifecycle

获取当前Activity或者Fragment的生命周期所有者

### 1.2 LifecycleRegistry.addObserver

根据observer和initialState构造ObserverWithState对象statefulObserver，然后将该对象存入mObserverMap，可以简单的把它理解成用来保存观察者的Map。

```java
 public void addObserver(@NonNull LifecycleObserver observer) {
        State initialState = mState == DESTROYED ? DESTROYED : INITIALIZED;
        ObserverWithState statefulObserver = new ObserverWithState(observer, initialState);
        ObserverWithState previous = mObserverMap.putIfAbsent(observer, statefulObserver);

        if (previous != null) {
            return;
        }
        LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
        if (lifecycleOwner == null) {
            // it is null we should be destroyed. Fallback quickly
            return;
        }

        boolean isReentrance = mAddingObserverCounter != 0 || mHandlingEvent;
        State targetState = calculateTargetState(observer);
        mAddingObserverCounter++;
        while ((statefulObserver.mState.compareTo(targetState) < 0
                && mObserverMap.contains(observer))) {
            pushParentState(statefulObserver.mState);
            statefulObserver.dispatchEvent(lifecycleOwner, upEvent(statefulObserver.mState));
            popParentState();
            // mState / subling may have been changed recalculate
            targetState = calculateTargetState(observer);
        }

        if (!isReentrance) {
            // we do sync only on the top level.
            sync();
        }
        mAddingObserverCounter--;
    }
```

### 1.3 LifecycleOwner.ObserverWithState

```java
static class ObserverWithState {
    State mState;
    GenericLifecycleObserver mLifecycleObserver;

    ObserverWithState(LifecycleObserver observer, State initialState) {
        mLifecycleObserver = Lifecycling.getCallback(observer);
        mState = initialState;
    }

    void dispatchEvent(LifecycleOwner owner, Event event) {
        State newState = getStateAfter(event);
        mState = min(mState, newState);
        mLifecycleObserver.onStateChanged(owner, event);
        mState = newState;
    }
}
```

## 二、生命周期的回调过程

生命周期的回调分为2种，Actvity（AppCompatActivity）和Fragment（android.support.v4.app.Fragment）

### 2.1 SupportActivity 和  ReportFragment 的关联

添加了一个没有页面的Fragment来完成Activity的生命周期事件的分发。

```java
protected void onCreate(@Nullable Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    ReportFragment.injectIfNeededIn(this);
}
```

### 2.2 ReportFragment.dispatch

```java
public class ReportFragment extends Fragment {
	...
    @Override
    public void onActivityCreated(Bundle savedInstanceState) {
        super.onActivityCreated(savedInstanceState);
        dispatchCreate(mProcessListener);
        dispatch(Lifecycle.Event.ON_CREATE);
    }

    @Override
    public void onStart() {
        super.onStart();
        dispatchStart(mProcessListener);
        dispatch(Lifecycle.Event.ON_START);
    }

    @Override
    public void onResume() {
        super.onResume();
        dispatchResume(mProcessListener);
        dispatch(Lifecycle.Event.ON_RESUME);
    }

    @Override
    public void onPause() {
        super.onPause();
        dispatch(Lifecycle.Event.ON_PAUSE);
    }

    @Override
    public void onStop() {
        super.onStop();
        dispatch(Lifecycle.Event.ON_STOP);
    }

    @Override
    public void onDestroy() {
        super.onDestroy();
        dispatch(Lifecycle.Event.ON_DESTROY);
        // just want to be sure that we won't leak reference to an activity
        mProcessListener = null;
    }

    private void dispatch(Lifecycle.Event event) {
        Activity activity = getActivity();
        if (activity instanceof LifecycleRegistryOwner) {
            ((LifecycleRegistryOwner) activity).getLifecycle().handleLifecycleEvent(event);
            return;
        }

        if (activity instanceof LifecycleOwner) {
            Lifecycle lifecycle = ((LifecycleOwner) activity).getLifecycle();
            if (lifecycle instanceof LifecycleRegistry) {
                ((LifecycleRegistry) lifecycle).handleLifecycleEvent(event);
            }
        }
    }
}
```

### 2.3 LifecycleRegistry.handleLifecycleEvent

```java
public void handleLifecycleEvent(@NonNull Lifecycle.Event event) {
    State next = getStateAfter(event);
    moveToState(next);
}
```

### 2.3.1  getCurrentState 和 getStateAfter：

通过不同的 events 知道不同的 States 。例如：当你的 events 是 ON_RESUME 的时候就代表他当前的 State 是 STARTE 所以 next State 就是 RESUMED：



```java
@NonNull
@Override
public State getCurrentState() {
    return mState;
}

static State getStateAfter(Event event) {
    switch (event) {
        case ON_CREATE:
        case ON_STOP:
            return CREATED;
        case ON_START:
        case ON_PAUSE:
            return STARTED;
        case ON_RESUME:
            return RESUMED;
        case ON_DESTROY:
            return DESTROYED;
        case ON_ANY:
            break;
    }
    throw new IllegalArgumentException("Unexpected event value " + event);
}
```

### 2.3.2 moveToState

状态分发  ：State中，状态值是从DESTROYED-INITIALIZED-CREATED-STARTED-RESUMED增大，如果当前状态值 < Observer状态值，需要通知Observer减小状态值，直到等于当前状态值；如果当前状态值 > Observer状态值，需要通知Observer增大状态值，直到等于当前状态值。

```java
/**
 * 改变状态
 */
private void moveToState(State next) {
    if (mState == next) {
        return;
    }
    mState = next;
    ......
    sync();
    ......
}

/**
 * 同步Observer状态，并分发事件
 */
private void sync() {
    LifecycleOwner lfecycleOwner = mLifecycleOwner.get();
    if (lifecycleOwner == null) {
        Log.w(LOG_TAG, "LifecycleOwner is garbage collected, you shouldn't try dispatch "
                + "new events from it.");
        return;
    }
    while (!isSynced()) {
        mNewEventOccurred = false;
        // State中，状态值是从DESTROYED-INITIALIZED-CREATED-STARTED-RESUMED增大
        // 如果当前状态值 < Observer状态值，需要通知Observer减小状态值，直到等于当前状态值
        if (mState.compareTo(mObserverMap.eldest().getValue().mState) < 0) {
            backwardPass(lifecycleOwner);
        }
        Entry<LifecycleObserver, ObserverWithState> newest = mObserverMap.newest();
        // 如果当前状态值 > Observer状态值，需要通知Observer增大状态值，直到等于当前状态值
        if (!mNewEventOccurred && newest != null
                && mState.compareTo(newest.getValue().mState) > 0) {
            forwardPass(lifecycleOwner);
        }
    }
    mNewEventOccurred = false;
}

/**
 * 向前传递事件，对应图中的INITIALIZED -> RESUMED
 * 增加Observer的状态值，直到状态值等于当前状态值
 */
private void forwardPass(LifecycleOwner lifecycleOwner) {
    Iterator<Entry<LifecycleObserver, ObserverWithState>> ascendingIterator =
            mObserverMap.iteratorWithAdditions();
    while (ascendingIterator.hasNext() && !mNewEventOccurred) {
        Entry<LifecycleObserver, ObserverWithState> entry = ascendingIterator.next();
        ObserverWithState observer = entry.getValue();
        while ((observer.mState.compareTo(mState) < 0 && !mNewEventOccurred
                && mObserverMap.contains(entry.getKey()))) {
            pushParentState(observer.mState);
            // 分发状态改变事件
            observer.dispatchEvent(lifecycleOwner, upEvent(observer.mState));
            popParentState();
        }
    }
}

/**
 * 向后传递事件，对应图中的RESUMED -> DESTROYED
 * 减小Observer的状态值，直到状态值等于当前状态值
 */
private void backwardPass(LifecycleOwner lifecycleOwner) {
    Iterator<Entry<LifecycleObserver, ObserverWithState>> descendingIterator =
            mObserverMap.descendingIterator();
    while (descendingIterator.hasNext() && !mNewEventOccurred) {
        Entry<LifecycleObserver, ObserverWithState> entry = descendingIterator.next();
        ObserverWithState observer = entry.getValue();
        while ((observer.mState.compareTo(mState) > 0 && !mNewEventOccurred
                && mObserverMap.contains(entry.getKey()))) {
            Event event = downEvent(observer.mState);
            // 分发状态改变事件
            pushParentState(getStateAfter(event));
            observer.dispatchEvent(lifecycleOwner, event);
            popParentState();
        }
    }
}
```


## 三、应用

使用mvp模式时，经常需要监听Activity中的生命周期变化，如果使用了lifecycler组件，那么就可以使用如下的这种方式：

### 3.1  实现 LifecycleObserver

```java
public class ViewModelPresenter implements LifecycleObserver {

    private static final String TAG = "ViewModelPresenter";

    @OnLifecycleEvent(Event.ON_CREATE)
    public void onCreate(@NotNull LifecycleOwner owner) {
        Log.d(TAG, "onCreate: ");
    }

    @OnLifecycleEvent(Event.ON_DESTROY)
    public void onDestroy(@NotNull LifecycleOwner owner) {
        Log.d(TAG, "onDestroy: ");

    }
}
```

### 3.2 addObserver

```java
getLifecycle().addObserver(new ViewModelPresenter());
```

### 3.3 DefaultLifecycleObserver

总结：

Lifecycle其实就是通过监听者的方式，为Activity和Fragment添加生命周期的监听回调。

AppCompatActivity 的绑定过程如下： 调用LifecycleRegistry 的 addObserver，将LifecycleObserver 封装成ObserverWithState并存入我们集合中。在 ObserverWithState 中调用了 Lifecycling.getCallback(observer)反射生成了不同的 GenericLifecycleObserver 对象。

AppCompatActivity 回调是通过关联的ReportFragment在对应的生命周期里调用LifecycleRegistry.dispatch，最后再调用到.ObserverWithState.onStateChange回调到具体的Observer对象。

Activty：

Fragment主要流程和Activity类似，回调过程如下

![Lifecycle Sequnce](https://img-blog.csdnimg.cn/20190117102726258.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3NkX3podXpoaXBlbmc=,size_16,color_FFFFFF,t_70)

## 四、ProcessLifecycleOwnerInitializer

```java
public class ProcessLifecycleOwnerInitializer extends ContentProvider {
    @Override
    public boolean onCreate() {
        LifecycleDispatcher.init(getContext());
        ProcessLifecycleOwner.init(getContext());
        return true;
    }
}
```

参考：
[LifeCycle 源码分析](https://blog.csdn.net/Liberty_zw/article/details/84587257)