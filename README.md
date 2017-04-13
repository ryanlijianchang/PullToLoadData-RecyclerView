*心灵鸡汤：知之者不如好之者，好之者不如乐之者。*

# 摘要 #

一直在用到RecyclerView时都会微微一颤，因为一直都没去了解怎么实现上拉加载，受够了每次去Github找开源引入，因为感觉就为了一个上拉加载功能而去引入一大堆你不知道有多少BUG的代码，不仅增加了项目的冗余程度，而且出现BUG的时候，你却发现很难去改，正因为这样，我就下定决心去了解如何来实现RecyclerView的上拉加载功能，相信大家和我有过同样的情况，但是我相信，只要你给自己几分钟看完这篇文章，你就会发现实现一个上拉加载是非常的简单。

# 什么是上拉加载 #

上拉加载和下拉刷新相对应，在Android API LEVEL 19(即4.4)之后，Google官方推出了SwipeRefreshLayout和RecyclerView的共同使用，为我们提供了更加便捷的列表下拉刷新功能，但是，并没有给我们提供上拉加载功能，但是在RecyclerView强大的可扩展之下，Github上面有了很多开源项目实现了上拉加载功能，即我们不会一次性将所有数据加载到列表中，当用户滑动到底部时，再向服务器请求数据，再填充数据到列表中，这样不仅可以有更好的人机交互，同时在减少了服务器的压力的同时也对客户端的性能有了更好的提升。本篇文章主要通过介绍实现以下简单的上拉加载功能，同学们可以在掌握了最基本的实现功能之后，再通过扩展和优化，甚至可以封装成比较通用的代码，开源到Github上面。

