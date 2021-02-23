本文主要介绍View的事件分发流程。
![touchEvent](https://leanote.com/api/file/getImage?fileId=5ae6d5e9ab64416766000948)
## 一、事件分发的对象 MotionEvent
在Android设备中，触摸事件主要包括点按、长按、拖拽、滑动等，点按又包括单击和双击，另外还包括单指操作和多指操作等。一个最简单的用户触摸事件一般经过以下几个流程：手指按下，手指滑动，手指抬起
Android把这些事件的每一步抽象为MotionEvent这一概念，MotionEvent包含了触摸的坐标位置，点按的数量(手指的数量)，时间点等信息，用于描述用户当前的具体动作，常见的MotionEvent有下面几种类型：
ACTION_DOWN，ACTION_UP，ACTION_MOVE，ACTION_CANCEL
其中，ACTION_DOWN、ACTION_MOVE、ACTION_UP就分别对应于上面的手指按下、手指滑动、手指抬起操作，即一个最简单的用户操作包含了一个ACTION_DOWN事件，若干个ACTION_MOVE事件和一个ACTION_UP事件。

## 二、事件传递顺序       
一个点击事件产生后，传递顺序：Activity -> Window -> ViewGroup -> View；如果一个 View 的 onTouchEvent 返回 false 即 View 没有处理事件，那么它的父容器的onTouchEvent 会被调用，以此类推，所有元素都不处理该事件，最终将传递给 Activity 处理，即 Activity 的 onTouchEvent 会被调用，事件顺序为：View -> ViewGroup -> Window -> Activity。

## 三、事件分发的三个重要过程  
1 dispatchTouchEvent:用于事件的分发，所有的事件都要通过此方法进行分发，决定是自己对事件进行消费还是交由子View处理
2 onInterceptTouchEvent 是ViewGroup中独有的方法，若返回true表示拦截当前事件，交由自己的onTouchEvent()进行处理，返回false表示不拦截
3 onTouchEvent 主要用于事件的处理，返回true表示消费当前事件

三者的关系如下：

    public boolean dispatchTouchEvent(MotionEvent ev) {
      //表示事件是否被分发出去（消耗），默认为false，即默认不拦截
        boolean consume = false;
        if (onInterceptTouchEvent(ev)) {
          //如果拦截了，那么事件交给当前view的onTouchEvent(ev)
          //方法进行处理，返回值即onTouchEvent(ev)的结果
            consume = onTouchEvent(ev);
        } else {
         //如果不拦截，那么事件交给子view的dispatchTouchEvent(ev)
          //方法进行处理，返回值即是dispatchTouchEvent(ev)事件
          //处理结果
            consume = child.dispatchTouchEvent(ev);
        }
        return consume;
    }

  

## 四、Activity 的事件分发

## 4.1 Activity.dispatchTouchEvent

    public boolean dispatchTouchEvent(MotionEvent ev) {
        //1.判断当前触摸事件的类型，如果是ACTION_DOWN事件，会触发onUserInteraction方法
        if (ev.getAction() == MotionEvent.ACTION_DOWN) {
        //2.当有任意一个按键、触屏或者轨迹球事件发生时，栈顶Activity的onUserInteraction会被触发。
        如果我们需要知道用户是不是正在和设备交互，可以在子类中重写这个方法，去获取通知（比如取消屏保这个场景）。
            onUserInteraction();
        }
        //3.调用当前Activity的顶层窗口PhoneWindow分发事件 
        if (getWindow().superDispatchTouchEvent(ev)) {
            return true;
        }
        //4.当没有任何view对事件处理时，交由activity的onTouchEvent
        return onTouchEvent(ev);
    }

## 4.2 PhoneWindow.superDispatchTouchEvent

    public class PhoneWindow extends Window implements MenuBuilder.Callback {
        ...
    	@Override
    	public boolean superDispatchTouchEvent(MotionEvent event) {
    	    return mDecor.superDispatchTouchEvent(event);
    	}
    	private final class DecorView extends FrameLayout implements RootViewSurfaceTaker {
    	    ...
    	    public boolean superDispatchTouchEvent(MotionEvent event) {
    	        return super.dispatchTouchEvent(event);
    	    }
    	    ...
    	}
    }

PhoneWindow内部调用了DecorView的同名方法，而DecorView其实是FrameLayout的子类，FrameLayout并没有重写dispatchTouchEvent方法，所以事件开始交由ViewGroup的dispatchTouchEvent开始分发。

## 五、ViewGroup的事件分发：

## 5.1 判断事件是否需要被viewGroup拦截
当事件类型是ACTION_DOWN或者mFirstTouchTarget不为空时，才会走是否需要拦截事件这一判断，如果事件是ACTION_DOWN的后续事件（如ACTION_MOVE、ACTION_UP等），且在传递ACTION_DOWN事件过程中没有找到目标子View时，事件将会直接被拦截，交给ViewGroup自己处理。

    // Check for interception.
    final boolean intercepted;
    //1.viewGroup 会在2种情况下判断是否拦截事件: 1. Action_Down 2.mFirstTouchTarget != null 这个条件指的事件被viewGroup的子元素处理了
    if (actionMasked == MotionEvent.ACTION_DOWN
            || mFirstTouchTarget != null) {
        // 对于actiondown 事件这个标记位永远为false，每次都会去判断viewGroup是否拦截事件
        final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
        if (!disallowIntercept) {
            intercepted = onInterceptTouchEvent(ev);
            ev.setAction(action); // restore action in case it was changed
        } else {
            intercepted = false;
        }
    } else {
        // There are no touch targets and this action is not an initial down
        // so this view group continues to intercept touches.
        //2. 一旦事件由当前viewGroup处理了，那么mFirstTouchTarget == null，而且事件也不是Action_DOWN的话，后续UP和MOVE事件都由
        intercepted = true;
    }

## 5.2 遍历所有子View，逐个分发事件
执行遍历分发的条件是：当前事件是ACTION_DOWN、ACTION_POINTER_DOWN或者ACTION_HOVER_MOVE三种类型中的一个（后两种用的比较少，暂且忽略）。所以，如果事件是ACTION_DOWN的后续事件，如ACTION_UP事件，将不会进入遍历流程！

进入遍历流程后，拿到一个子View，首先会判断触摸点是不是在子View范围内，如果不是直接跳过该子View；
否则通过dispatchTransformedTouchEvent方法，间接调用child.dispatchTouchEvent达到传递的目的；

如果dispatchTransformedTouchEvent返回true，即事件被子View消费，就会把mFirstTouchTarget设置为child，即不为null，并将alreadyDispatchedToNewTouchTarget设置为true，然后跳出循环，事件不再继续传递给其他子View。

可以理解为，这一步的主要作用是，在事件的开始，即传递ACTION_DOWN事件过程中，找到一个需要消费事件的子View，我们可以称之为目标子View，执行第一次事件传递，并把mFirstTouchTarget设置为这个目标子View

    // Check for cancelation.
    final boolean canceled = resetCancelNextUpFlag(this)
            || actionMasked == MotionEvent.ACTION_CANCEL;
    
    // Update list of touch targets for pointer down, if needed.
    final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0;
    TouchTarget newTouchTarget = null;
    boolean alreadyDispatchedToNewTouchTarget = false;
    if (!canceled && !intercepted) {
    
        // If the event is targeting accessiiblity focus we give it to the
        // view that has accessibility focus and if it does not handle it
        // we clear the flag and dispatch the event to all children as usual.
        // We are looking up the accessibility focused host to avoid keeping
        // state since these events are very rare.
        View childWithAccessibilityFocus = ev.isTargetAccessibilityFocus()
                ? findChildWithAccessibilityFocus() : null;
    
        if (actionMasked == MotionEvent.ACTION_DOWN
                || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
            final int actionIndex = ev.getActionIndex(); // always 0 for down
            final int idBitsToAssign = split ? 1 << ev.getPointerId(actionIndex)
                    : TouchTarget.ALL_POINTER_IDS;
    
            // Clean up earlier touch targets for this pointer id in case they
            // have become out of sync.
            removePointersFromTouchTargets(idBitsToAssign);
    
            final int childrenCount = mChildrenCount;
            if (newTouchTarget == null && childrenCount != 0) {
                final float x = ev.getX(actionIndex);
                final float y = ev.getY(actionIndex);
                // Find a child that can receive the event.
                // Scan children from front to back.
                final ArrayList<View> preorderedList = buildTouchDispatchChildList();
                final boolean customOrder = preorderedList == null
                        && isChildrenDrawingOrderEnabled();
                final View[] children = mChildren;
                for (int i = childrenCount - 1; i >= 0; i--) {
                    final int childIndex = getAndVerifyPreorderedIndex(
                            childrenCount, i, customOrder);
                    final View child = getAndVerifyPreorderedView(
                            preorderedList, children, childIndex);
    
                    // If there is a view that has accessibility focus we want it
                    // to get the event first and if not handled we will perform a
                    // normal dispatch. We may do a double iteration but this is
                    // safer given the timeframe.
                    if (childWithAccessibilityFocus != null) {
                        if (childWithAccessibilityFocus != child) {
                            continue;
                        }
                        childWithAccessibilityFocus = null;
                        i = childrenCount - 1;
                    }
    
                    if (!canViewReceivePointerEvents(child)
                            || !isTransformedTouchPointInView(x, y, child, null)) {
                        ev.setTargetAccessibilityFocus(false);
                        continue;
                    }
    
                    newTouchTarget = getTouchTarget(child);
                    if (newTouchTarget != null) {
                        // Child is already receiving touch within its bounds.
                        // Give it the new pointer in addition to the ones it is handling.
                        newTouchTarget.pointerIdBits |= idBitsToAssign;
                        break;
                    }
    
                    resetCancelNextUpFlag(child);
                    if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                        // Child wants to receive touch within its bounds.
                        mLastTouchDownTime = ev.getDownTime();
                        if (preorderedList != null) {
                            // childIndex points into presorted list, find original index
                            for (int j = 0; j < childrenCount; j++) {
                                if (children[childIndex] == mChildren[j]) {
                                    mLastTouchDownIndex = j;
                                    break;
                                }
                            }
                        } else {
                            mLastTouchDownIndex = childIndex;
                        }
                        mLastTouchDownX = ev.getX();
                        mLastTouchDownY = ev.getY();
                        newTouchTarget = addTouchTarget(child, idBitsToAssign);
                        alreadyDispatchedToNewTouchTarget = true;
                        break;
                    }
    
                    // The accessibility focus didn't handle the event, so clear
                    // the flag and do a normal dispatch to all children.
                    ev.setTargetAccessibilityFocus(false);
                }
                if (preorderedList != null) preorderedList.clear();
            }
    
            if (newTouchTarget == null && mFirstTouchTarget != null) {
                // Did not find a child to receive the event.
                // Assign the pointer to the least recently added target.
                newTouchTarget = mFirstTouchTarget;
                while (newTouchTarget.next != null) {
                    newTouchTarget = newTouchTarget.next;
                }
                newTouchTarget.pointerIdBits |= idBitsToAssign;
            }
        }
    }
