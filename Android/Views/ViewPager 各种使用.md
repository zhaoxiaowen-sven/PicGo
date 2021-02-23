ViewPager作为广泛使用的控件，本文主要介绍他的各种使用方式。

## 1. viewPager 基本使用
   1.导入依赖
   2.新建Adapter文件继承自PagerAdapter
   3.添加view
1.1 布局中加入viewpager

    <android.support.v4.view.ViewPager
        android:id="@+id/vp2"
        android:layout_weight="1"
        android:layout_width="match_parent"
        android:layout_height="0dp" />

1.2 添加view 创建Adapter

    class MyPagerAdapter extends PagerAdapter {
        private List<View> mViewList;
        public MyPagerAdapter(List<View> viewList) {
            this.mViewList = viewList;
        }
        @Override
        public int getCount() {
            return mViewList == null ? 0 : mViewList.size();
        }
        @Override
        public boolean isViewFromObject(View view, Object object) {
            return view == object;
        }
        @Override
        public Object instantiateItem(ViewGroup container, int position) {
            container.addView(mViewList.get(position));
            return mViewList.get(position);
        }
        @Override
        public void destroyItem(ViewGroup container, int position, Object object) {
            container.removeView(mViewList.get(position));
        }
    }

1.3 调用

    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_vp2);
        mViewPager = (ViewPager) findViewById(R.id.vp2);
        bindViews();
    }
    private void bindViews() {
        LayoutInflater layoutInflater = getLayoutInflater();
        view1 = layoutInflater.inflate(R.layout.page1, null);
        view2 = layoutInflater.inflate(R.layout.page2, null);
        view3 = layoutInflater.inflate(R.layout.page3, null);
        List<View> viewList = new ArrayList<>();
        viewList = new ArrayList<>();
        viewList.add(view1);
        viewList.add(view2);
        viewList.add(view3);
    }

## 2. ViewPager标题栏，PagerTitleStrip和PagerTabStrip
2.1 将它作为子控件添加在ViewPager中

    <android.support.v4.view.ViewPager
        android:id="@+id/vp2"
        android:layout_weight="1"
        android:layout_width="match_parent"
        android:layout_height="0dp">
        <android.support.v4.view.PagerTabStrip
            android:id="@+id/vp_strip"
            android:layout_gravity="top"
            android:layout_width="match_parent"
            android:layout_height="wrap_content">
        </android.support.v4.view.PagerTabStrip>
    </android.support.v4.view.ViewPager>

2.2 重写适配器的getPageTitle(int)函数来获取标题

    mPagerTabStrip = (PagerTabStrip) findViewById(R.id.vp_strip);
    
    class MyPagerAdapter extends PagerAdapter {
    ...
        @Override
        public CharSequence getPageTitle(int position) {
            return mTitleList.get(position);
        }
    }

2.3 区别：
    1.PagerTabStrip在当前页面下，会有一个下划线条来提示当前页面的Tab是哪个。
    2.PagerTabStrip的Tab是可以点击的，当用户点击某一个Tab时，当前页面就会跳转到这个页面，而PagerTitleStrip则没这个功能。   

## 3.ViewPager + Fragment
### 3.1 Fragment的滑动切换

    fragmentList.add(new Fragment1());
    fragmentList.add(new Fragment2());
    fragmentList.add(new Fragment3());
    mViewPager.setAdapter(new FragPagerAdapter(getSupportFragmentManager(), fragmentList));
    for (int i = 0; i < titleList.size(); i++){
        mTabLayout.addTab(mTabLayout.newTab().setText(titleList.get(i)));
    }

### 3.2 FragmentPagerAdapter 和 FragmentStatePagerAdapter

    class FragPagerAdapter extends FragmentStatePagerAdapter {
        private List<Fragment> mFragmentList;
        public FragPagerAdapter(FragmentManager fm , List<Fragment> fragmentList){
            super(fm);
            this.mFragmentList = fragmentList;
        }
        @Override
        public Fragment getItem(int position) {
            return mFragmentList.get(position);
        }
        @Override
        public int getCount() {
            return mFragmentList.size();
        }
    }

使用 FragmentPagerAdapter 时，ViewPager 中的所有 Fragment 实例常驻内存，当 Fragment 变得不可见时仅仅是视图结构的销毁，即调用了 onDestroyView 方法。由于 FragmentPagerAdapter 内存消耗较大，所以适合少量静态页面的场景。

使用 FragmentStatePagerAdapter 时，当 Fragment 变得不可见，不仅视图层次销毁，实例也被销毁，即调用了 onDestroyView 和 onDestroy 方法，仅仅保存 Fragment 状态。相比而言， FragmentStatePagerAdapter 内存占用较小，所以适合大量动态页面，比如我们常见的新闻列表类应用。
http://yifeng.studio/2016/12/23/android-fragment-and-viewpager-attentions/

