# CycleViewPager
##前言
目前市场上的APP中，轮播图可以说是很常见的。一个好的轮播图，基本上适用于所有的APP。是时候打造一个自己的轮播图了，不要等到用的时候才取Google。
> 本文参考自[Android实现Banner界面广告图片循环轮播（包括实现手动滑动循环）](http://blog.csdn.net/stevenhu_223/article/details/45577781)，根据该代码改编

##功能
轮播图需要实现一下功能
- 图片循环轮播
- 可添加文字
- 最后一张到第一张的切换也要有切换效果
- 循环、自动播放可控制

还有我们都比较关注的一点：**这轮子必须易拆、易装，可扩展性强**。每次换个项目就要拷贝好几个文件，改一大堆代码，这是很烦的。

##实现
再多的文字也不如一张图来得直观，先来个福利，回头再说怎实现的。

![效果](http://upload-images.jianshu.io/upload_images/1638147-53477b8a3916f5bc.gif?imageMogr2/auto-orient/strip)

####思路
这里使用ViewPager来实现轮播的效果，但是ViewPager是滑动到最后一张时，是不能跳转到第一张的。于是，我们可以这样：
- 需要显示的轮播图有N张
- 往ViewPager中添加N个View，这时ViewPager中有：
View（1）、View（2）、View（3） ... View（N）
- 再往ViewPager中添加View（1），这时ViewPager中有：
View（1）、View（2）、View（3） ... View（N）、View（1）

这样就可以实现一种视觉效果：滑动到最后一张 View（N）的时候，再往后滑动就回到了第一张View（1）。
这也适用于从第一张条转到最后一张的实现。
**文字看着费解？那就看图吧（还好会那么一点点PS）**
**例：**
需要显示三张图：

![需要轮播的图片](http://upload-images.jianshu.io/upload_images/1638147-8354fecf38dc4f32.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
经过处理，变成这样

![处理后的轮播图](http://upload-images.jianshu.io/upload_images/1638147-9f0b7e21bf1e434b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在界面上看到的是三张图片，而实际在ViewPager中的是这样的5张。
- 当从View4跳转到View5时，在代码中立刻将视图切换到View2，应为图片是一样的，所有在界面上看不到任何效果。
- 同理，当从View2跳转到View1时，在代码中将视图切换到View4。

自动轮播流程：
【View2】 -->【View3】 --> 【View4】 --> 【View5 -->View2】（完成一次循环）-->【View3】 -->【View4】....
当显示View5的时候，立刻切换到View2（View5和View2显示的内容是相同的），这样就实现了图片轮播。
这里View5 ->View2的切换巧妙利用了ViewPager中的方法：
```java
setCurrentItem(int item, boolean smoothScroll)
```
参数smoothScroll为false的时候，实现了“看不见”的跳转。

还是不大清楚？那就直接看代码吧

####代码
思路说完，上代码
- **创建model**
这里创建一个Info类，模拟实际应用中的数据。里面有title和url字段。
```java
public class Info {
    private String url;
    private String title;

    public Info(String title, String url) {
        this.url = url;
        this.title = title;
    }

    public String getUrl() {
        return url;
    }

    public void setUrl(String url) {
        this.url = url;
    }

    public String getTitle() {
        return title;
    }

    public void setTitle(String title) {
        this.title = title;
    }
}
```
- **布局**
为了实现画面重叠的效果，这里用了相对布局，轮播图使用ViewPager来实现。后面有两个LinearLayout，第一个LinearLayout用来放指示器，在java代码中动态添加；第二个LinearLayout就用来显示Title了，当然，如果还需要显示的其他内容，可以在这个布局里面中添加。
```xml
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:background="@android:color/white"
    android:orientation="vertical">
    <android.support.v4.view.ViewPager
        android:id="@+id/cycle_view_pager"
        android:layout_width="match_parent"
        android:layout_height="match_parent" />
    <LinearLayout
        android:id="@+id/cycle_indicator"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_alignParentBottom="true"
        android:layout_marginBottom="10dp"
        android:gravity="center"
        android:orientation="horizontal" />
    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_above="@id/cycle_indicator"
        android:orientation="vertical">
        <TextView
            android:id="@+id/cycle_title"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_marginBottom="10dp"
            android:gravity="center"
            android:textColor="@android:color/white"
            android:textSize="20sp" />
    </LinearLayout>
</RelativeLayout>
```
- **CycleViewPager**
重点来了，自定义的轮播图。来个重磅炸弹，别看晕了
```java
public class CycleViewPager extends FrameLayout implements ViewPager.OnPageChangeListener {

    private static final String TAG = "CycleViewPager";

    private Context mContext;

    private ViewPager mViewPager;//实现轮播图的ViewPager

    private TextView mTitle;//标题

    private LinearLayout mIndicatorLayout; // 指示器

    private Handler handler;//每几秒后执行下一张的切换

    private int WHEEL = 100; // 转动

    private int WHEEL_WAIT = 101; // 等待

    private List<View> mViews = new ArrayList<>(); //需要轮播的View，数量为轮播图数量+2

    private ImageView[] mIndicators;    //指示器小圆点

    private boolean isScrolling = false; // 滚动框是否滚动着

    private boolean isCycle = true; // 是否循环，默认为true

    private boolean isWheel = true; // 是否轮播，默认为true

    private int delay = 4000; // 默认轮播时间

    private int mCurrentPosition = 0; // 轮播当前位置

    private long releaseTime = 0; // 手指松开、页面不滚动时间，防止手机松开后短时间进行切换

    private ViewPagerAdapter mAdapter;

    private ImageCycleViewListener mImageCycleViewListener;

    private List<Info> infos;//数据集合

    private int mIndicatorSelected;//指示器图片，被选择状态

    private int mIndicatorUnselected;//指示器图片，未被选择状态

    final Runnable runnable = new Runnable() {
        @Override
        public void run() {
            if (mContext != null && isWheel) {
                long now = System.currentTimeMillis();
                // 检测上一次滑动时间与本次之间是否有触击(手滑动)操作，有的话等待下次轮播
                if (now - releaseTime > delay - 500) {
                    handler.sendEmptyMessage(WHEEL);
                } else {
                    handler.sendEmptyMessage(WHEEL_WAIT);
                }
            }
        }
    };

    public CycleViewPager(Context context) {
        this(context, null);
    }

    public CycleViewPager(Context context, AttributeSet attrs) {
        this(context, attrs, 0);
    }

    public CycleViewPager(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        this.mContext = context;
        initView();
    }

    /**
     * 初始化View
     */
    private void initView() {
      LayoutInflater.from(mContext).inflate(R.layout.layout_cycle_view, this, true);
        mViewPager = (ViewPager) findViewById(R.id.cycle_view_pager);
        mTitle = (TextView) findViewById(R.id.cycle_title);
        mIndicatorLayout = (LinearLayout) findViewById(R.id.cycle_indicator);
        handler = new Handler() {
            @Override
            public void handleMessage(Message msg) {
                super.handleMessage(msg);
                if (msg.what == WHEEL && mViews.size() > 0) {
                    if (!isScrolling) {
                        //当前为非滚动状态，切换到下一页
                        int posttion = (mCurrentPosition + 1) % mViews.size();
                        mViewPager.setCurrentItem(posttion, true);
                    }
                    releaseTime = System.currentTimeMillis();
                    handler.removeCallbacks(runnable);
                    handler.postDelayed(runnable, delay);
                    return;
                }
                if (msg.what == WHEEL_WAIT && mViews.size() > 0) {
                    handler.removeCallbacks(runnable);
                    handler.postDelayed(runnable, delay);
                }
            }
        };
    }

    /**
     * 设置指示器图片，在setData之前调用
     *
     * @param select   选中时的图片
     * @param unselect 未选中时的图片
     */
    public void setIndicators(int select, int unselect) {
        mIndicatorSelected = select;
        mIndicatorUnselected = unselect;
    }
    public void setData(List<Info> list, ImageCycleViewListener listener) {
        setData(list, listener, 0);
    }

    /**
     * 初始化viewpager
     *
     * @param list         要显示的数据
     * @param showPosition 默认显示位置
     */
    public void setData(List<Info> list, ImageCycleViewListener listener, 
        int showPosition) {
        if (list == null || list.size() == 0) {
            //没有数据时隐藏整个布局
            this.setVisibility(View.GONE);
            return;
        }
        mViews.clear();
        infos = list;
        if (isCycle) {
            //添加轮播图View，数量为集合数+2
            // 将最后一个View添加进来
            mViews.add(getImageView(mContext, infos.get(infos.size() - 1).getUrl()));
            for (int i = 0; i < infos.size(); i++) {
                mViews.add(getImageView(mContext, infos.get(i).getUrl()));
            }
            // 将第一个View添加进来
            mViews.add(getImageView(mContext, infos.get(0).getUrl()));
        } else {
            //只添加对应数量的View
            for (int i = 0; i < infos.size(); i++) {
                mViews.add(getImageView(mContext, infos.get(i).getUrl()));
            }
        }
        if (mViews == null || mViews.size() == 0) {
            //没有View时隐藏整个布局
            this.setVisibility(View.GONE);
            return;
        }
        mImageCycleViewListener = listener;
        int ivSize = mViews.size();
        // 设置指示器
        mIndicators = new ImageView[ivSize];
        if (isCycle)
            mIndicators = new ImageView[ivSize - 2];
        mIndicatorLayout.removeAllViews();
        for (int i = 0; i < mIndicators.length; i++) {
            mIndicators[i] = new ImageView(mContext);
            LinearLayout.LayoutParams lp = new LinearLayout.LayoutParams(
                    LinearLayout.LayoutParams.WRAP_CONTENT, 
                    LinearLayout.LayoutParams.WRAP_CONTENT);
            lp.setMargins(10, 0, 10, 0);
            mIndicators[i].setLayoutParams(lp);
            mIndicatorLayout.addView(mIndicators[i]);
        }
        mAdapter = new ViewPagerAdapter();
        // 默认指向第一项，下方viewPager.setCurrentItem将触发重新计算指示器指向
        setIndicator(0);
        mViewPager.setOffscreenPageLimit(3);
        mViewPager.setOnPageChangeListener(this);
        mViewPager.setAdapter(mAdapter);
        if (showPosition < 0 || showPosition >= mViews.size())
            showPosition = 0;
        if (isCycle) {
            showPosition = showPosition + 1;
        }
        mViewPager.setCurrentItem(showPosition);
        setWheel(true);//设置轮播
    }

    /**
     * 获取轮播图View
     *
     * @param context
     * @param url
     */
    private View getImageView(Context context, String url) {
        return MainActivity.getImageView(context, url);
    }

    /**
     * 设置指示器
     *
     * @param selectedPosition 默认指示器位置
     */
    private void setIndicator(int selectedPosition) {
        setText(mTitle, infos.get(selectedPosition).getTitle());
        try {
            for (int i = 0; i < mIndicators.length; i++) {
                mIndicators[i]
                        .setBackgroundResource(mIndicatorUnselected);
            }
            if (mIndicators.length > selectedPosition)
                mIndicators[selectedPosition]
                        .setBackgroundResource(mIndicatorSelected);
        } catch (Exception e) {
            Log.i(TAG, "指示器路径不正确");
        }
    }

    /**
     * 页面适配器 返回对应的view
     *
     * @author Yuedong Li
     */
    private class ViewPagerAdapter extends PagerAdapter {
        @Override
        public int getCount() {
            return mViews.size();
        }
        @Override
        public boolean isViewFromObject(View arg0, Object arg1) {
            return arg0 == arg1;
        }
        @Override
        public void destroyItem(ViewGroup container, int position, Object object) {
            container.removeView((View) object);
        }
        @Override
        public View instantiateItem(ViewGroup container, final int position) {
            View v = mViews.get(position);
            if (mImageCycleViewListener != null) {
                v.setOnClickListener(new OnClickListener() {
                    @Override
                    public void onClick(View v) {
                     mImageCycleViewListener.onImageClick(
                        infos.get(mCurrentPosition - 1), mCurrentPosition, v);
                    }
                });
            }
            container.addView(v);
            return v;
        }
        @Override
        public int getItemPosition(Object object) {
            return POSITION_NONE;
        }
    }
    @Override
    public void onPageScrolled(
          int position, float positionOffset, int positionOffsetPixels) {
    }
    @Override
    public void onPageSelected(int arg0) {
        int max = mViews.size() - 1;
        int position = arg0;
        mCurrentPosition = arg0;
        if (isCycle) {
            if (arg0 == 0) {
                //滚动到mView的1个（界面上的最后一个），将mCurrentPosition设置为max - 1
                mCurrentPosition = max - 1;
            } else if (arg0 == max) {
                //滚动到mView的最后一个（界面上的第一个），将mCurrentPosition设置为1
                mCurrentPosition = 1;
            }
            position = mCurrentPosition - 1;
        }
        setIndicator(position);
    }
    @Override
    public void onPageScrollStateChanged(int state) {
        if (state == 1) { // viewPager在滚动
            isScrolling = true;
            return;
        } else if (state == 0) { // viewPager滚动结束

            releaseTime = System.currentTimeMillis();
            //跳转到第mCurrentPosition个页面（没有动画效果，实际效果页面上没变化）
            mViewPager.setCurrentItem(mCurrentPosition, false);
        }
        isScrolling = false;
    }

    /**
     * 为textview设置文字
     * @param textView
     * @param text
     */
    public static void setText(TextView textView, String text) {
        if (text != null && textView != null) textView.setText(text);
    }

    /**
     * 为textview设置文字
     *
     * @param textView
     * @param text
     */
    public static void setText(TextView textView, int text) {
        if (textView != null) setText(textView, text + "");
    }

    /**
     * 是否循环，默认开启。必须在setData前调用
     *
     * @param isCycle 是否循环
     */
    public void setCycle(boolean isCycle) {
        this.isCycle = isCycle;
    }

    /**
     * 是否处于循环状态
     *
     * @return
     */
    public boolean isCycle() {
        return isCycle;
    }

    /**
     * 设置是否轮播，默认轮播,轮播一定是循环的
     *
     * @param isWheel
     */
    public void setWheel(boolean isWheel) {
        this.isWheel = isWheel;
        isCycle = true;
        if (isWheel) {
            handler.postDelayed(runnable, delay);
        }
    }

    /**
     * 刷新数据，当外部视图更新后，通知刷新数据
     */
    public void refreshData() {
        if (mAdapter != null)
            mAdapter.notifyDataSetChanged();
    }

    /**
     * 是否处于轮播状态
     *
     * @return
     */
    public boolean isWheel() {
        return isWheel;
    }

    /**
     * 设置轮播暂停时间,单位毫秒（默认4000毫秒）
     * @param delay
     */
    public void setDelay(int delay) {
        this.delay = delay;
    }

    /**
     * 轮播控件的监听事件
     *
     * @author minking
     */
    public static interface ImageCycleViewListener {

        /**
         * 单击图片事件
         *
         * @param info
         * @param position
         * @param imageView
         */
        public void onImageClick(Info info, int position, View imageView);
    }
}
```
从里面挑了几个变量和方法说明一下：
**变量**：
handler、runnable：实现定时轮播
mCurrentPosition：表示当前位置
**方法**：
setIndicators()：设置指示器的图片（必须在setData前调用）
setData()：根据数据，生成对应的轮播图
setIndicator()：设置指示器和文字内容
onPageSelected()、onPageScrollStateChanged()：利用ViewPager的滚动监听，实现了上面的思路。onPageSelected()中根据ViewPager中显示的位置，改变mCurrentPosition的值，然后在onPageScrollStateChanged()中根据mCurrentPosition重新设置页面（这里的setCurrentItem没有动画效果）。
getImageView()：根据URL生成Viewpager中对应的各个View（*根据实际的图片加载框架来生成，这里使用了Picasso是想了网络图片的加载*），看看getImageView()中调用的代码
```
    /**
     * 得到轮播图的View
     * @param context
     * @param url
     * @return
     */
    public static View getImageView(Context context, String url) {
        RelativeLayout rl = new RelativeLayout(context);
        //添加一个ImageView，并加载图片
        ImageView imageView = new ImageView(context);
        RelativeLayout.LayoutParams layoutParams = new RelativeLayout.LayoutParams(
                RelativeLayout.LayoutParams.MATCH_PARENT,
                RelativeLayout.LayoutParams.MATCH_PARENT);
        imageView.setScaleType(ImageView.ScaleType.CENTER_CROP);
        imageView.setLayoutParams(layoutParams);
        //使用Picasso来加载图片
        Picasso.with(context).load(url).into(imageView);
        //在Imageview前添加一个半透明的黑色背景，防止文字和图片混在一起
        ImageView backGround = new ImageView(context);
        backGround.setLayoutParams(layoutParams);
        backGround.setBackgroundResource(R.color.cycle_image_bg);
        rl.addView(imageView);
        rl.addView(backGround);
        return rl;
    }
```
```xml
<color name="cycle_image_bg">#44222222</color>
```
代码很简单，创建了一个显示图片的布局，先在布局中添加了需要显示的图片，然后加了个半透明的图，防止显示时文字和图片中白色的部分重叠在一起，导致看不清文字。

- **在Acitivty中使用**
![](http://upload-images.jianshu.io/upload_images/1638147-2b309ce0aa2389a1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
轮子打造好了，不拿出来溜一溜？
```java
public class MainActivity extends AppCompatActivity {

    /**
     * 模拟请求后得到的数据
     */
    List<Info> mList = new ArrayList<>();

    /**
     * 轮播图
     */
    CycleViewPager mCycleViewPager;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        initData();
        initView();
    }

    /**
     * 初始化数据
     */
    private void initData() {
        mList.add(new Info("标题1", 
              "http://img2.3lian.com/2014/c7/25/d/40.jpg"));
        mList.add(new Info("标题2", 
              "http://img2.3lian.com/2014/c7/25/d/41.jpg"));
        mList.add(new Info("标题3", 
             "http://imgsrc.baidu.com/forum/pic/item/b64543a98226cffc8872e00cb9014a90f603ea30.jpg"));
        mList.add(new Info("标题4", 
             "http://imgsrc.baidu.com/forum/pic/item/261bee0a19d8bc3e6db92913828ba61eaad345d4.jpg"));
    }

    /**
     * 初始化View
     */
    private void initView() {
        mCycleViewPager = (CycleViewPager) findViewById(R.id.cycle_view);
        //设置选中和未选中时的图片
        mCycleViewPager.setIndicators(R.mipmap.ad_select, R.mipmap.ad_unselect);
        //设置轮播间隔时间
        mCycleViewPager.setDelay(2000);
        mCycleViewPager.setData(mList, mAdCycleViewListener);
    }

    /**
     * 轮播图点击监听
     */
    private CycleViewPager.ImageCycleViewListener mAdCycleViewListener = 
                  new CycleViewPager.ImageCycleViewListener() {

        @Override
        public void onImageClick(Info info, int position, View imageView) {

            if (mCycleViewPager.isCycle()) {
                position = position - 1;
            }
            Toast.makeText(MainActivity.this, info.getTitle() +
                 "选择了--" + position, Toast.LENGTH_LONG).show();
        }
    };
}
```

使用起来也是很简单的，只要设置下图片、数据、点击监听就可以了。（之前贴过MainActivity.getImageView()方法了，这里就不贴了）
**放到自己的项目中？**
只需要调下布局，根据自己的图片加载框架改下getImageView（或者也可以直接用我的），然后把CycleViewPager中的Info改成自己的Model就可以了。