### 5.2.1 dispatchTransformedTouchEvent
如果child不为空，就调用child的dispatchTouchEvent，如果child为空，就调用ViewGroup的dispatchTouchEvent。

    /**
     * Transforms a motion event into the coordinate space of a particular child view,
     * filters out irrelevant pointer ids, and overrides its action if necessary.
     * If child is null, assumes the MotionEvent will be sent to this ViewGroup instead.
     */
    private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
            View child, int desiredPointerIdBits) {
        final boolean handled;
            if (child == null) {
                handled = super.dispatchTouchEvent(event);
            } else {
                handled = child.dispatchTouchEvent(event);
            }
        }
    ...
    }

### 5.2.2 addTouchTarget 
为mFirstTouchTarget设置值

    /**
     * Adds a touch target for specified child to the beginning of the list.
     * Assumes the target child is not already present.
     */
    private TouchTarget addTouchTarget(@NonNull View child, int pointerIdBits) {
        final TouchTarget target = TouchTarget.obtain(child, pointerIdBits);
        target.next = mFirstTouchTarget;
        mFirstTouchTarget = target;
        return target;
    }

## 5.3 将事件交给ViewGroup自己或者目标子View处理
经过上面一步后，如果mFirstTouchTarget仍然为空，说明没有任何一个子View消费事件，将同样会调用dispatchTransformedTouchEvent，但此时这个方法的View child参数为null，所以调用的其实是super.dispatchTouchEvent(event)，即事件交给ViewGroup自己处理。ViewGroup是View的子View，所以事件将会使用View的dispatchTouchEvent(event)方法判断是否消费事件。

