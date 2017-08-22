---
title: BRVAH+MTRVA，复杂？不存在的
toc: true
date: 2017-08-14 11:36:09
tags: [Android,RecyclerView,Adapter]
---

## 前言
遥想Android当年，UI出来了，两眼一定，一Bean一XML，谈笑间，设计师瑟瑟发抖。额，不要在意这首尬诗，请忽略- -!物是人非啊，现在动不动掏出个淘宝页面，还条目不固定，还能愉快玩耍吗？再加上各种其它需求，比如说埋点，初始化展示Loading页面，错误时又要切换错误页面等，UI框架越来越沉重，都是泪啊！如果我们能回到过去那该多好，来吧，朋友，这是真的这不是梦。

![](/img/mocamoca.jpg)

## 实战
就算是以前，很多基础工作还是要做的，XML你总要写吧？Bean你总要写吧？渲染逻辑逃不掉吧？那么，今天过后，额。。。这些你还是要写，但是其他的非业务逻辑你可以不写。首先，我们来看看这样的首页效果。

### 效果图展示
![](/img/adapterHelper10.gif)

<!--  more-->
复杂不失简单，简单又不失内涵，这就是我朋友孙老师的伟大杰作，而你看到的是我用程序把它展现出来了，孙老师，哪里不满意，你尽管说。

![](/img/maobing.gif)

### 基础介绍
言归正传，这样的一个首页，我们需要做怎么样的基础工作呢？或者说，碰到以后更复杂的页面我们应该怎么做？这里小提示下，不要再用什么类似ScrollView的这种东西了，诶，好像说的有点绝对，尽量不要用，这不是谷歌想要看到的，5.0谷歌推出了RecyclerView，从它的整个设计架构来看，简直就是为这而生的。而RecyclerView的视图是通过Adapter来渲染的。原始的Adapter，让人很蛋疼，重复工作太多，我们应该要有封装的思想，把最需要的部分提供出来，其它不用管。

Adapter最火的库我想是BRVAH了，怎么个简单法呢？

```
public class QuickAdapter extends BaseQuickAdapter<Status, BaseViewHolder> {
    public QuickAdapter() {
        super(R.layout.tweet, DataServer.getSampleData());
    }

    @Override
    protected void convert(BaseViewHolder viewHolder, Status item) {
        viewHolder.setText(R.id.tweetName, item.getUserName())
                .setText(R.id.tweetText, item.getText())
                .setText(R.id.tweetDate, item.getCreatedAt())
                .setVisible(R.id.tweetRT, item.isRetweet())
                .linkify(R.id.tweetText);
                 Glide.with(mContext).load(item.getUserAvatar()).crossFade().into((ImageView) viewHolder.getView(R.id.iv));
    }
}
```

