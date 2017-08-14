---
title: BRVAH+MTRVA，复杂？不存在的
toc: true
date: 2017-08-14 11:36:09
tags: [Android,RecyclerView,Adapter]
---

## 前言
遥想Android当年，UI出来了，两眼一定，一Bean一XML，谈笑间，设计师瑟瑟发抖。额，不要在意这首尬诗，请忽略- -!物是人非啊，现在动不动掏出个淘宝页面，还条目不固定，还能愉快玩耍吗？再加上杂七杂八的技术加进去，比如说埋点，UI框架越来越沉重，都是泪啊！如果我们能回到过去那该多好，来吧，朋友，这是真的这不是梦。

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
Bean我就不说了哈，跟服务端同志好好沟通，嗯，好好沟通。既然Helper接管了Adapter的数据源和资源，然后再把自己创建的提供给Adapter即可，提供方法在BaseAdapter封装已经介绍过了，那么它是如何创建的呢？

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

最简单的时候，只要这样就行了，这里的工作量就是每一个Adapter可能会多创建一个Helper，这里用可能是因为我们有时候可以复用。在构造函数中注入数据源，当然你也可以像示例代码中传null，它会默认创建一个空集合。我们可以为每个ItemType注册对应xml视图，正如过去每个Activity对应一个xml，当然资源注册功能远远不止这些，比如我们的库把一个ItemType视图，分为header，data，footer三个部分，你可以分别填充不同的资源，单个Type你又可以在data，loading，empty，error这几个视图自如切换，毫无压力。其它还有很多，我就不介绍了，不然篇幅太长，很多手机党要骂娘了。

### 回归Adapter
Helper创建完资源后总是给回归于Adapter，在Base封装中，我们已经知道，Helper是如何和Adapter绑定在一起的。来看下Adapter的示例代码，结合代码更加形象直观。

```
public class MyAdapter extends BaseAdapter<MutiTypeTitleEntity, BaseViewHolder, MyAdapterHelper> {


    private CommonHeadEntity entity1Header = new CommonHeadEntity(ItemEntity1.HEADER_TITLE, ItemEntity1.TYPE_1);
    private CommonHeadEntity entity2Header = new CommonHeadEntity(ItemEntity2.HEADER_TITLE, ItemEntity2.TYPE_2);
    private CommonHeadEntity entity4Header = new CommonHeadEntity(ItemEntity2.HEADER_TITLE, ItemEntity4.TYPE_4);
    private CommonFooterEntity entity1Footer = new CommonFooterEntity(Constants.EXPAND, ItemEntity1.TYPE_1);

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
        helper.setImageResource(R.id.item_1_img, item.getImg());
        helper.setText(R.id.item_1_title, item.getTitle());
        helper.setText(R.id.item_1_content, item.getContent());
        helper.setText(R.id.item_1_time, item.getTime());
        helper.setText(R.id.item_1_time_flag, item.getTimeFlag());
    }

	...

    private void renderHeader(BaseViewHolder helper, CommonHeadEntity item) {
        helper.setText(R.id.title, item.getTitle());
    }

    private void renderFooter(BaseViewHolder helper, final CommonFooterEntity item) {
        final int type = item.getItemType() + RecyclerViewAdapterHelper.FOOTER_TYPE_DIFFER;
        final TextView footer = helper.getView(R.id.item_footer);
        footer.setText(item.getTitle());
        footer.setOnClickListener(v -> {
            if (Constants.EXPAND.equals(item.getTitle())) {
                item.setTitle(Constants.FOLD);
                mHelper.foldType(type, false);
            } else {
                item.setTitle(Constants.EXPAND);
                mHelper.foldType(type, true);
            }
            footer.setText(item.getTitle());
        });

    }
	...
    public void notifyType1(List<ItemEntity1> itemEntity1s) {
        mHelper.notifyMoudleDataAndHeaderAndFooterChanged(entity1Header, itemEntity1s, entity1Footer, ItemEntity1.TYPE_1);
    }
	...
}
```

这么一个复杂视图，就这么点代码，你没有看错。核心方法就是convert()方法，根据ItemType渲染相对应的视图，这正如对应不同的Activity有相对应的xml，Bean和渲染逻辑。关于render()方法，我就不说了吧，估计大家都写吐了。Helper除了支持整个数据源注入外，还支持单个Type注入，甚至细化到单个Type中一个小Type，例如header。简直炫酷的不要不要的。而这里的notifyType1()方法是为了注入type为ItemEntity1.TYPE_1相对应的数据，取这个方法是因为它把所有小类型都体现了。这里有个小tips是，建议大家把刷新的方法封装在Adapter中，万一哪天版本大升级，直接使用helper的朋友可要改死了，这个也同我们在开发中封装图片加载库的需要二次封装一个工具类，万一哪天换库了，你不会想跑路吧？
![](/img/huaji.jpg)

## 结束语
似乎就这么结束了？这么一个复杂界面就这么结束了？逗我？这么简单？	额，老哥，确实就这么多，我不会告诉你我写这个界面只花了kjadadnkdkladllllll，sorry，我整理下发型。老实说，配合BRVAH工作量减少真的太多，说%70真的不为过。如果你不服，你可以用原生的写同样的界面，和用BRVHA+MTRVA计时比较下，我都不想跟你算代码质量问题。甚至你可以用专门针对多类型复杂视图的Adapter库，同样的效果，同样的功能，计时比较下你就知道了。这里插个嘴，要善用include和merge标签，你会有意外收获的。这里我们不是说你敲代码有多快，而是整体的一个效率问题，时间短，质量高，流程简单易懂，还有什么理由不使用一下？

这里感谢下孙老师提供的设计以及一直支持我的人，很感谢。关于库的使用确实就那么点流程，很简单。如果想体验更多的用法，可以在github上看使用指南。

## 传送门
Github：[https://github.com/crazysunj/MultiTypeRecyclerViewAdapter](https://github.com/crazysunj/MultiTypeRecyclerViewAdapter)
