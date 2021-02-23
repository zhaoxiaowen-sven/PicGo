WindowManager DecorView:顶级view ViewRoot 

## 一、入口流程
1.ActivityThread.handleResumeActivity
2.WindowManager(Impl).addView
3.WindowManagerGlobal.addView
4.ViewRootImpl.setView
5.ViewRootImpl.requestLayout
6.ViewRootImpl.scheduleTraversals

    @Override
    public void requestLayout() {
        if (!mHandlingLayoutInLayoutRequest) {
            checkThread();
            mLayoutRequested = true;
            scheduleTraversals();
        }
    }
    
    void scheduleTraversals() {
        if (!mTraversalScheduled) {
            mTraversalScheduled = true;
            mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
            mChoreographer.postCallback(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
            if (!mUnbufferedInputDispatch) {
                scheduleConsumeBatchedInput();
            }
            notifyRendererOfFramePending();
            pokeDrawLockIfNeeded();
        }
    }
    
    final class TraversalRunnable implements Runnable {
        @Override
        public void run() {
            doTraversal();
        }
    }
    
    void doTraversal() {
        if (mTraversalScheduled) {
            mTraversalScheduled = false;
            mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);
    
            if (mProfile) {
                Debug.startMethodTracing("ViewAncestor");
            }
    
            performTraversals();
    
            if (mProfile) {
                Debug.stopMethodTracing();
                mProfile = false;
            }
        }
    }

7.ViewRootImpl.performTraversals

    private void performTraversals() {
        // cache mView since it is used so much below...
        //我们在Step3知道，mView就是DecorView根布局
        final View host = mView;
        //在Step3 成员变量mAdded赋值为true，因此条件不成立
        if (host == null || !mAdded)
            return;
        //是否正在遍历
        mIsInTraversal = true;
        //是否马上绘制View
        mWillDrawSoon = true;
        ...
        WindowManager.LayoutParams lp = mWindowAttributes;
        //顶层视图DecorView所需要窗口的宽度和高度
        int desiredWindowWidth;
        int desiredWindowHeight;
        ....
        //在构造方法中mFirst已经设置为true，表示是否是第一次绘制DecorView
        if (mFirst) {
            mFullRedrawNeeded = true;
            mLayoutRequested = true;
            //如果窗口的类型是有状态栏的，那么顶层视图DecorView所需要窗口的宽度和高度就是除了状态栏
            if (lp.type == WindowManager.LayoutParams.TYPE_STATUS_BAR_PANEL
                    || lp.type == WindowManager.LayoutParams.TYPE_INPUT_METHOD) {
                // NOTE -- system code, won't try to do compat mode.
                Point size = new Point();
                mDisplay.getRealSize(size);
                desiredWindowWidth = size.x;
                desiredWindowHeight = size.y;
            } else {//否则顶层视图DecorView所需要窗口的宽度和高度就是整个屏幕的宽高
                DisplayMetrics packageMetrics =
                    mView.getContext().getResources().getDisplayMetrics();
                desiredWindowWidth = packageMetrics.widthPixels;
                desiredWindowHeight = packageMetrics.heightPixels;
            }
    }
    ...
    //获得view宽高的测量规格，mWidth和mHeight表示窗口的宽高，lp.width和lp.height表示DecorView根布局宽和高
     int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);
     int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);
    
    // Ask host how big it wants to be
    //执行测量操作
    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
    ...
    //执行布局操作
    performLayout(lp, desiredWindowWidth, desiredWindowHeight);
    ...
    //执行绘制操作
    performDraw();
    }


performMeasure -> measure 获取view 测量后的宽高
performLayout -> layout view 4 个定点的位置和实际的宽高
performDraw -> draw 绘制view

## 二、MeasureSpec
Measure很大程度上决定了一个view的尺寸规格。MeasureSpec 代表了一个32位的int值，高2位代表SpecMode，低30位代表SpecSize,SpecMode代表测量模式，SpecSize代表某种测量模式下的规格大小。
## 2.1 SpecMode: 
UNSPECIFIED：父容器对view没有任何限制
EXACTLY: 精准模式 match_parent 或者 具体参数
AT_MOST: 最大模式 wrap_content

    public static class MeasureSpec {
        private static final int MODE_SHIFT = 30;
        private static final int MODE_MASK  = 0x3 << MODE_SHIFT;
        
        /**
         * Measure specification mode: The parent has not imposed any constraint
         * on the child. It can be whatever size it wants.
         */
        public static final int UNSPECIFIED = 0 << MODE_SHIFT;
    
        /**
         * Measure specification mode: The parent has determined an exact size
         * for the child. The child is going to be given those bounds regardless
         * of how big it wants to be.
         */
        public static final int EXACTLY     = 1 << MODE_SHIFT;
    
        /**
         * Measure specification mode: The child can be as large as it wants up
         * to the specified size.
         */
        public static final int AT_MOST     = 2 << MODE_SHIFT;
    
        public static int makeMeasureSpec(@IntRange(from = 0, to = (1 << MeasureSpec.MODE_SHIFT) - 1) int size,
                                          @MeasureSpecMode int mode) {
            if (sUseBrokenMakeMeasureSpec) {
                return size + mode;
            } else {
                return (size & ~MODE_MASK) | (mode & MODE_MASK);
            }
        }
    
        public static int getMode(int measureSpec) {
            //noinspection ResourceType
            return (measureSpec & MODE_MASK);
        }
        
                public static int getSize(int measureSpec) {
            return (measureSpec & ~MODE_MASK);
        }
    }