![Demo](http://onq81n53u.bkt.clouddn.com/ezgif.com-video-to-gif%20%288%29.gif)

# 实现思路 #

## 一、XML的实现 ##

布局很简单，只有一个SwipeRefreshLayout包裹了一个RecyclerView，相信用过RecyclerView的都很容易看懂。如下为activity_main.xml：


```
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <android.support.v4.widget.SwipeRefreshLayout
        android:id="@+id/refreshLayout"
        android:layout_width="match_parent"
        android:layout_height="match_parent">

        <android.support.v7.widget.RecyclerView
            android:id="@+id/recyclerView"
            android:layout_width="match_parent"
            android:layout_height="match_parent"/>

    </android.support.v4.widget.SwipeRefreshLayout>

</LinearLayout>

```

然后，我们RecyclerView的Item布局也是非常简单，只有一个TextView。如下为item.xml：


```
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:orientation="vertical">

    <TextView
        android:id="@+id/tv"
        android:layout_width="match_parent"
        android:layout_height="120dp"
        android:background="@android:color/holo_blue_dark"
        android:gravity="center"
        android:textSize="30sp"
        android:textColor="#ffffff"
        android:text="11"
        android:layout_marginBottom="1dp"/>

</LinearLayout>

```

看到我们效果图都知道，在我们上拉时，还有一个提示的条目，我定义为 footview.xml：


```
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:orientation="vertical">

    <TextView
        android:id="@+id/tips"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:gravity="center"
        android:padding="30dp"
        android:textSize="15sp"
        android:layout_marginBottom="1dp"/>

</LinearLayout>

```

## 二、初始化SwipeRefreshLayout ##

在准备好了布局文件之后，我们就把目光转到Activity中去，首先我们需要初始化SwipeRefreshLayout，初始化也是很简单，这里省去了findView操作，所以就只有设置转动的颜色，还有设置刷新的监听事件：


```
private void initRefreshLayout() {
    refreshLayout.setColorSchemeResources(android.R.color.holo_blue_light, android.R.color.holo_red_light,
            android.R.color.holo_orange_light, android.R.color.holo_green_light);
    refreshLayout.setOnRefreshListener(this);
}

@Override
public void onRefresh() {
    // 设置可见
    refreshLayout.setRefreshing(true);
    // 重置adapter的数据源为空
    adapter.resetDatas();
    // 获取第第0条到第PAGE_COUNT（值为10）条的数据
    updateRecyclerView(0, PAGE_COUNT);
    mHandler.postDelayed(new Runnable() {
        @Override
        public void run() {
            // 模拟网络加载时间，设置不可见
            refreshLayout.setRefreshing(false);
        }
    }, 1000);
}

```

## 三、定义RecyclerView的Adapter ##


```
public class MyAdapter extends RecyclerView.Adapter<RecyclerView.ViewHolder> {
    private List<String> datas; // 数据源
    private Context context;    // 上下文Context
    
    private int normalType = 0;     // 第一种ViewType，正常的item
    private int footType = 1;       // 第二种ViewType，底部的提示View
    
    private boolean hasMore = true;   // 变量，是否有更多数据
    private boolean fadeTips = false; // 变量，是否隐藏了底部的提示
    
    private Handler mHandler = new Handler(Looper.getMainLooper()); //获取主线程的Handler

    public MyAdapter(List<String> datas, Context context, boolean hasMore) {
        // 初始化变量
        this.datas = datas;
        this.context = context;
        this.hasMore = hasMore;
    }
    
    // 获取条目数量，之所以要加1是因为增加了一条footView
    @Override
    public int getItemCount() {
        return datas.size() + 1;
    }
    
    // 自定义方法，获取列表中数据源的最后一个位置，比getItemCount少1，因为不计上footView
    public int getRealLastPosition() {
        return datas.size();
    }


    // 根据条目位置返回ViewType，以供onCreateViewHolder方法内获取不同的Holder
    @Override
    public int getItemViewType(int position) {
        if (position == getItemCount() - 1) {
            return footType;
        } else {
            return normalType;
        }
    }
    
    // 正常item的ViewHolder，用以缓存findView操作
    class NormalHolder extends RecyclerView.ViewHolder {
        private TextView textView;

        public NormalHolder(View itemView) {
            super(itemView);
            textView = (TextView) itemView.findViewById(R.id.tv);
        }
    }

    // // 底部footView的ViewHolder，用以缓存findView操作
    class FootHolder extends RecyclerView.ViewHolder {
        private TextView tips;

        public FootHolder(View itemView) {
            super(itemView);
            tips = (TextView) itemView.findViewById(R.id.tips);
        }
    }
    
    @Override
    public RecyclerView.ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
        // 根据返回的ViewType，绑定不同的布局文件，这里只有两种
        if (viewType == normalType) {
            return new NormalHolder(LayoutInflater.from(context).inflate(R.layout.item, null));
        } else {
            return new FootHolder(LayoutInflater.from(context).inflate(R.layout.footview, null));
        }
    }

    @Override
    public void onBindViewHolder(final RecyclerView.ViewHolder holder, int position) {
        // 如果是正常的imte，直接设置TextView的值
        if (holder instanceof NormalHolder) {
            ((NormalHolder) holder).textView.setText(datas.get(position));
        } else {
            // 之所以要设置可见，是因为我在没有更多数据时会隐藏了这个footView
            ((FootHolder) holder).tips.setVisibility(View.VISIBLE);
            // 只有获取数据为空时，hasMore为false，所以当我们拉到底部时基本都会首先显示“正在加载更多...”
            if (hasMore == true) {
                // 不隐藏footView提示
                fadeTips = false;
                if (datas.size() > 0) {
                    // 如果查询数据发现增加之后，就显示正在加载更多
                    ((FootHolder) holder).tips.setText("正在加载更多...");
                }
            } else {
                if (datas.size() > 0) {
                    // 如果查询数据发现并没有增加时，就显示没有更多数据了
                    ((FootHolder) holder).tips.setText("没有更多数据了");
                    
                    // 然后通过延时加载模拟网络请求的时间，在500ms后执行
                    mHandler.postDelayed(new Runnable() {
                        @Override
                        public void run() {
                            // 隐藏提示条
                            ((FootHolder) holder).tips.setVisibility(View.GONE);
                            // 将fadeTips设置true
                            fadeTips = true;
                            // hasMore设为true是为了让再次拉到底时，会先显示正在加载更多
                            hasMore = true;
                        }
                    }, 500);
                }
            }
        }
    }

    // 暴露接口，改变fadeTips的方法
    public boolean isFadeTips() {
        return fadeTips;
    }

    // 暴露接口，下拉刷新时，通过暴露方法将数据源置为空
    public void resetDatas() {
        datas = new ArrayList<>();
    }
    
    // 暴露接口，更新数据源，并修改hasMore的值，如果有增加数据，hasMore为true，否则为false
    public void updateList(List<String> newDatas, boolean hasMore) {
        // 在原有的数据之上增加新数据
        if (newDatas != null) {
            datas.addAll(newDatas);
        }
        this.hasMore = hasMore;
        notifyDataSetChanged();
    }

}

```


## 四、初始化RecyclerView ##


```
private void initRecyclerView() {
    // 初始化RecyclerView的Adapter
    // 第一个参数为数据，上拉加载的原理就是分页，所以我设置常量PAGE_COUNT=10，即每次加载10个数据
    // 第二个参数为Context
    // 第三个参数为hasMore，是否有新数据
    adapter = new MyAdapter(getDatas(0, PAGE_COUNT), this, getDatas(0, PAGE_COUNT).size() > 0 ? true : false);
    mLayoutManager = new GridLayoutManager(this, 1);
    recyclerView.setLayoutManager(mLayoutManager);
    recyclerView.setAdapter(adapter);
    recyclerView.setItemAnimator(new DefaultItemAnimator());

    // 实现上拉加载重要步骤，设置滑动监听器，RecyclerView自带的ScrollListener
    recyclerView.addOnScrollListener(new RecyclerView.OnScrollListener() {
        @Override
        public void onScrollStateChanged(RecyclerView recyclerView, int newState) {
            super.onScrollStateChanged(recyclerView, newState);
            // 在newState为滑到底部时
            if (newState == RecyclerView.SCROLL_STATE_IDLE) {
                // 如果没有隐藏footView，那么最后一个条目的位置就比我们的getItemCount少1，自己可以算一下
                if (adapter.isFadeTips() == false && lastVisibleItem + 1 == adapter.getItemCount()) {
                    mHandler.postDelayed(new Runnable() {
                        @Override
                        public void run() {
                            // 然后调用updateRecyclerview方法更新RecyclerView
                            updateRecyclerView(adapter.getRealLastPosition(), adapter.getRealLastPosition() + PAGE_COUNT);
                        }
                    }, 500);
                }
                
                // 如果隐藏了提示条，我们又上拉加载时，那么最后一个条目就要比getItemCount要少2
                if (adapter.isFadeTips() == true && lastVisibleItem + 2 == adapter.getItemCount()) {
                    mHandler.postDelayed(new Runnable() {
                        @Override
                        public void run() {
                            // 然后调用updateRecyclerview方法更新RecyclerView
                            updateRecyclerView(adapter.getRealLastPosition(), adapter.getRealLastPosition() + PAGE_COUNT);
                        }
                    }, 500);
                }
            }
        }

        @Override
        public void onScrolled(RecyclerView recyclerView, int dx, int dy) {
            super.onScrolled(recyclerView, dx, dy);
            // 在滑动完成后，拿到最后一个可见的item的位置
            lastVisibleItem = mLayoutManager.findLastVisibleItemPosition();
        }
    });
}

// 上拉加载时调用的更新RecyclerView的方法
private void updateRecyclerView(int fromIndex, int toIndex) {
    // 获取从fromIndex到toIndex的数据
    List<String> newDatas = getDatas(fromIndex, toIndex);
    if (newDatas.size() > 0) {
        // 然后传给Adapter，并设置hasMore为true
        adapter.updateList(newDatas, true);
    } else {
        adapter.updateList(null, false);
    }
}
```

所以，Activity的完整代码如下：


```
public class MainActivity extends AppCompatActivity implements SwipeRefreshLayout.OnRefreshListener {
    private SwipeRefreshLayout refreshLayout;
    private RecyclerView recyclerView;
    private List<String> list;

    private int lastVisibleItem = 0;
    private final int PAGE_COUNT = 10;
    private GridLayoutManager mLayoutManager;
    private MyAdapter adapter;
    private Handler mHandler = new Handler(Looper.getMainLooper());

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        initData();
        findView();
        initRefreshLayout();
        initRecyclerView();
    }

    private void initData() {
        list = new ArrayList<>();
        for (int i = 1; i <= 40; i++) {
            list.add("条目" + i);
        }
    }


    private void findView() {
        refreshLayout = (SwipeRefreshLayout) findViewById(R.id.refreshLayout);
        recyclerView = (RecyclerView) findViewById(R.id.recyclerView);

    }

    private void initRefreshLayout() {
        refreshLayout.setColorSchemeResources(android.R.color.holo_blue_light, android.R.color.holo_red_light,
                android.R.color.holo_orange_light, android.R.color.holo_green_light);
        refreshLayout.setOnRefreshListener(this);
    }

    private void initRecyclerView() {
        adapter = new MyAdapter(getDatas(0, PAGE_COUNT), this, getDatas(0, PAGE_COUNT).size() > 0 ? true : false);
        mLayoutManager = new GridLayoutManager(this, 1);
        recyclerView.setLayoutManager(mLayoutManager);
        recyclerView.setAdapter(adapter);
        recyclerView.setItemAnimator(new DefaultItemAnimator());

        recyclerView.addOnScrollListener(new RecyclerView.OnScrollListener() {
            @Override
            public void onScrollStateChanged(RecyclerView recyclerView, int newState) {
                super.onScrollStateChanged(recyclerView, newState);
                if (newState == RecyclerView.SCROLL_STATE_IDLE) {
                    if (adapter.isFadeTips() == false && lastVisibleItem + 1 == adapter.getItemCount()) {
                        mHandler.postDelayed(new Runnable() {
                            @Override
                            public void run() {
                                updateRecyclerView(adapter.getRealLastPosition(), adapter.getRealLastPosition() + PAGE_COUNT);
                            }
                        }, 500);
                    }

                    if (adapter.isFadeTips() == true && lastVisibleItem + 2 == adapter.getItemCount()) {
                        mHandler.postDelayed(new Runnable() {
                            @Override
                            public void run() {
                                updateRecyclerView(adapter.getRealLastPosition(), adapter.getRealLastPosition() + PAGE_COUNT);
                            }
                        }, 500);
                    }
                }
            }

            @Override
            public void onScrolled(RecyclerView recyclerView, int dx, int dy) {
                super.onScrolled(recyclerView, dx, dy);
                lastVisibleItem = mLayoutManager.findLastVisibleItemPosition();
            }
        });
    }

    private List<String> getDatas(final int firstIndex, final int lastIndex) {
        List<String> resList = new ArrayList<>();
        for (int i = firstIndex; i < lastIndex; i++) {
            if (i < list.size()) {
                resList.add(list.get(i));
            }
        }
        return resList;
    }

    private void updateRecyclerView(int fromIndex, int toIndex) {
        List<String> newDatas = getDatas(fromIndex, toIndex);
        if (newDatas.size() > 0) {
            adapter.updateList(newDatas, true);
        } else {
            adapter.updateList(null, false);
        }
    }

    @Override
    public void onRefresh() {
        refreshLayout.setRefreshing(true);
        adapter.resetDatas();
        updateRecyclerView(0, PAGE_COUNT);
        mHandler.postDelayed(new Runnable() {
            @Override
            public void run() {
                refreshLayout.setRefreshing(false);
            }
        }, 1000);
    }
}
```

# 后话 #

以上代码我是考虑到了更多的边界条件，所以在代码上会稍微多了一点，但是也不影响观看。大家也可以通过改变数据源的数量和PAGE_COUNT等来测试，每个人在具体使用上都会有不同的要求，所以基本代码我摆了出来，众口难调，更多的细节需要大家来优化，例如footView可以设置一个动画条，下拉刷新用其他样式替换原生的样式等，我想，这些对于学习完这篇文章的你来说，都会是简单的问题了。

# Demo下载 #

Github下载：[PullToLoadData-RecyclerView](https://github.com/ryanlijianchang/PullToLoadData-RecyclerView)

CSDN资源：[PullToLoadData-RecyclerView](http://download.csdn.net/detail/ljcitworld/9813262)