反之，如果mFirstTouchTarget不为null，说明上一次事件传递时，找到了需要处理事件的目标子View，此时，ACTION_DOWN的后续事件，如ACTION_UP等事件，都会传递至mFirstTouchTarget中保存的目标子View中。这里面还有一个小细节，如果在上一节遍历过程中已经把本次事件传递给子View，alreadyDispatchedToNewTouchTarget的值会被设置为true，代码会判断alreadyDispatchedToNewTouchTarget的值，避免做重复分发。

    // Dispatch to touch targets.
    if (mFirstTouchTarget == null) {
        // No touch targets so treat this as an ordinary view.
        handled = dispatchTransformedTouchEvent(ev, canceled, null,
                TouchTarget.ALL_POINTER_IDS);
    } else {
        // Dispatch to touch targets, excluding the new touch target if we already
        // dispatched to it.  Cancel touch targets if necessary.
        TouchTarget predecessor = null;
        TouchTarget target = mFirstTouchTarget;
        while (target != null) {
            final TouchTarget next = target.next;
            if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                handled = true;
            } else {
                final boolean cancelChild = resetCancelNextUpFlag(target.child)
                        || intercepted;
                if (dispatchTransformedTouchEvent(ev, cancelChild,
                        target.child, target.pointerIdBits)) {
                    handled = true;
                }
                if (cancelChild) {
                    if (predecessor == null) {
                        mFirstTouchTarget = next;
                    } else {
                        predecessor.next = next;
                    }
                    target.recycle();
                    target = next;
                    continue;
                }
            }
            predecessor = target;
            target = next;
        }
    }