## 4. ViewPager和Tablayout联动
viewPager 和 tablayout关联的3种方式：
### 4.1.addOnTabSelectedListener + addOnPageChangeListener

    mTabLayout.addOnTabSelectedListener(new TabLayout.OnTabSelectedListener() {
            @Override
            public void onTabSelected(TabLayout.Tab tab) {
                mViewPager.setCurrentItem(tab.getPosition());
            }
            @Override
            public void onTabUnselected(TabLayout.Tab tab) {
            }
            @Override
            public void onTabReselected(TabLayout.Tab tab) {
            }
        });
    
    mViewPager.addOnPageChangeListener(new ViewPager.OnPageChangeListener() {
            @Override
            public void onPageScrolled(int position, float positionOffset, int positionOffsetPixels) {
            }
            @Override
            public void onPageSelected(int position) {
                mTabLayout.getTabAt(position).select();
            }
            @Override
            public void onPageScrollStateChanged(int state) {
            }
        });

### 4.2 setupWithViewPager + 复写 getPageTitle
通过setupWithViewPager将TabLayout和ViewPager建立关联，会发现之前设置的tab消失了，需要复写VpAdapter的getPageTitle

    mTabLayout.setupWithViewPager(mViewPager);
    @Override
    public CharSequence getPageTitle(int position) {
       return mTitleList.get(position);
    }

https://www.jianshu.com/p/d83f10c1a765

### 4.3 addOnPageChangeListener + 复写addOnTabSelectedListener

    mViewPager.addOnPageChangeListener(new TabLayout.TabLayoutOnPageChangeListener(mTabLayout));
    mTabLayout.addOnTabSelectedListener(new TabLayout.OnTabSelectedListener() {
        @Override
        public void onTabSelected(TabLayout.Tab tab) {
            mViewPager.setCurrentItem(tab.getPosition());
        }
        @Override
        public void onTabUnselected(TabLayout.Tab tab) {
        }
        @Override
        public void onTabReselected(TabLayout.Tab tab) {
        }
    });

## 5. ViewPager 轮播图
有2种方法：
5.1 Integer.MAX_VALUE ???
再在初始化时设置当前页面为几千页(如：ViewPager.setCurrentItem(1000*data.size)),其实就是障眼法，大爷心情好的话，向前滑动几千页也不是不可能的;

5.2 滑动到最左或最右时，设置滑动的index 为
   1，2，3 -> 3, 1, 2, 3, 1
    
    class MyPagerAdapter extends PagerAdapter {
        private List<View> mViewList;
        public MyPagerAdapter(List<View> viewList) {
            this.mViewList = viewList;
        }
        @Override
        public int getCount() {
            return mViewList == null ? 0 : mViewList.size();
        }
        @Override
        public boolean isViewFromObject(View view, Object object) {
            return view == object;
        }
        @Override
        public Object instantiateItem(ViewGroup container, int position) {
            View view = mViewList.get(position);
            container.addView(view);
            return view;
        }
        @Override
        public void destroyItem(ViewGroup container, int position, Object object) {
            container.removeView(mViewList.get(position));
        }
    }

5.3 自动循环,触摸停止
通过handler postdelay实现：
    private Runnable autoScrollRunnable = new Runnable() {
    
        @Override
        public void run() {
            if (!stopAutoScroll) {
                int currentItem = viewPager.getCurrentItem();
                viewPager.setCurrentItem(currentItem + 1, true);
                handler.postDelayed(autoScrollRunnable, 4000);
            }
        }
    };


    handler.postDelayed(autoScrollRunnable, 4000);
    viewPager.setOnTouchListener(new View.OnTouchListener() {
    
       @Override
       public boolean onTouch(View v, MotionEvent event) {
           switch (event.getAction()) {
           case MotionEvent.ACTION_DOWN:
               stopAutoScroll = true;
               handler.removeCallbacks(autoScrollRunnable);
               break;
    
           case MotionEvent.ACTION_UP:
           case MotionEvent.ACTION_CANCEL:
               stopAutoScroll = false;
               handler.postDelayed(autoScrollRunnable, 4000);
               break;
           }
           return false;
       }
    })


3.自动循环

遇到的问题：
java.lang.IllegalStateException: The specified child already has a parent. You must call removeView() on the child's parent first.
解决：http://blog.csdn.net/qibanxuehua/article/details/47253299

## 6.切换动画


http://blog.csdn.net/shedoor/article/details/78957852
Q:
The application's PagerAdapter changed the adapter's contents without calling PagerAdapter#notifyDataSetChanged!

mTabLayout.setupWithViewPager 和 mViewPager.addOnPageChangeListener