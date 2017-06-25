---
title: 一行代码实现ViewPager卡片效果
toc: true
date: 2017-06-25 20:34:12
tags: [ViewPager,CardView,Card,PageTransformer]
---

## 前言
最近看到越来越多ViewPager卡片效果，甚至自己公司的产品也用到了。正如自己看到这个效果时，内心的想法是，这个简单，github一搜一箩筐，看了不下4个库，使用起来都比较麻烦，不是说写得不好，都是这方面的先驱者，值得学习。关键是在这三天一小需求，一周一大需求的年代里，不一行代码搞定，怎么完成任务啊，更何况这体现不了我们优秀程序员的逼格啊！开个玩笑，哈哈！我封装了常见的卡片效果，达到一行代码就能使用。No picture，say a J8！全程采用图文并茂的形式分析，帮助你理解。

<!--  more-->

## 使用方法

```
viewPager.bind(getSupportFragmentManager(), new MyCardHandler(), Arrays.asList(imageArray));
```

其中ViewPager是我们自定义的，名曰CardViewPager。可能比较纠结的就是MyCardHandler。

```
public class MyCardHandler implements CardHandler<String> {
    @Override
    public View onBind(final Context context, final String data, final int position) {
        View view = View.inflate(context, R.layout.item, null);
        ImageView imageView = (ImageView) view.findViewById(R.id.image);
        Glide.with(context).load(data).into(imageView);
        view.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Toast.makeText(context, "data:" + data + "position:" + position, Toast.LENGTH_SHORT).show();
            }
        });
        return view;
    }
}


public interface CardHandler<T> {
    View onBind(Context context, T data, int position);
}
```

你可以在onBind中操作你的View，这里提供数据以及该view的索引。就跟ListView的getView类似。

差不多了，使用方法就是这么简单。但是台上十分钟，台下十年功。

## 效果

![](/img/vp_card5.gif)

## 原理解析
从视觉效果来看，最先映入眼帘的是卡片，其次是卡片左右微微探头，暗中观察的两张卡片。卡片效果想都不用想，CardView。关于这个控件的封装，我放在了最后，因为我们对于view是完全开放的，只要是这样的滑动效果，可以填充任何的view，不局限于CardView，对于view更多布局效果，我也会推荐几种。

我们先来看看暗中观察的这两个家伙到底是怎么实现的。

这是我们最常见的ViewPager效果。

![](/img/vp_card1.gif)


### PageTransformer
我们再来看看这样的效果。

![](/img/vp_card2.gif)

不知道小伙伴们看清楚了吗？我看清楚了，前一个ViewPager在缩小和向右偏移，后一个ViewPager在放大和向左偏移。WTF？缩放我看到了，前面一个不是在左移吗？你跟我说右移？

同学，请先放下你手中的菜刀，听我细细道来，我们先看看源代码。

```
@Override
public void transformPage(View page, float position) {
    if (mViewPager == null) {
        mViewPager = (ViewPager) page.getParent();
    }

    int leftInScreen = page.getLeft() - mViewPager.getScrollX();
    int centerXInViewPager = leftInScreen + page.getMeasuredWidth() / 2;
    int offsetX = centerXInViewPager - mViewPager.getMeasuredWidth() / 2;
    float offsetRate = (float) offsetX * 0.38f / mViewPager.getMeasuredWidth();
    float scaleFactor = 1 - Math.abs(offsetRate);
    if (scaleFactor > 0) {
        page.setScaleX(scaleFactor);
        page.setScaleY(scaleFactor);
        page.setTranslationX(-mMaxTranslateOffsetX * offsetRate);
    }
}
```

这是PageTransformer核心方法，直接说你可能比较懵逼，秉承图文并茂的style，先来看看两张图，图也是盗的，有现成的为什么还要自己去做，就算你现在不这么认为，过几年就这么想了，我赌半包辣条。

![](/img/vp_transform1.png)

![](/img/vp_transform2.png)

这是你定制ViewPager各种炫酷的切换效果的基础，view就是ViewPager填充的view或者是fragment中的填充view。position上面已经很清楚，同学们大可自己玩玩，只有自己玩过，才知道什么是浪费时间，没毛病。

现在，我们来分析下该库为什么要这么写？一行一行分析，才能虎得住人，先来分析前3行代码，拿到ViewPager的引用，判空是防止重复获取，为什么page.getParent获取的是ViewPager，你就把它当成横铺的LinearLayout你就理解了（但内部并不是这样的，因为它有销毁和创建这么一说）。

接下来我们来看看leftInScreen，page.getLeft()表示当前view相对于父容器(ViewPager)离左边的距离，mViewPager.getScrollX()表示ViewPager在X轴滑动的距离(内容)，那么很明显啦，leftInScreen就是该view的左边距离当前view(ViewPager的当前Item)的左边的距离(因为该view的padding是算在getLeft里面的，因此得减去，padding值有正负，左正右负)。

这样说来centerXInViewPager就很好理解啦，由leftInScreen加上该view的宽度的一半，那就是从该view的中间开始算起。

offsetX相对来说比较绕，但也是核心所在。它是由centerXInViewPager加上ViewPager宽度的一半。一般来说，offsetX与leftInScreen是相等的，先加上一半再减去一半有毛区别，但这一半不一定相等，例如ViewPager加上padding。如果你对上面的解析都理解了的话，那就很简单了，就是该view相对于当前item的偏移量。