至此，viewGroup的事件分发流程结束，下面介绍下view的事件分发。
    
## 六、View的事件分发

## 6.1 dispatchEvent的流程
1.如果View设置了OnTouchListener，且处于enable状态时，会先调用mOnTouchListener的onTouch方法
2.onTouch返回false，事件传递给onTouchEvent方法继续处理
3.如果onTouchEvent也没有消费这个事件，将返回false，告知上层parent将事件给其他兄弟View

    public boolean dispatchTouchEvent(MotionEvent event) {  
        if (onFilterTouchEventForSecurity(event)) {
            if ((mViewFlags & ENABLED_MASK) == ENABLED && handleScrollBarDragging(event)) {
                result = true;
            }
            //noinspection SimplifiableIfStatement
            ListenerInfo li = mListenerInfo;
            //1.判断view有没有设置onTouchListener且是enabled状态
            if (li != null && li.mOnTouchListener != null
                    && (mViewFlags & ENABLED_MASK) == ENABLED
                    && li.mOnTouchListener.onTouch(this, event)) {
                result = true;
            }
            //2.onTouchEvent有没有消费事件    
            if (!result && onTouchEvent(event)) {
                result = true;
            }
            ...
            return result;
    }

## 6.2 onTouchEvent的流程
1.不可用的view会消耗事件

2.如果view有设置mTouchDelegate，那么会执行TouchDelegate的onTouchEvent事件

3.如果View是enable的且处于可点击状态，事件将被这个View消费：在方法返回前，onTouchEvent会根据MotionEvent事件类型做出不同响应，主要是处理ACTION_UP的事件。

    public boolean onTouchEvent(MotionEvent event) {
        final float x = event.getX();
        final float y = event.getY();
        final int viewFlags = mViewFlags;
        final int action = event.getAction();
    
        final boolean clickable = ((viewFlags & CLICKABLE) == CLICKABLE
                || (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)
                || (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE;
    
        if ((viewFlags & ENABLED_MASK) == DISABLED) {
            if (action == MotionEvent.ACTION_UP && (mPrivateFlags & PFLAG_PRESSED) != 0) {
                setPressed(false);
            }
            mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
            // A disabled view that is clickable still consumes the touch
            // events, it just doesn't respond to them.
            return clickable;
        }
        if (mTouchDelegate != null) {
            if (mTouchDelegate.onTouchEvent(event)) {
                return true;
            }
        }
        
        if (clickable || (viewFlags & TOOLTIP) == TOOLTIP) {
            ...
            }
            return true;
        }
    
        return false;
    }


## 6.3 onLongClick和onClick
onLongClick是在ACTION_DOWN时调用的，onClick 是在ACTION_UP时调用的。performLongClick 最终会调用到mOnLongClickListener.onLongClick()，如果onLongClick中返回的时true，mHasPerformedLongPress 就被置为true，那么onClick方法在ACTION_UP的时候就不会被调用了。

    private final class CheckForLongPress implements Runnable {
        private int mOriginalWindowAttachCount;
        private float mX;
        private float mY;
    
        @Override
        public void run() {
            if (isPressed() && (mParent != null)
                    && mOriginalWindowAttachCount == mWindowAttachCount) {
                if (performLongClick(mX, mY)) {
                    mHasPerformedLongPress = true;
                }
            }
        }
        ...
    }
    
    public boolean onTouchEvent(MotionEvent event) {
        case MotionEvent.ACTION_UP:
            if (!mHasPerformedLongPress && !mIgnoreNextUpEvent) {
            // This is a tap, so remove the longpress check
            removeLongPressCallback();
    
            // Only perform take click actions if we were in the pressed state
            if (!focusTaken) {
                // Use a Runnable and post this rather than calling
                // performClick directly. This lets other visual state
                // of the view update before click actions start.
                if (mPerformClick == null) {
                    mPerformClick = new PerformClick();
                }
                if (!post(mPerformClick)) {
                    performClick();
                }
            }
        break;
    }

[onClick和onLongClick](https://blog.csdn.net/ddna/article/details/5451722)

## 六、总结
事件由Activity的dispatchTouchEvent()开始，将事件传递给当前Activity的根ViewGroup：mDecorView，事件开始自上而下进行传递，直至被消费。

事件传递至ViewGroup时，调用dispatchTouchEvent()进行分发处理:

检查送否应该对事件进行拦截:onInterceptTouchEvent()，若为true，跳过2步骤；
将事件依次分发给子View，若事件被某个View消费了，将不再继续分发；
如果2中没有子View对事件进行消费或者子View的数量为零，事件将由ViewGroup自己处理，处理流程和View的处理流程一致；
事件传递至View的dispatchTouchEvent()时， 首先会判断OnTouchListener是否存在，倘若存在，则执行onTouch()，若onTouch()未对事件进行消费，事件将继续交由onTouchEvent处理，根据上面分析可知，View的onClick事件是在onTouchEvent的ACTION_UP中触发的，因此，onTouch事件优先于onClick事件。

若事件在自上而下的传递过程中一直没有被消费，而且最底层的子View也没有对其进行消费，事件会反向向上传递，此时，父ViewGroup可以对事件进行消费，若仍然没有被消费的话，最后会回到Activity的onTouchEvent。

如果一个子View没有消费ACTION_DOWN类型的事件，那么事件将会被另一个子View或者ViewGroup自己消费，之后的事件都只会传递给目标子View（mFirstTouchTarget）或者ViewGroup自身。简单来说，就是如果一个View没有消费ACTION_DOWN事件，后续事件也不会传递进来。

![touch](https://leanote.com/api/file/getImage?fileId=5ae9a529ab6441232a001958)

## 七、关键结论：
1.同一个事件序列是指从手指触摸屏幕那一刻开始，中间包含数量不定的 move 事件到手指离开屏幕那一刻（down->move…move->up)。
2.正常情况下一个事件序列只能被一个 View 拦截且消耗，每个 View 一旦决定拦截，同一个事件序列所有事件都会直接交给它处理，并且它的 onInterceptTouchEvent 不会再被调用。
3.某个 View 一旦开始处理事件，如果它不消耗 ACTION_DOWN（ onTouchEvent 返回了 false ），那么同一事件序列中其他事件都不会再交给它来处理，事件将重新交给他的父元素处理，即父元素的 onTouchEvent 会被调用。
4.如果某个 View 不消耗除 ACTION_DOWN 以外的其他事件，那么这个点击事件会消失，此时父元素的 onTouchEvent 并不会被调用，并且当前 View 可以收到后续事件，最终这些消失的点击事件会传递给 Activity 处理。
5.ViewGroup 默认不拦截任何事件，ViewGroup 的 onInterceptTouchEvent 方法默认返回 false。
6.View 没有 onInterceptTouchEvent 方法，一旦有事件传递给它，那么它的 onTouchEvent 方法就会被调用。
7.View 的 onTouchEvent 方法默认消耗事件（返回 true ），除非他是不可点击的（ clickable 和 longClickable 同时为 false ）。View 的 longClickable 属性默认都为 false ，clickable 属性分情况，Button 默认为 true，TextView 默认为 false。
8.onClick 发生的前提是View可点击，并且它收到了down 和 up 事件。
事件传递过程是由外而内，事件总是先传递给父元素，然后在由父元素分发给子 View，通过requestDisallowInterceptTouchEvent 方法可以在子元素干预父元素的事件分发过程，但 ACTION_DOWN 事件除外。

参考：
Android开发艺术探索
http://gityuan.com/2015/09/19/android-touch/
http://allenfeng.com/2017/02/22/android-touch-event-transfer-mechanism/