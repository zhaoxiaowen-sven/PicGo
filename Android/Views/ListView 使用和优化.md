ListView作为android 中最常用的控件之一，几乎所有的应用程序都会用到它。ListView的使用关键又在于Adapter的使用，常见的adapter有以下几种：ArrayAdapter，SimpleAdapter，SimpleCursorAdapterListView以及BaseAdapter。虽然BaseAdapter使用起来有些复杂，但是开发中使用最多的还是它。本文主要介绍下BaseAdapter的用法及优化，加载图片时用到了Glide库，完全是为了方便不用太多关心。
##一、基本用法
   1.定义布局文件

   (1)activity的布局 activity_list.xml

       <?xml version="1.0" encoding="utf-8"?>
       <LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
           xmlns:tools="http://schemas.android.com/tools"
           android:layout_width="match_parent"
           android:layout_height="match_parent"
           tools:context="com.sven.variousviews.activities.ListActivity">
       
           <ListView
               android:id="@+id/list_view"
               android:layout_width="match_parent"
               android:layout_height="match_parent" />
       
       </LinearLayout>

   (2)item的布局 fruit_item.xml

       <?xml version="1.0" encoding="utf-8"?>
       <LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
           android:layout_width="match_parent"
           android:layout_height="wrap_content"
           android:layout_gravity="center_vertical"
           android:descendantFocusability="blocksDescendants"
           android:orientation="horizontal">
       
           <ImageView  
               android:id="@+id/iv_pic"
               android:layout_width="100dp"
               android:layout_height="80dp"
               android:layout_marginStart="10dp" />
       
           <RelativeLayout
               android:layout_width="0dp"
               android:layout_height="80dp"
               android:layout_marginStart="10dp"
               android:layout_weight="1">
       
               <TextView
                   android:id="@+id/tv_id"
                   android:layout_width="match_parent"
                   android:layout_height="50dp"
                   android:layout_alignParentStart="true"
                   android:layout_alignParentTop="true"
                   android:gravity="center_vertical" />
       
               <TextView
                   android:id="@+id/tv_name"
                   android:layout_width="match_parent"
                   android:layout_height="30dp"
                   android:layout_alignParentBottom="true"
                   android:layout_alignStart="@+id/tv_id"
                   android:gravity="center_vertical" />
           </RelativeLayout>
       
           <Button
               android:id="@+id/bt_download"
               android:layout_width="100dp"
               android:layout_height="60dp"
               android:layout_gravity="center_vertical"
               android:layout_marginEnd="10dp"
               android:layout_marginStart="10dp"
               style="@style/bt_style"
               />
       </LinearLayout>

   

   2.创建适配器
        
    public class ListAdapter extends BaseAdapter {
     
         private static final String TAG = "ListAdapter";
         private Context mContext;
         private List<FruitBean> mFruitBeanList;
     
         public ListAdapter(Context context, List<FruitBean> list) {
             this.mContext = context;
             this.mFruitBeanList = list;
         }
     
         @Override
         public void notifyDataSetChanged() {
             super.notifyDataSetChanged();
         }
     
         @Override
         public int getCount() {
             return mFruitBeanList.size();
         }
     
         @Override
         public Object getItem(int position) {
             return mFruitBeanList.get(position);
         }
     
         @Override
         public long getItemId(int position) {
             return position;
         }
     
         @Override
         public View getView(final int position, View convertView, ViewGroup parent) {
     
             ViewHolder viewHolder;
             if (convertView == null) {
                 convertView = View.inflate(mContext, R.layout.fruit_item, null);
                 viewHolder = new ViewHolder();
                 viewHolder.tv_id = (TextView) convertView.findViewById(R.id.tv_id);
                 viewHolder.tv_name = (TextView) convertView.findViewById(R.id.tv_name);
                 viewHolder.iv_pic = (ImageView) convertView.findViewById(R.id.iv_pic);
                 viewHolder.bt_download = (Button) convertView.findViewById(R.id.bt_download);
                 convertView.setTag(viewHolder);
             }else{
                 viewHolder = (ViewHolder) convertView.getTag();
             }
     
             final FruitBean fruitBean = mFruitBeanList.get(position);
             if (fruitBean != null) {
                 Glide.with(mContext).load(fruitBean.getUrl()).into(viewHolder.iv_pic);
                 viewHolder.tv_id.setText(fruitBean.getId());
                 viewHolder.tv_name.setText(fruitBean.getName());
                 viewHolder.bt_download.setText("delete");
                 convertView.setOnClickListener(new View.OnClickListener() {
                     @Override
                     public void onClick(View v) {
                         Log.i(TAG, "onClick: convertView clicked");
                     }
                 });
                 
                 viewHolder.bt_download.setOnClickListener(new View.OnClickListener() {
                     @Override
                     public void onClick(View v) {
                         Log.i(TAG, "onClick: bt_download");
     //                    Toast.makeText(mContext, "onclick bt_down", Toast.LENGTH_SHORT).show();
                         mFruitBeanList.remove(position);
     //                    mFruitBeanList.add(fruitBean);
                         notifyDataSetChanged();
                     }
                 });
     
                 viewHolder.tv_name.setOnClickListener(new View.OnClickListener() {
                     @Override
                     public void onClick(View v) {
                         Toast.makeText(mContext, "onclick tv_name " + fruitBean.getId(), Toast.LENGTH_SHORT).show();
                     }
                 });
             }
             return convertView;
         }
     
         static class ViewHolder {
             public ImageView iv_pic;
             public TextView tv_id;
             public TextView tv_name;
             public Button bt_download;
         }
     }

   3.调用

    public class ListActivity extends AppCompatActivity {
    
        private static final String TAG = "ListActivity2";
        private ListView listView;


​    
        @Override
        protected void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            setContentView(R.layout.activity_list);
    
            listView = (ListView) findViewById(R.id.list_view);
            ListAdapter adapter = new ListAdapter(ListActivity.this, Utils.initData());
    //        listView.setOnItemClickListener(new AdapterView.OnItemClickListener() {
    //            @Override
    //            public void onItemClick(AdapterView<?> parent, View view, int position, long id) {
    //                Log.i(TAG, "onItemClick: " + position);
    //                Toast.makeText(ListActivity.this, "onItemClick: " + position, Toast.LENGTH_SHORT).show();
    //            }
    //        });
            listView.setAdapter(adapter);
        }

