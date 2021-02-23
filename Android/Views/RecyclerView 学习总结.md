RecyclerView 作为增强版的ListView，RecycleView可以实现listView 和GridView的所有功能。本文主要讲解RecyclerView一些基本用法。
    
##一、基本用法
  1.导入依赖，app build.gradle 文件中导入依赖

    compile 'com.android.support:recyclerview-v7:25.3.1'

  2.布局文件 item文件和上面的ListView相同

      <?xml version="1.0" encoding="utf-8"?>
      <LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
          android:layout_width="match_parent"
          android:layout_height="match_parent">
          <android.support.v7.widget.RecyclerView
              android:id="@+id/recycler_view"
              android:layout_width="match_parent"
              android:layout_height="match_parent"/>
      </LinearLayout>

  3.适配器
    1.继承RecyclerView.Adapter
    2.创建adapter的内部类ViewHolder继承RecyclerView.ViewHolder
    3.复写onCreateViewHolder和onBindViewHolder
         onCreateViewHolder 创建viewHolder实例
         onBindViewHolder 滑动时到屏幕里时绑定viewHolder的数据  
       
    public class RecyclerAdapter extends RecyclerView.Adapter<RecyclerAdapter.ViewHolder> {
        private static final String TAG = "RecyclerAdapter";
        private List<FruitBean> mFruitBeanList;
        private Context mContext;
        private RecyclerItemClickListener mItemListener;
        private boolean isLoadImage = true;
    
        public RecyclerAdapter(List<FruitBean> fruitBeanList, Context context) {
            this.mFruitBeanList = fruitBeanList;
            this.mContext = context;
        }
    
        public void setLoadImage(boolean loadImage) {
            isLoadImage = loadImage;
        }
    
        public void setItemListener(RecyclerItemClickListener itemListener) {
            this.mItemListener = itemListener;
        }
    
        @Override
        public ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
    //        Log.i(TAG, "onCreateViewHolder: ");
            View view = LayoutInflater.from(parent.getContext()).inflate(R.layout.fruit_item, parent, false);
            return new ViewHolder(view);
        }
    
        @Override
        public void onBindViewHolder(final ViewHolder holder, final int position) {
    //        Log.i(TAG, "onBindViewHolder: ");
            FruitBean fruitBean = mFruitBeanList.get(position);
            Log.i(TAG, "onBindViewHolder: " + isLoadImage);
            if (isLoadImage) {
                Glide.with(mContext).load(fruitBean.getUrl()).into(holder.iv_pic);
            }
            holder.tv_id.setText(fruitBean.getId());
            holder.tv_name.setText(fruitBean.getName());
            holder.bt_download.setText("recycler");
        }
    
        @Override
        public int getItemCount() {
            return mFruitBeanList.size();
        }
    
        static class ViewHolder extends RecyclerView.ViewHolder {
    
            ImageView iv_pic;
            TextView tv_id;
            TextView tv_name;
            Button bt_download;
    
            public ViewHolder(View itemView) {
                super(itemView);
                iv_pic = (ImageView) itemView.findViewById(R.id.iv_pic);
                tv_id = (TextView) itemView.findViewById(R.id.tv_id);
                tv_name = (TextView) itemView.findViewById(R.id.tv_name);
                bt_download = (Button) itemView.findViewById(R.id.bt_download);
            }
        }
    }

  4.调用显示

    public class RecyclerActivity extends AppCompatActivity {
    
        private static final String TAG = "RecyclerActivity";
    
        @Override
        protected void onCreate(@Nullable Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            setContentView(R.layout.activity_recycler);
            RecyclerView recyclerView = (RecyclerView) findViewById(R.id.recycler_view);
            LinearLayoutManager layoutManager = new LinearLayoutManager(this);
            recyclerView.setLayoutManager(layoutManager);
            final RecyclerAdapter adapter = new RecyclerAdapter(Utils.initData(), this);
            recyclerView.setAdapter(adapter);
        }
    }

