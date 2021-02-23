view的measure过程由其measure方法实现，measure方法是一个final方法，这意味着子类不能重写此方法, measure方法的主要逻辑如下：

## 一、 View的measure过程

## 1.1 measure    

    public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
        ...
        // 1.如果上一次的测量规格和这次不一样，则条件满足，重新测量视图View的大小
        if ((mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT ||
                widthMeasureSpec != mOldWidthMeasureSpec ||
                heightMeasureSpec != mOldHeightMeasureSpec) {
    
            // first clears the measured dimension flag
            mPrivateFlags &= ~PFLAG_MEASURED_DIMENSION_SET;
    
            resolveRtlPropertiesIfNeeded();
    
            int cacheIndex = (mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT ? -1 :
                    mMeasureCache.indexOfKey(key);
            if (cacheIndex < 0 || sIgnoreMeasureCache) {
                // measure ourselves, this should set the measured dimension flag back
                // 2.调用了onMeasure方法进行测量，说明View主要的测量逻辑是在该方法中实现
                onMeasure(widthMeasureSpec, heightMeasureSpec);
                mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
            } else {
                long value = mMeasureCache.valueAt(cacheIndex);
                // Casting a long to int drops the high 32 bits, no mask needed
                setMeasuredDimensionRaw((int) (value >> 32), (int) value);
                mPrivateFlags3 |= PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
            }
    
            mPrivateFlags |= PFLAG_LAYOUT_REQUIRED;
        }
        //保存本次视图View的测量规格到mOldWidthMeasureSpec和mOldHeightMeasureSpec以便下次测量条件的判断是否需要重新测量
        mOldWidthMeasureSpec = widthMeasureSpec;
        mOldHeightMeasureSpec = heightMeasureSpec;
        ...
    }

## 1.2 onMeasure

在view的measure过程中会调用view的onMeasure方法，view的measure主要在onMeaure实现。 

    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
                getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
    }
    
    protected final void setMeasuredDimension(int measuredWidth, int measuredHeight) {
        boolean optical = isLayoutModeOptical(this);
        if (optical != isLayoutModeOptical(mParent)) {
            Insets insets = getOpticalInsets();
            int opticalWidth  = insets.left + insets.right;
            int opticalHeight = insets.top  + insets.bottom;
    
            measuredWidth  += optical ? opticalWidth  : -opticalWidth;
            measuredHeight += optical ? opticalHeight : -opticalHeight;
        }
        setMeasuredDimensionRaw(measuredWidth, measuredHeight);
    }
    
    private void setMeasuredDimensionRaw(int measuredWidth, int measuredHeight) {
        mMeasuredWidth = measuredWidth;
        mMeasuredHeight = measuredHeight;
    
        mPrivateFlags |= PFLAG_MEASURED_DIMENSION_SET;
    }


getDefaultSize返回的大小就是MeasureSpec.specSize，这个specSize就是view测量后的大小。这里的多次提到测量大小，是因为view的最终大小是在layout阶段确定的，所以这里必须要加以区分，但几乎所有情况下，测量大小就是最终大小。

    public static int getDefaultSize(int size, int measureSpec) {
       int result = size;
       int specMode = MeasureSpec.getMode(measureSpec);
       int specSize = MeasureSpec.getSize(measureSpec);
    
       switch (specMode) {
       case MeasureSpec.UNSPECIFIED:
           result = size;
           break;
       case MeasureSpec.AT_MOST:
       case MeasureSpec.EXACTLY:
           result = specSize;
           break;
       }
       return result;
    }

对于UNSPECIFIED这种情况，一般用于系统内部的测量过程，这种情况下，view的大小为getSuggestedMinimumWidth和getSuggestedMinimumHeight，这里分析下getSuggestedMinimumWidth的实现，getSuggestedMinimumHeight和它是相同的。

    protected int getSuggestedMinimumWidth() {
        return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinimumWidth());
    }

如果view没有设置背景，那么返回mMinWidth，mMinWidth是android:minWidth这个属性指定的值，这个值可以为0；如果View设置了背景，则返回mMinWidth和背景最小宽度这2者中的最大值。