##二、优化
对ListView进行优化一般有2种方式   
1.convertView：
通过缓存convertView,这种利用缓存contentView的方式可以判断如果缓存中不存在View才创建View，如果已经存在可以利用缓存中的View，提升性能。
    
2.viewHolder：
通过convertView+ViewHolder来实现，ViewHolder就是一个静态类，使用ViewHolder的关键好处是缓存了显示数据的视图（View），加快了 UI 的响应速度。当我们判断 convertView == null  的时候，如果为空，就会根据设计好的List的Item布局（XML），来为convertView赋值，并生成一个viewHolder来绑定convertView里面的各个View控件（XML布局里面的那些控件）。再用convertView的setTag将viewHolder设置到Tag中，以便系统第二次绘制ListView时从Tag中取出。
    
##三、事件监听
为ListView的条目设置监听有2种方式：
1.通过listView.setOnItemClickListener，注意这种方式可能会出现点击事件冲突的情况，比如你的item里包含Button，解决方法有三种：
    (1)将ListView中的Item布局中的子控件focusable属性设置为false
    (2)在getView方法中设置button.setFocusable(false)
    (3)设置item的根布局的属性Android:descendantFocusability="blocksDescendant"
    
2.在adapter中为convertView设置监听setOnClickListener

##四、实现多种样式的ListView布局样式
    getItemViewType和getViewTypeCount
    

##五、添加HeaderView 和 FooterView
添加header时调用的 addHeaderView方法必须放在listview.setadapter前面，和Android的版本有关，不会再低版本有兼容问题。
    
    headerView = LayoutInflater.from(this).inflate(R.layout.list_header_view, null);
    footerView = LayoutInflater.from(this).inflate(R.layout.list_footer_view, null);
    headerButton = (Button) headerView.findViewById(R.id.header_bt);
    headerButton.setOnClickListener(new View.OnClickListener() {
        @Override
        public void onClick(View v) {
            Log.i(TAG, "onClick: header_bt");
            Toast.makeText(ListActivity.this, "headerButton clicked" , Toast.LENGTH_SHORT).show();
        }
    });
    footerButton = footerView.findViewById(R.id.footer_bt);
    footerButton.setOnClickListener(new View.OnClickListener() {
        @Override
        public void onClick(View v) {
            Log.i(TAG, "onClick: footer_bt");
            Toast.makeText(ListActivity.this, "footerButton clicked", Toast.LENGTH_SHORT).show();
        }
    });
    listView.addHeaderView(headerView, null, false);
    listView.addFooterView(footerView);
    listView.setAdapter(adapter);

##六、进阶使用：
    
1.滑动优化加载图片

2.局部更新

   未完待续，尽请期待！

参考资料：
   《第一行代码》 郭霖
   http://blog.csdn.net/zhaokaiqiang1992/article/details/28430607
   http://www.cnblogs.com/RGogoing/p/5872217.html
   http://892848153.iteye.com/blog/1923680
Demo：https://github.com/zhaoxiaowen-sven/VariousViews.git