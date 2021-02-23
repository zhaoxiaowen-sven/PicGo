## 一、layout过程
View视图绘制流程中的布局layout是由ViewRootImpl中的performLayout成员方法开始的，

    private void performLayout(WindowManager.LayoutParams lp, int desiredWindowWidth,
            int desiredWindowHeight) {
        ...
        //标记当前开始布局
        mInLayout = true;
        //mView就是DecorView
        final View host = mView;
        ...
        //DecorView请求布局
        host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());
        //标记布局结束
        mInLayout = false;
        ...
    }
代码第10行发现，DecorView的四个位置左=0，顶=0，右=屏幕宽，底=屏幕宽，说明DecorView布局的位置是从屏幕最左最顶端开始布局，到屏幕最低最右结束。因此DecorView根布局是充满整个屏幕的。   

layout过程主要处理了2件事情：
1.通过setFrame设置4个顶点的位置确定view本身的位置
2.调用onLayout方法，确定子元素的位置，这个方法在View和ViewGroup均没有真正的实现，需要各个view或layout去实现，ViewGroup 中 onLayout 多了关键字abstract的修饰，也就说对于继承自 ViewGroup 的自定义 View 必须要重写 onLayout 方法

    public void layout(int l, int t, int r, int b) {
        if ((mPrivateFlags3 & PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT) != 0) {
            onMeasure(mOldWidthMeasureSpec, mOldHeightMeasureSpec);
            mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
        }
    
        int oldL = mLeft;
        int oldT = mTop;
        int oldB = mBottom;
        int oldR = mRight;
        //1.设置4个顶点的位置
        boolean changed = isLayoutModeOptical(mParent) ?
                setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);
    
        if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
            //2.调用onLayout方法
            onLayout(changed, l, t, r, b);
    
            if (shouldDrawRoundScrollbar()) {
                if(mRoundScrollbarRenderer == null) {
                    mRoundScrollbarRenderer = new RoundScrollbarRenderer(this);
                }
            } else {
                mRoundScrollbarRenderer = null;
            }
    
            mPrivateFlags &= ~PFLAG_LAYOUT_REQUIRED;
    
            ListenerInfo li = mListenerInfo;
            if (li != null && li.mOnLayoutChangeListeners != null) {
                ArrayList<OnLayoutChangeListener> listenersCopy =
                        (ArrayList<OnLayoutChangeListener>)li.mOnLayoutChangeListeners.clone();
                int numListeners = listenersCopy.size();
                for (int i = 0; i < numListeners; ++i) {
                    listenersCopy.get(i).onLayoutChange(this, l, t, r, b, oldL, oldT, oldR, oldB);
                }
            }
        }
    
        mPrivateFlags &= ~PFLAG_FORCE_LAYOUT;
        mPrivateFlags3 |= PFLAG3_IS_LAID_OUT;
    }


​    
getWidth() 、getHeight() 和 getMeasuredWidth()、getMeasuredHeight() 这两对函数之间的区别，即 View 的测量宽/高和最终显示宽/高之间的区别。首先我们看一下 getWith() 和 getHeight() 方法的具体实现：

    public final int getWidth() {
        return mRight - mLeft;
    }
     public final int getHeight() {
        return mBottom - mTop;
    }

通过 getWith() 和 getHeight() 源码和上面 setChildFrame(View child, int left, int top, int width, int height) 方法设置子元素四个顶点位置的四个变量 mLeft、mTop、mRight、mBottom 的赋值过程来看，默认情况下 getWidth() 、getHeight() 方法返回的值正好就是 view 的测量宽/高，只不过 view 的测量宽/高形成于 view 的measure 过程，而最终宽/高形成于 view 的 layout 方法中，但是对于特殊情况，两者的值是不相等的，就是我们在 layout 过程中不按默认常规套路出牌，即不使用 measure 过程得到的 mMeasuredWidth 和 mMeasuredHeight ，而是人为的去自己根据需要设定的一个值的情况，例如以下代码，重写 view 的 layout 方法：

    public void layout(int l, int t, int r, int b) {
        //在得到的测量值基础上加100
        super.layout(int l, int t, int r+100, int b+100);
    }