##二、为itemView添加事件监听
### 1.在viewHolder.itemview setonClickListener添加回调
###(1)添加一个监听的接口，可以添加到Adapter中

    public interface RecyclerItemClickListener {
        void onItemClickListener(View view, int position);
        void onItemLongClickListener(View view, int position);
    }

###(2)在onBindViewHolder 方法中，为viewHolder.itemview添加监听时，调用自定义的方法

    public void onBindViewHolder(final ViewHolder holder, final int position) {
        ...
        if (mItemListener != null) {
            Log.i(TAG, "onBindViewHolder: here");
            holder.itemView.setOnClickListener(new View.OnClickListener() {
                @Override
                public void onClick(View v) {
                    mItemListener.onItemClickListener(holder.itemView, position);
                }
            });
            holder.itemView.setOnLongClickListener(new View.OnLongClickListener() {
                @Override
                public boolean onLongClick(View v) {
                    mItemListener.onItemLongClickListener(holder.itemView, position);
                    return true;
                }
            });
        }
    }

### (3)Adapter中添加一个对接口的引用，并且在调用时传入接口的方法

    adapter.setItemListener(new RecyclerAdapter.RecyclerItemClickListener() {
      @Override
      public void onItemClickListener(View view, int position) {
          Log.i(TAG, "onItemClickListener: RecyclerItemClickListener");
      }
      
      @Override
      public void onItemLongClickListener(View view, int position) {
          Log.i(TAG, "onItemLongClickListener: RecyclerItemClickListener");
      }
    });
这样就可以在传入时控制接口的操作。

### 2.采用官方提供的addOnItemTouchListener
这种方式涉及到3个类：RecyclerView.OnItemTouchListener，GestureDetector（GestureDetectorCompat兼容），SimpleOnGestureListener，通过一个手势探测器 GestureDetectorCompat来探测屏幕事件，然后通过手势监听器SimpleOnGestureListener来识别手势事件的种类，然后调用我们设置的对应的回调方法。这里值得说的是：当获取到了RecyclerView的点击事件和触摸事件数据MotionEvent，那么如何才能知道点击的是哪一个item呢？RecyclerView已经为我们提供了这样的方法：在onInterceptTouchEvent中通过findChildViewUnder()获取对应的view。我们可以通过这个方法获得点击的 item ，同时我们调用 RecyclerView的另一个方法getChildViewHolder()，可以获得该item的ViewHolder，最后再回调我们定义的虚方法 onItemClick() 就ok了，这样我们就可以在外部实现该方法来获得 item 的点击事件了。

    public abstract class RecyclerTouchListener implements RecyclerView.OnItemTouchListener{
        private View childView;
        private RecyclerView touchView;
        private GestureDetector mGestureDetector;
    
        public RecyclerTouchListener(Context context){
            mGestureDetector = new GestureDetector(context, new InternalGestureListener());
        }
    
        @Override
        public boolean onInterceptTouchEvent(RecyclerView rv, MotionEvent e) {
            childView = rv.findChildViewUnder(e.getX(), e.getY());
            touchView = rv;
            mGestureDetector.onTouchEvent(e);
            return false;
        }
    
        @Override
        public void onTouchEvent(RecyclerView rv, MotionEvent e) {
    
        }
    
        @Override
        public void onRequestDisallowInterceptTouchEvent(boolean disallowIntercept) {
    
        }
    
        public abstract void onItemClickListener(RecyclerView.ViewHolder viewHolder);
        public abstract void onItemLongClickListener(RecyclerView.ViewHolder viewHolder);
    
        class InternalGestureListener extends GestureDetector.SimpleOnGestureListener{
            @Override
            public boolean onSingleTapUp(MotionEvent e) {
                if (childView != null) {
                    onItemClickListener(touchView.getChildViewHolder(childView));
                }
                return true;
            }
    
            @Override
            public void onLongPress(MotionEvent e) {
                if (childView != null) {
                    onItemLongClickListener(touchView.getChildViewHolder(childView));
                }
            }
        }
    }   