空间想象力强的人已经理解了，但我肯定会照顾所有的同学，下面给出图，同学们可以参考着图来理解上面的概念。

![](/img/vp_offset.png)

offsetRate和scaleFactor比较好理解啦，前者是offsetX相对于ViewPager的一个比值，后者用了Math.abs取绝对值是因为上面所返回的值都是有正负之分的，左负右正。

最后的属性修改我就不想多说了，作为一名有信仰的Android程序员，这个都不知道，我还是会选择原谅你。这里比较卡人的就是为什么要setTranslationX，就是因为我们调用了setScaleX，你想想你缩放了，是不是中间空出了这么点距离？所以这个就是弥补中间空出的，让人看上去一直是衔接的，你完全可以改，我们这里最大值取180dp，这也是为什么我一开始说前一个在右移的原因。

```
setPageTransformer(false, new CardTransformer(context));
```

这样就设置进去了，这里我解释下，第一个参数，我们库其实无关紧要，但作为一名有逼格的程序员，不刨根问底，实在对不起自己。

```
* @param reverseDrawingOrder true if the supplied PageTransformer requires page views
*                            to be drawn from last to first instead of first to last.
```

大致意思是说，如果为false，那么就从第一页开始绘制至最后一页，为true就是从最后一页开始绘制至第一页，如果对布局FrameLayout比较熟悉的同学，我相信难不倒你。

PageTransformer可以让你设计出超炫酷的切换动画，什么立体翻转啊，什么淡入淡出啊，等等。

大家也看看洋哥的[《Android 实现个性的ViewPager切换动画 实战PageTransformer（兼容Android3.0以下）》](http://blog.csdn.net/lmj623565791/article/details/40411921)

### ViewPager

我们的主角可是暗中观察的两个家伙啊，主角肯定是要登场啊。现在我们希望ViewPgaer能够留出两块区域来显示它们。回想一下，一开始，我让你们把ViewPager当成LinearLayout横铺，这里我依然这么做，ViewPager显示视图默认是只显示当前Item的，但我们只要把内容挤一挤不就有了，挤内容常见的就是padding了，如果仔细看上面部分的同学应该早就猜出来了。这里我设置了40dp的padding，来看看效果。

![](/img/vp_card3.gif)

看了效果图的同学又要骂娘了，右边有一小块覆盖了，最重要根本没看到那两家伙啊，前者是因为我们前面说的第一个参数为false，右边覆盖左边，这也证实了我们的结论。后者是因为我们没加clipToPadding属性设为false。看看名字就是知道了，是否剪裁padding，我不剪裁不就显示出来了？我们再来看看效果。

![](/img/vp_card4.gif)

有点意思了哈，要是视图之间再空开点就完美了，刚才我们用到了padding，不妨用用margin，俗话说外事不决问margin，内事不决问padding，很熟悉，但好像哪里出了点问题，没毛病，就是这样的，圈起来，这是重点。

```
int margin = typedArray
                .getDimensionPixelOffset(R.styleable.CardViewPager_card_margin,
                        (int) TypedValue.applyDimension(TypedValue.COMPLEX_UNIT_DIP, 40, displayMetrics));
setPageMargin(margin);
```

看看最终的效果。

![](/img/vp_card5.gif)

是不是很完美，这里关于CardView的使用我就不介绍了，我也提供了一个可自定义宽高比例，默认支持阴影的CardView，稍微要注意的一点是，如果用自己的CardView，最好加上

```
app:cardPreventCornerOverlap="true"
app:cardUseCompatPadding="true"
```
5.0之后默认是false，也就是说不支持阴影（阴影被遮住了）。

### Adapter
现在来分析如何构建视图的，这里我们用到的fragment，因为谷歌已经为我们处理了很多加载问题，拿来用就是，这里我们adapter继承于FragmentStatePagerAdapter，是因为我们卡片可能很多。

来看一看核心代码：

```
Context context = getContext();
List<CardItem> cardItems = new ArrayList<CardItem>();
for (int i = 0, size = data.size(); i < size; i++) {
    T t = data.get(i);
    CardItem<T> item = new CardItem<T>();
    item.bindHandler(handler);
    item.bindData(t, i);
    cardItems.add(item);
}

if (mHandler == null) {
    throw new RuntimeException("please bind the handler !");
}
return mHandler.onBind(mContext, mData, mPosition);
```

这里我们采用mHandler去重新构造View，而不是复用构造出来的View，是配合FragmentStatePagerAdapter使用的，能够更好利用资源和释放资源，不然很容易造成OOM哦。

## 结束语
文章差不多就到这里，我相信你现在有很多疑问，有很多想法。这些疑问并不是对我上面阐述的概念，而是对ViewPager的理解，或许你已经可以用另一种方法实现了，或许你已经在敲以前想设计但是没实现的代码了。如果对你有一点点帮助，那么我的目的就达到了。

## 传送门
Github:[https://github.com/crazysunj/CardSlideView](https://github.com/crazysunj/CardSlideView)

博客:[http://crazysunj.com/](http://crazysunj.com/)