Tips：从getDefaultSize的方法来看，View的宽高由specSize决定，所以我们可以得出如下的结论：直接继承自 View 的自定义控件需要重写 onMeasure 方法来设置 wrap_content 时候的自身大小，否则在布局中使用了wrap_content，相当于使用match_parent.而设置的具体值需要根据实际情况自己去计算或者直接给定一个默认固定值，否则在布局中使用 wrap_content 时候就相当于使用 match_parent ，因为在布局中使用 wrap_content 的时候，它的 specMode 是 AT_MOST 最大测量模式，在这种模式下 View 的宽/高等于 speceSize 大小，即父容器中可使用的大小，也就是父容器当前剩余全部空间大小，这种情况，很显然，View的宽/高就是等于父容器剩余空间的大小，填充父布局，这种效果和布局中使用 match_parent 一样，解决这个问题代码如下：
    

    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);
        int widthSpecMode = MeasureSpec.getMode(widthMeasureSpec);
        int widthSpecSize = MeasureSpec.getSize(widthMeasureSpec);
        int heightSpecMode = MeasureSpec.getMode(heightMeasureSpec);
        int heightSpecSize = MeasureSpec.getSize(heightMeasureSpec);
      // 在 MeasureSpec.AT_MOST 模式下，给定一个默认值
      //其他情况下沿用系统测量规则即可
        if (widthSpecMode == MeasureSpec.AT_MOST
                && heightSpecMode == MeasureSpec.AT_MOST) {
            setMeasuredDimension(mWith, mHeight);
        } else if (widthSpecMode == MeasureSpec.AT_MOST) {
            setMeasuredDimension(mWith, heightSpecSize);
        } else if (heightSpecMode == MeasureSpec.AT_MOST) {
            setMeasuredDimension(widthSpecSize, mHeight);
        }
    }

## 二、ViewGroup的measure过程

ViewGroup没有重写View的onMeasure的方法，viewGroup是一个抽象类， 其测量过程的onMeasure需要每一个子类去具体实现，不过他的内部提供了一个叫measureChildren的方法
整个过程是先去测量子View的大小，再去测量自己的大小。

    protected void measureChildren(int widthMeasureSpec, int heightMeasureSpec) {
        final int size = mChildrenCount;
        final View[] children = mChildren;
        for (int i = 0; i < size; ++i) {
            final View child = children[i];
            if ((child.mViewFlags & VISIBILITY_MASK) != GONE) {
                // 对每一个子view进行测量
                measureChild(child, widthMeasureSpec, heightMeasureSpec);
            }
        }
    }
    
    protected void measureChild(View child, int parentWidthMeasureSpec,
            int parentHeightMeasureSpec) {
        final LayoutParams lp = child.getLayoutParams();
    
        final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
                mPaddingLeft + mPaddingRight, lp.width);
        final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
                mPaddingTop + mPaddingBottom, lp.height);
    
        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    }

measureChild的思想就是取出子元素的LayoutParams，然后再通过getChildMeasureSpec 获得子元素的MeasureSpec，接着将measureSpec传递给View的measure方法，重复view的measure过程。

LinearLayout的绘制过程，涉及到的重要方法：

    public static int resolveSizeAndState(int size, int measureSpec, int childMeasuredState) {
        final int specMode = MeasureSpec.getMode(measureSpec);
        final int specSize = MeasureSpec.getSize(measureSpec);
        final int result;
        switch (specMode) {
            case MeasureSpec.AT_MOST:
                if (specSize < size) {
                    result = specSize | MEASURED_STATE_TOO_SMALL;
                } else {
                    result = size;
                }
                break;
            case MeasureSpec.EXACTLY:
                result = specSize;
                break;
            case MeasureSpec.UNSPECIFIED:
            default:
                result = size;
        }
        return result | (childMeasuredState & MEASURED_STATE_MASK);
    }

## 三、如何在activity启动时获取view的宽高？

View 的 measure 过程和 Activity 的生命周期方法不是同步执行的，因此无法保证 Activity 执行了onCreate、onStart、onResume 时某个 View 已经测量完毕了。如果View还没有测量完毕，那么获得的宽和高都是 0。下面是四种解决该问题的方法：

## 3.1 Activity/View#onWindowsChanged 方法