调用

    recyclerView.addOnItemTouchListener(new RecyclerTouchListener(this) {
         @Override
         public void onItemClickListener(RecyclerView.ViewHolder viewHolder) {
             Log.i(TAG, "addOnItemTouchListener: onItemClickListener");
         }
         @Override
         public void onItemLongClickListener(RecyclerView.ViewHolder viewHolder) {
             Log.i(TAG, "addOnItemTouchListener: onItemLongClickListener");
         }
     });

 

第一种相对第二种来说，更加简便，实现起来也方便，也比较好理解。第二种还能用于更加复杂的手势监听，我们可以利用 GestureDetector 类来实现更加复杂的事件监听回调，而第一种监听的事件比较有限。
    
## 三、拖曳排序和滑动删除
主要涉及到2个类：ItemTouchHelper 和 ItemTouchHelper.Callback 
1.自定义一个接口ItemTouchHelperAdapter用来操作Adapter中的数据
    
    public interface ItemTouchHelperAdapter {
        //数据交换
        void onItemMove(int fromPosition,int toPosition);
        //数据删除
        void onItemDismiss(int position);
    }

2.adapter 实现该接口ItemTouchHelperAdapter，用于处理数据

    @Override
    public void onItemMove(int fromPosition, int toPosition) {
        //交换位置
        Collections.swap(mFruitBeanList,fromPosition,toPosition);
        notifyItemMoved(fromPosition,toPosition);
    }
    
    @Override
    public void onItemDismiss(int position) {
        mFruitBeanList.remove(position);
        notifyItemRemoved(position);
    }

3.构建需要的ItemTouchHelper.Callback

    public class SimpleItemTouchHelperCallback extends ItemTouchHelper.Callback{
        private ItemTouchHelperAdapter mAdapter;
    
        public SimpleItemTouchHelperCallback(ItemTouchHelperAdapter adapter){
            this.mAdapter = adapter;
        }
    
        @Override
        public int getMovementFlags(RecyclerView recyclerView, RecyclerView.ViewHolder viewHolder) {
            int dragFlags = ItemTouchHelper.UP | ItemTouchHelper.DOWN;
            int swipeFlags = ItemTouchHelper.LEFT;
            return makeMovementFlags(dragFlags,swipeFlags);
        }
    
        @Override
        public boolean isLongPressDragEnabled() {
            return true;
        }
    
        @Override
        public boolean isItemViewSwipeEnabled() {
            return true;
        }
    
        @Override
        public boolean onMove(RecyclerView recyclerView, RecyclerView.ViewHolder viewHolder, RecyclerView.ViewHolder target) {
            mAdapter.onItemMove(viewHolder.getAdapterPosition(),target.getAdapterPosition());
            return true;
        }
    
        @Override
        public void onSwiped(RecyclerView.ViewHolder viewHolder, int direction) {
            mAdapter.onItemDismiss(viewHolder.getAdapterPosition());
        }
    }

4.调用

    //先实例化Callback
     ItemTouchHelper.Callback callback = new SimpleItemTouchHelperCallback(adapter);
     //用Callback构造ItemtouchHelper
     ItemTouchHelper touchHelper = new ItemTouchHelper(callback);
     //调用ItemTouchHelper的attachToRecyclerView方法建立联系
     touchHelper.attachToRecyclerView(recyclerView);



## 四、网格布局和瀑布流

    GridLayoutManager gridLayoutManager = new GridLayoutManager(this,3);
    StaggeredGridLayoutManager layoutManager = new StaggeredGridLayoutManager(3, StaggeredGridLayoutManager.VERTICAL);

More：
1.分割线addItemDecoration
2.删除添加的动画 ItemAnimator
3.HeaderView 和 FooterView
http://blog.csdn.net/lmj623565791/article/details/51854533

参考资料：
http://www.jianshu.com/p/70788a7a5547
http://blog.csdn.net/lmj623565791/article/details/45059587
http://blog.csdn.net/lmj623565791/article/details/51854533

Demo：https://github.com/zhaoxiaowen-sven/VariousViews.git