这样就行了，具体戳[这里](http://www.jianshu.com/p/b343fcff51b0)。

该库也提供了渲染复杂视图的方法，但用起来还是有一定麻烦。我个人觉得不是不写，而是没必要写，分工明确就行了，侵入性太强不太好。那么MTRVA就诞生了，大家不要再纠结这个名字了。

![](/img/baobaoku.jpg)

MTRVA最初是因为多类型视图又继承了Adapter才这样命名的，后来领略了继承的局限性，改成了与Adapter组合的形式呈现，叫AdapterHelper更为合适点，因为它可以配合绝大多数的Adapter。处理多类型视图是最初的一个想法，也是现在的一个功能点而已，其实它内部是接管了Adapter的资源和数据源，让我们的数据处理更加方便，快捷，不用再去考虑资源和数据源的问题。说再多也没用哈，实战演练走起。

	Tips:全文，甚至库的demo都是以BRVAH为配合对象。

### BaseAdapter封装
既然要配合，那么总是要加上无畏的代码量，放心，我都考虑到了。放上我们简单基础的BaseAdapter，当然你可以根据自己的项目加入其它。

```
public abstract class BaseAdapter<T extends MutiTypeTitleEntity, K extends BaseViewHolder, H extends RecyclerViewAdapterHelper<T, BaseAdapter>> extends BaseQuickAdapter<T, K> {

    protected H mHelper;

    public BaseAdapter(H helper) {
        super(helper.getData());
        mHelper = helper;
        mHelper.bindAdapter(this);
    }

    @Override
    protected K onCreateDefViewHolder(ViewGroup parent, int viewType) {
        return createBaseViewHolder(parent, mHelper.getLayoutId(viewType));
    }

    @Override
    protected int getDefItemViewType(int position) {
        return mHelper.getItemViewType(position);
    }

    public H getHelper() {
        return mHelper;
    }
}
```

在构造方法中，让Helper去绑定Adapter，并把自己的数据源还给Adapter，在onCreateDefViewHolder方法中，把Helper注册的资源还给Adapter，ItemViewType同理。到这里，BaseAdapter封装就结束了，并没有什么难度，代码量也不大。

### XML和Bean
Bean我就不说了哈，跟服务端同志好好沟通，嗯，好好沟通。

现在我要展示没有使用我们库的时候xml的布局，前方高能，注意安全！

```
<android.support.v4.widget.NestedScrollView
    android:id="@+id/scrollView"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    app:layout_behavior="@string/appbar_scrolling_view_behavior">

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="vertical">

        <android.support.v7.widget.RecyclerView
            android:id="@+id/head_list"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:background="@color/color_white" />

        <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="vertical">
            ...
            省略40行
            ...
        </LinearLayout>

        <android.support.v7.widget.RecyclerView
            android:id="@+id/type1_list"
            android:layout_width="match_parent"
            android:layout_height="wrap_content" />

        <TextView
            ...
            省略10行
            .../>

        <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="vertical">
            ...
            省略40行
            ...
        </LinearLayout>

        <android.support.v7.widget.RecyclerView
            android:id="@+id/type2_list"
            android:layout_width="match_parent"
            android:layout_height="wrap_content" />

        <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="vertical">
            ...
            省略100行
            ...
        </LinearLayout>

        <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="vertical">
            ...
            省略40行
            ...
        </LinearLayout>

        <android.support.v7.widget.RecyclerView
            android:id="@+id/type3_list"
            android:layout_width="match_parent"
            android:layout_height="wrap_content" />

        <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="vertical">

            <LinearLayout
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:orientation="vertical">
                ...
                省略40行
                ...
            </LinearLayout>

            <android.support.design.widget.TabLayout
                android:id="@+id/footer_tab"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                app:tabBackground="@color/color_white"
                app:tabIndicatorColor="@color/colorPrimary"
                app:tabSelectedTextColor="@color/colorPrimary"
                app:tabTextColor="@color/color_333333" />

            <android.support.v4.view.ViewPager
                android:id="@+id/footer_pager"
                android:layout_width="match_parent"
                android:layout_height="135dp" />

            <TextView
                style="@style/text_14_F7657D"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:gravity="center"
                android:lineSpacingExtra="6dp"
                android:padding="10dp"
                android:text="@string/copyright" />

        </LinearLayout>

    </LinearLayout>
</android.support.v4.widget.NestedScrollView>
```

全部看完的的同学，我给你82分，剩下的18分以666的形式给你。如果有想看省略部分的朋友，我直接给跪(demo中有)。哥们，如果你接手了这样的代码，我真的很心疼你。这里给点意见，赶紧用include和merge标签，让xml层次更清晰点。玩过这样的布局的同学，肯定搜了不少文章，如何嵌套？如何不卡顿？等等。不多BB，如果使用该库，那么将会是这样：

```
<android.support.v7.widget.RecyclerView
        android:id="@+id/recyclerview"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        app:layout_behavior="@string/appbar_scrolling_view_behavior" />
```
忍住，朋友，我知道你想说卧槽，我可以告诉你个好消息，从本质上，xml量是不会变的，不然怎么展示一样的视图？但是，后者更趋向于模块化开发的思想，可以分配具体某个type到某个人，就算换了个人，也能很快的接手，不仅如此，开发效率大大地提高，怎么说？每人到手的就是一个HelloWord渲染在Activity里面，这个都不会？高端点说，View层和Modle层分开了，Helper就是我们的P层，甚至我们可以忽略Model层的逻辑，下文会简单提到。而前者需要自己去封装，谷歌看见这样的场景会流泪的，关于使用RecyclerView的好处，只有用了才知道。

朋友，我还想再贴Activity里面一堆初始化和渲染的代码。

![](/img/chouju.jpeg)

一堆关于一会儿显示这一会儿显示那，一会儿又不显示，诸如此类的代码都让人看得头疼。想想这样布局，来个全局loading的需求，Boom!不敢多说了，恐怕有小伙伴已经拿起了菜刀，我好好做人，请放下！

继续今天的头条，话说Helper是接管了Adapter的数据源和资源，然后再把自己创建的提供给Adapter即可，提供方法在BaseAdapter封装已经介绍过了，那么它是如何创建的呢？

![](/img/lizi.jpeg)

```
public class MyAdapterHelper extends AsynAdapterHelper<MutiTypeTitleEntity, BaseAdapter> {


    public MyAdapterHelper() {
        super(null);
    }

    @Override
    protected void registerMoudle() {

		...
        registerMoudle(ItemEntity3.TYPE_3)
                .level(2)
                .layoutResId(R.layout.item_3)
                .register();
		...
    }
}
```

最简单的时候，只要这样就行了，这里的工作量就是每一个Adapter可能会多创建一个Helper，这里用可能是因为我们有时候可以复用。在构造函数中注入数据源，当然你也可以像示例代码中传null，它会默认创建一个空集合。我们可以为每个ItemType注册对应xml视图，正如过去每个Activity对应一个xml，当然资源注册功能远远不止这些，比如我们的库把一个ItemType视图，分为header，data，footer三个部分，你可以分别填充不同的资源，单个Type你又可以在data，loading，empty，error这几个视图自如切换，毫无压力，如果你用嵌套这种布局，会不会加班到天明啊。。。行行行，放下，我不说了。关于库的其它功能还有很多，我就不介绍了，不然篇幅太长，很多手机党要骂娘了。

### 回归Adapter
Helper创建完资源后总是要回归于Adapter，在BaseAdapter封装中，我们已经知道，Helper是如何和Adapter绑定在一起的。来看下Adapter的示例代码，结合代码更加形象直观。

```
public class MyAdapter extends BaseAdapter<MutiTypeTitleEntity, BaseViewHolder, MyAdapterHelper> {

    public MyAdapter() {
        super(new MyAdapterHelper());
    }

    @Override
    protected void convert(BaseViewHolder helper, MutiTypeTitleEntity item) {
        int itemType = item.getItemType();
        switch (itemType) {
            case ItemEntity1.TYPE_1:
                renderEntity1(helper, (ItemEntity1) item);
                break;
          	...
            case ItemEntity1.TYPE_1 - RecyclerViewAdapterHelper.HEADER_TYPE_DIFFER:
            case ItemEntity2.TYPE_2 - RecyclerViewAdapterHelper.HEADER_TYPE_DIFFER:
            case ItemEntity4.TYPE_4 - RecyclerViewAdapterHelper.HEADER_TYPE_DIFFER:
                renderHeader(helper, (CommonHeadEntity) item);
                break;
            case ItemEntity1.TYPE_1 - RecyclerViewAdapterHelper.FOOTER_TYPE_DIFFER:
                renderFooter(helper, (CommonFooterEntity) item);
                break;
        }
    }

    private void renderEntity1(BaseViewHolder helper, ItemEntity1 item) {
    	...
    	渲染Entity1
    	...
    }

	...

    private void renderHeader(BaseViewHolder helper, CommonHeadEntity item) {
    	渲染header
    }

    private void renderFooter(BaseViewHolder helper, final CommonFooterEntity item) {
    	...
    	渲染footer
    	...
    }
	...
    public void notifyType1(List<ItemEntity1> itemEntity1s) {
        mHelper.notifyMoudleDataAndHeaderAndFooterChanged(entity1Header, itemEntity1s, entity1Footer, ItemEntity1.TYPE_1);
    }
	...
}
```

这么一个复杂视图，就这么点代码，你没有看错。核心方法就是convert()方法，根据ItemType渲染相对应的视图，这正如对应不同的Activity有相对应的xml，Bean和渲染逻辑，在这里你只要关心View层的渲染逻辑就行了。关于render()方法，我就不说了吧，估计大家都写吐了。Helper除了支持整个数据源注入外，还支持单个Type注入，甚至细化到单个Type中一个小Type，例如header。简直炫酷的不要不要的。而这里的notifyType1()方法是为了注入type为ItemEntity1.TYPE_1相对应的数据，取这个方法是因为所有类型都体现了(header,data,footer)。

	Tips:建议大家把刷新的方法封装在Adapter中，万一哪天版本大升级，直接使用helper的朋友可要改死了，这个也同我们在开发中封装图片加载库的需要二次封装一个工具类，万一哪天换库了，你不会想跑路吧？

![](/img/huaji.jpg)

## 原理分析
似乎就这么结束了？这么一个复杂界面就这么结束了？逗我？这么简单？	额，老哥，确实就这么多，我不会告诉你我写这个界面只花了kjadadnkdkladllllll，sorry，我整理下发型。老实说，配合BRVAH工作量减少真的太多，说%70真的不为过。如果你不服，你可以用原生的写同样的界面，和用BRVHA+MTRVA计时比较下，我都不想跟你算代码质量问题。甚至你可以用专门针对多类型复杂视图的Adapter库，同样的效果，同样的功能，计时比较下你就知道了。这里插个嘴，要善用include和merge标签，你会有意外收获的。这里我们不是说你敲代码有多快，而是整体的一个效率问题，时间短，质量高，流程简单易懂，还有什么理由不使用一下？

到这里，使用确实结束了，而且很简单，那原理呢？

本库的差量刷新的核心是DiffUtil，那么我们从这里切入。

### DiffUtil
DiffUtil内部采用的Eugene W. Myers’s difference 算法，通过传入新老数据集，计算二者间的差异，再调用相应的方法进行刷新：

```
adapter.notifyItemRangeInserted(position, count);
adapter.notifyItemRangeRemoved(position, count);
adapter.notifyItemMoved(fromPosition, toPosition);
adapter.notifyItemRangeChanged(position, count, payload);
```
这样一来，我们可以跟notifyDataSetChanged说88啦。那么新老数据源怎么来？

### 数据
以下type新老数据集，简称数据集，整个数据集，简称数据源。

老数据源好说，不就是adapter里面的数据源嘛。新数据源呢？如果是整一个数据集，直接拿来用就是了，可我现在只想更新数据源里面的某个type的数据集，最好有个以type为key的map集合管理着每个type的数据集，当我们要更新数据集的时候，可以直接掏出老数据集。I have a old type list，I have a new type list，new type list！

到这里，我想了两个办法。其一，更新map集合中需要更新的type的value为新数据集，然后再遍历组合成新数据源。其二，copy一份老数据源，先移除老数据集，再添加新数据集。这里先不分析孰优孰劣，我选择了后者。

无论是第一种还是第二种都面临一个问题，数据集的位置。假设现在我们有3种type，不妨为A，B，C，产品要求顺序为A，B，C（如果B数据集为空，那么就为A，C，以此类推），但你现在可能出现C，B，A，老铁，这不是打篮球，恐怕你得被开除啊。这种优先级的概念，我们见得太多了，比如进程的优先级，有序广播的优先级等等。所以这里我们引入了level的概念，且把map的key改为level，举个比较明显问题的栗子，遍历map的时候，你还是不知道谁前谁后。那你会说，我根据level找到type，再根据type找到数据集不就好了？很不幸，我们这里，level跟type是一对多的关系，比如上面说的A，它可能用来显示正常的数据，万一产品说如果数据出错，我们需要有错误页面（错误页面级别是type），那岂不是GG？

map的key改为level后，两种方法的实现思路很明显了。这里贴出第二种的代码。

```
mNewData.addAll(mData);//copy老数据源

mNewData.removeAll(oldItemData);
mNewData.remove(oldItemHeader);//移除老数据集

mNewData.addAll(header == null ? positionStart : positionStart + 1, data);
mNewData.add(positionStart, newHeader);
mNewData.add(positionStart, footer);//添加新数据集      
```

细心的同学可能发现，positionStart怎么来的？它是靠map遍历得到的，但它是不完全遍历。

到这里，新老数据源都有了，剩下的就是交给diffutil去更新UI。

```
DiffUtil.DiffResult result = DiffUtil.calculateDiff(getDiffCallBack(mData, mNewData), isDetectMoves());
...
diffResult.dispatchUpdatesTo(getListUpdateCallback(mAdapter));
 
mData.addAll(mNewData);//切记，切记，一定要保持内外数据源一致     
```
这里我们看到getDiffCallBack(mData, mNewData)，isDetectMoves()，getListUpdateCallback(mAdapter)。先来说说isDetectMoves()方法，返回值作为DiffUtil静态方法calculateDiff的第二个参数，这个参数官方是这么说的。

```
If move detection is enabled, it takes an additional O(N^2) time where N is the total number of added and removed items. If your lists are already sorted by the same constraint (e.g. a created timestamp for a list of posts), you can disable move detection to improve performance.
```

大概意思是说，如果该参数为true，那么在计算的时候，会额外增加O(N^2) 的时间复杂度，N为移动的数量（增加和删除），如果列表已经按约束设计了（不需要调整），建议填false。可能这么说比较抽象，官方也给出了测试数据（调试数据为一组随机UUID字符串，运行在Android版本型号为M的Nexus 5X），我们来看看吧。

```
100项中10项修改：平均值：0.39毫秒，中位数：0.35毫秒
100项中100项修改：平均值：3.82毫秒，中位数：3.75毫秒
100个项目中100个修改（不移动）：平均值：2.09毫秒，中位数：2.06毫秒
1000项中50项修改：平均值：4.67毫秒，中位数：4.59毫秒。
1000个项目中50个修改（不移动）：平均值：3.59毫秒，中位数：3.50毫秒
1000项中200项修改：平均值：27.07毫秒，中位数：26.92毫秒
1000个项目中200个修改（不移动）：平均值：13.54毫秒，中位数：13.36毫秒
```

接下来我们来看看getListUpdateCallback方法，这个比较好理解，它作为dispatchUpdatesTo方法的参数。返回值ListUpdateCallback是对计算数据的回调。我们来看看库的默认实现。

```
protected ListUpdateCallback getListUpdateCallback(final RecyclerView.Adapter adapter) {

    final int preDataCount = getPreDataCount();

    return new ListUpdateCallback() {
        @Override
        public void onInserted(int position, int count) {
            adapter.notifyItemRangeInserted(position + preDataCount, count);
        }

        @Override
        public void onRemoved(int position, int count) {
            adapter.notifyItemRangeRemoved(position + preDataCount, count);
        }

        @Override
        public void onMoved(int fromPosition, int toPosition) {
            adapter.notifyItemMoved(fromPosition + preDataCount, toPosition + preDataCount);
        }

        @Override
        public void onChanged(int position, int count, Object payload) {
            adapter.notifyItemRangeChanged(position + preDataCount, count, payload);
        }
    };
}
```

比官方的默认实现多了preDataCount变量，该变量是为了保证内部数据position与adapter的position一致。同学们可以通过重写getPreDataCount方法改变值。

最后一个getDiffCallBack方法，这个较为复杂，但是熟悉了也还好，我这里简单介绍一下，感兴趣的可以到官方文档看，官方是最权威的，小弟的英文也不太好，所以。。。那个。。。言归正传，该方法的返回值为抽象类DiffUtil.Callback，共有5个方法。

```
public abstract int getOldListSize();

public abstract int getNewListSize();

public abstract boolean areItemsTheSame(int oldItemPosition, int newItemPosition);

public abstract boolean areContentsTheSame(int oldItemPosition, int newItemPosition);

@Nullable
public Object getChangePayload(int oldItemPosition, int newItemPosition) {
    return null;
}
```
前面两个我就不说了，见名知意，中间2个，其实也很明显，第三个看名字是说判断2个条目是否相同的，恭喜你答对了，这个地方我们一般判断两个条目的“主键”，如果返回true才会调areContentsTheSame方法，看名字就是让我们判断条目中的内容是否一样，可以判断其中一项，也可以判断多项，甚至全部。最后一个方法getChangePayload，是配合Adapter中

```
public void onBindViewHolder(VH holder, int position, List<Object> payloads)
```
Android源码中该方法是调两个参数的方法，那么第三个参数怎么来的呢？我们回上去看看getListUpdateCallback方法，里面有这么一个方法

```
@Override
public void onChanged(int position, int count, Object payload) {
    adapter.notifyItemRangeChanged(position + preDataCount, count, payload);
}
```

卧槽，我懂了，是的，就是这么一回事。那它有什么用呢？比如像这样，

```
@Nullable
@Override
public Object getChangePayload(int oldItemPosition, int newItemPosition) {
	Bundle payload = new Bundle();
	payload.putInt("MONEY", 998);
	return payload;
}
```

最后在adapter回调方法onBindViewHolder中取出Bundle，根据Bundle来局部更新，不用全部走一遍。

好了，这里栗子用的Bundle，大家可以看到数据类型其实是Object，之所以用Bundle是因为我们要跟随谷歌爸爸的脚步，Bundle在Android数据通信这一块作用还是很大的。

    Tips：如果想用第五个方法带来的好处，要版本23以上哦，因为adapter的3参数notifyItemRangeChanged是在23上加的。那是不是说23以下，我们库不能用了？细心的同学肯定会去查diffutil在什么版本出现的，不妨告诉你，24.2.0出现的，我靠，玩个毛线。放心，我这里有兼容方案，只要你用RecyclerView，那么，朋友，就放心用这个库吧，没毛病！关于兼容方案，这里就不给出了。使用说明书上有。

### 异步

上面result与diffResult不一致是我采用了两个方法，原因是DiffUtil造成的。

```
If the lists are large, this operation may take significant time so you are advised to run this on a background thread, get the DiffUtil.DiffResult then apply it on the RecyclerView on the main thread.
```

大概意思说，数据量太大的时候，计算最好放到子线程，计算结束再到主线程更新UI。没毛病，因此拆成2个方法，如果是异步，则先调用前者，再切到主线程调用后者。但这其实是线程不安全的，与Android引入Handler更新UI有点类似（不理解的同学，可以去看看Handler的相关文章）。这里我们选择了串行的方法并引入了以单链表结构的队列来管理每次刷新的数据源。

```
HandleBase<T> pollData = mRefreshQueue.poll();
if (pollData != null) {
    startRefresh(pollData);
} else {
    onEnd();
}
```

我们这里没有Looper的概念，因为我知道它什么时候开始，什么时候结束。

以上就是本库的核心原理啦，其它还有像什么资源管理（链式注册），数据的创建，模式的切换，生命周期的回调等。感兴趣的同学可以看看源码。

## 结束语
这里感谢下孙老师提供的设计以及一直支持我的人，很感谢。关于库的使用确实就那么点流程，很简单。如果想体验更多的用法，可以在github上看使用说明书。如果使用和原理都写下来，手机党要开枪了！嘿嘿。

![](/img/biekaiqiang.jpeg)

## 传送门
Github：[https://github.com/crazysunj/MultiTypeRecyclerViewAdapter](https://github.com/crazysunj/MultiTypeRecyclerViewAdapter)