上面代码会导致在任何情况下 view 的最终宽/高总会比测量宽高大100px。

## 二、draw 过程：
draw 的作用是将 view 绘制到屏幕上，view 的绘制过程遵守以下几个步骤：

    绘制背景：background.draw(canvas)；
    绘制自己：onDraw()；
    绘制 children：dispatchDraw；
    绘制装饰：onDrawScrollBars。

   public void draw(Canvas canvas) {
        final int privateFlags = mPrivateFlags;
        final boolean dirtyOpaque = (privateFlags & PFLAG_DIRTY_MASK) == PFLAG_DIRTY_OPAQUE &&
                (mAttachInfo == null || !mAttachInfo.mIgnoreDirtyState);
        mPrivateFlags = (privateFlags & ~PFLAG_DIRTY_MASK) | PFLAG_DRAWN;

        /*
         * Draw traversal performs several drawing steps which must be executed
         * in the appropriate order:
         *
         *      1. Draw the background
         *      2. If necessary, save the canvas' layers to prepare for fading
         *      3. Draw view's content
         *      4. Draw children
         *      5. If necessary, draw the fading edges and restore layers
         *      6. Draw decorations (scrollbars for instance)
         */
    
        // Step 1, draw the background, if needed
        int saveCount;
    
        if (!dirtyOpaque) {
            drawBackground(canvas);
        }
    
        // skip step 2 & 5 if possible (common case)
        final int viewFlags = mViewFlags;
        boolean horizontalEdges = (viewFlags & FADING_EDGE_HORIZONTAL) != 0;
        boolean verticalEdges = (viewFlags & FADING_EDGE_VERTICAL) != 0;
        if (!verticalEdges && !horizontalEdges) {
            // Step 3, draw the content
            if (!dirtyOpaque) onDraw(canvas);
    
            // Step 4, draw the children
            dispatchDraw(canvas);
    
            // Overlay is part of the content and draws beneath Foreground
            if (mOverlay != null && !mOverlay.isEmpty()) {
                mOverlay.getOverlayView().dispatchDraw(canvas);
            }
    
            // Step 6, draw decorations (foreground, scrollbars)
            onDrawForeground(canvas);
    
            // we're done...
            return;
        }
    ...
    }

dispatchDraw(canvas);来绘制它的子视图，我们进入dispatchDraw(canvas);
    
    protected void dispatchDraw(Canvas canvas) {
    
    }
这也是一个空实现，既然是这样，那么其实现的逻辑也会在它的子类中实现了，由于只有ViewGroup容器才有其子视图，因此，该方法的实现应该在ViewGroup类中，我们进入ViewGroup类中看其源码如下：
    
    protected void dispatchDraw(Canvas canvas) {
        ...
        for (int i = 0; i < childrenCount; i++) {
            while (transientIndex >= 0 && mTransientIndices.get(transientIndex) == i) {
                final View transientChild = mTransientViews.get(transientIndex);
                if ((transientChild.mViewFlags & VISIBILITY_MASK) == VISIBLE ||
                        transientChild.getAnimation() != null) {
                    more |= drawChild(canvas, transientChild, drawingTime);
                }
                transientIndex++;
                if (transientIndex >= transientCount) {
                    transientIndex = -1;
                }
            }
            final int childIndex = getAndVerifyPreorderedIndex(childrenCount, i, customOrder);
            final View child = getAndVerifyPreorderedView(preorderedList, children, childIndex);
            if ((child.mViewFlags & VISIBILITY_MASK) == VISIBLE || child.getAnimation() != null) {
                more |= drawChild(canvas, child, drawingTime);
            }
        }
        ...
    }    
第 8行viewGroup中会遍历调用drawChild，此处又调用View类中的draw方法来绘制视图，
    
    protected boolean drawChild(Canvas canvas, View child, long drawingTime) {
        return child.draw(canvas, this, drawingTime);
    }    

因此形成了一个嵌套调用，知道所有的子视图View绘制结束。到此关于视图View绘制已经基本完成。

参考：
《Android开发艺术探索》