onWindowFocusChanged 方法表示 View 已经初始化完毕了，宽高已经准备好了，这个时候去获取是没问题的。这个方法会被调用多次，当 Activity 继续执行或者暂停执行的时候，这个方法都会被调用，典型代码如下：
    

    public void onWindowFocusChanged(boolean hasWindowFocus) {
         super.onWindowFocusChanged(hasWindowFocus);
       if(hasWindowFocus){
       int width=view.getMeasuredWidth();
       int height=view.getMeasuredHeight();
      }      

  }

## 3.2 View.post(runnable)

通过 post 将一个 Runnable 投递到消息队列的尾部，然后等待 Looper 调用此 runnable 的时候 View 也已经初始化好了。

    protected void onStart() {
        super.onStart();
        view.post(new Runnable() {
            @Override
            public void run() {
                int width=view.getMeasuredWidth();
                int height=view.getMeasuredHeight();
            }
        });
    }

## 3.3 ViewTreeObsever

使用 ViewTreeObserver 的众多回调方法可以完成这个功能，比如使用 onGlobalLayoutListener 接口，当 View 树的状态发生改变或者 View 树内部的 View 的可见性发生改变时，onGlobalLayout 方法将被回调。伴随着View树的变化，这个方法也会被多次调用。

    @Override
    protected void onStart() {
        super.onStart();
        ViewTreeObserver viewTreeObserver=view.getViewTreeObserver();
        viewTreeObserver.addOnGlobalLayoutListener(new ViewTreeObserver.OnGlobalLayoutListener() {
            @Override
            public void onGlobalLayout() {
                view.getViewTreeObserver().removeOnGlobalLayoutListener(this);
                int width=view.getMeasuredWidth();
                int height=view.getMeasuredHeight();
            }
        });
    }

## 3.4 view.measure(int widthMeasureSpec, int heightMeasureSpec)

通过手动对 View 进行 measure 来得到 View 的宽高，这个要根据 View 的 LayoutParams 来处理：

（1）match_parent：无法 measure 出具体的宽高，原因是根据上面我们分析 View 的measure 过程原理可知，此种 MeasureSpec 需要知道 parentSize ，即父容器剩余空间，而这个时候无法知道 parentSize 大小，所以无法测量。

（2）具体数值（dp/px)：例如100px，如下 measure :

    int widthMeasureSpec = MeasureSpec.makeMeasureSpec(100, MeasureSpec.EXACTLY);
    int heightMeasureSpec = MeasureSpec.makeMeasureSpec(100, MeasureSpec.EXACTLY);
    view.measure(widthMeasureSpec, heightMeasureSpec);

（3）wrap_content: 可以采用设置最大值方法进 measure ：

    int widthMeasureSpec = MeasureSpec.makeMeasureSpec((1 << 30) - 1, MeasureSpec.AT_MOST);
    int heightMeasureSpec = MeasureSpec.makeMeasureSpec((1 << 30) - 1, MeasureSpec.AT_MOST);
    view.measure(widthMeasureSpec, heightMeasureSpec);

注意这里作者为什么使用 (1 << 30) - 1 ) 来构造 MeasureSpec 呢？笔者解释是：”通过分析 MeasureSpec 的实现可以得知 View 的尺寸是使用 30 位的二进制表示，也就是说最大是 30 个 1 即（2^30-1)，也就是 (1 << 30) - 1 )，在最大化模式下，使用 View 能支持的最大值去构造 MeasureSpec 是合理的“。为什么这样就合理呢？我们前面分析在子 View 使用 wrap_content 模式的时候，其测量规则是根据自身的情况去测量尺寸，但是不能超过父容器的剩余空间的最大值，换句话说就是父容器给子 View 一个最大值，然后告诉子 View 你自己看着办，但是别超过这个尺寸就行，但是现在我们自己去测量的时候不知道父容器给定的 MeasureSpec 情况， 也就是不知道父容器给多大的限定值，需要自己去构造一个MeasureSpec ，那么这个最大值我们给定多少合适呢？所以这里干脆就给一个 View 所能支持的最大值，然子 View 根据自身情况去测量，怎么也不能超过这个值就行了。  

参考：
《Android开发艺术探索》