## 2.2 MeasureSpec & LayoutParams
view 的 MeasureSpec 是由父容器的和自身的LayoutParams决定的。
1.对于DecorView，其MeasureSpec 由窗口的尺寸和自身的LayoutParams决定
ViewRootImpl.getRootMeasureSpec
    
    /**
     * Figures out the measure spec for the root view in a window based on it's
     * layout params.
     */
    private static int getRootMeasureSpec(int windowSize, int rootDimension) {
        int measureSpec;
        switch (rootDimension) {
    
        case ViewGroup.LayoutParams.MATCH_PARENT:
            // Window can't resize. Force root view to be windowSize.
            measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.EXACTLY);
            break;
        case ViewGroup.LayoutParams.WRAP_CONTENT:
            // Window can resize. Set max size for root view.
            measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.AT_MOST);
            break;
        default:
            // Window wants to be an exact size. Force root view to be that size.
            measureSpec = MeasureSpec.makeMeasureSpec(rootDimension, MeasureSpec.EXACTLY);
            break;
        }
        return measureSpec;
    }

2.对于普通view，其MeasureSpec 由父容器的MeasureSpec和其自身的LayoutParams决定
对于普通view,这里指的是我们布局中的view，view的measure过程由ViewGroup传递而来。

    protected void measureChildWithMargins(View child,
            int parentWidthMeasureSpec, int widthUsed,
            int parentHeightMeasureSpec, int heightUsed) {
      
        final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();
    
        final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
                mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                        + widthUsed, lp.width);
        final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
                mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin
                        + heightUsed, lp.height);
    
        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    }


    public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
        int specMode = MeasureSpec.getMode(spec);
        int specSize = MeasureSpec.getSize(spec);
        // 子view的可用空间大小是父view的尺寸 - padding
        int size = Math.max(0, specSize - padding);
    
        int resultSize = 0;
        int resultMode = 0;
    
        switch (specMode) {
        // Parent has imposed an exact size on us
        case MeasureSpec.EXACTLY:
            if (childDimension >= 0) {
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size. So be it.
                resultSize = size;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size. It can't be
                // bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;
    
        // Parent has imposed a maximum size on us
        case MeasureSpec.AT_MOST:
            if (childDimension >= 0) {
                // Child wants a specific size... so be it
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size, but our size is not fixed.
                // Constrain child to not be bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size. It can't be
                // bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;
    
        // Parent asked to see how big we want to be
        case MeasureSpec.UNSPECIFIED:
            if (childDimension >= 0) {
                // Child wants a specific size... let him have it
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size... find out how big it should
                // be
                resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                resultMode = MeasureSpec.UNSPECIFIED;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size.... find out how
                // big it should be
                resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                resultMode = MeasureSpec.UNSPECIFIED;
            }
            break;
        }
        //noinspection ResourceType
        return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
    }

对于普通view，如果指定了固定宽高，不管父容器的MeasureSpec是什么，View的MeasureSpec都是MeasureSpec.EXACTLY。当View 的宽高是MATCH_PARENT时，如果父容器的MeasureSpec是MeasureSpec.EXACTLY，那么view也是MeasureSpec.EXACTLY模式；如果父容器是MeasureSpec.AT_MOST，那么view也是MeasureSpec.AT_MOST，且view的大小不能比父容器的剩余空间还大。当view的宽高是WRAP_CONTENT时，不管父容器的MeasureSpec是什么，view的测量模式总是MeasureSpec.AT_MOST，且view的大小不超过父容器的剩余空间。
![view_measurerule](https://leanote.com/api/file/getImage?fileId=5ae6c38cab6441695200079e)

参考：
《Android开发艺术探索》
https://blog.csdn.net/feiduclear_up/article/details/46732879
https://blog.csdn.net/feiduclear_up/article/details/46711921