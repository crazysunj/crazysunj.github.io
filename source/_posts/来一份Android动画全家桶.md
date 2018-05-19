---
title: 来一份Android动画全家桶
toc: true
date: 2018-05-19 20:07:11
tags: [Android,Animator,Lottie,3D,Transition]
---
## 前言
自上次[《MTRVA2.0来啦》](https://juejin.im/post/5adb6322518825671a635aac)发布后，马上就有小伙伴问我有哪些Android动画，过了一段时间又有小伙伴问我啥时候发布Android动画。其实，在写《MTRVA2.0来啦》的时候，这次要讲的Android动画已经完成的差不多了，而在写这篇文章的时候，下个版本的内容也快写的差不多了（捂脸）。想提前学习的同学可以去我的开源项目[CrazyDaily](https://github.com/crazysunj/CrazyDaily)的develop分支，然后再跟我的文章过一遍，挺不错的学习方法。废话不多说，Android动画似乎已经是老生常谈的技术点，老生常谈的技术感觉总是潜移默化成为Android程序员的必备技能。今天，大家再跟我一起过一遍Android动画及进阶使用。

![](/img/jidong.jpg)
## 效果图
国际惯例，No picture,say a J8!

![](/img/crazydaily_anim.gif)

先来看看今天讲解的内容吧！

* lottie
* 3D动画
* 列表侧滑删除
* 列表展开闭合
* 转场动画进阶
* 卫星菜单

一个大类可能包含多种动画，比如转场动画用到了贝塞尔曲线的路径动画。再则说到Android动画总是避免不了扯到View，所以期间会带大家玩一玩自定义View。

`高能预警：本篇较长，可以先选择自己喜欢的内容进行阅读或者收藏一下有空再看。`

**温馨提示**：想看针对第五章节额外附送的属性动画源码分析请点[这里](http://crazysunj.com/2018/05/19/%E7%8E%A9%E4%B8%80%E7%8E%A9Android%E5%B1%9E%E6%80%A7%E5%8A%A8%E7%94%BB%E6%BA%90%E7%A0%81/)。
<!--  more-->
## 分析
这次的分析随着效果图的展示顺序一一讲解，首先，向我们迎面走来的是...额，不是，首先，是我们的可能是动画趋势的lottie动画。
### lottie
#### 简介
[lottie](https://github.com/airbnb/lottie-android)可能是一个革命性的动画库。为什么这么说？当然也只是我想这么说，先来看看lottie的震撼特点：

* 跨平台，支持Web、Android、iOS和React Native
* 在线更新动画，意味着不用发版和减少资源的占用体积等
* 相对于原生动画和GIF更为轻量

我们再来看看这样的一个效果图（官方效果图）：

![](/img/example_lottie_1.gif)

这么说吧，这种效果原生肯定是能写的，但是非常费脑子和精力，不信的同学可以尝试一下。其次用帧动画，缺点也很明显，资源占用很大。最简单的就是一张GIF图片，没有什么动画是GIF搞不定的（手动滑稽），但这应该是最差的方案了。

而lottie只要一份记录动画信息的json文件，就可以在各大平台跑起来。是不是很炫酷？
#### Android
So Easy！除了这个词我还真没想到怎么形容。废话不多说，先上代码：

```
implementation 'com.airbnb.android:lottie:2.5.4' //gradle依赖

<com.airbnb.lottie.LottieAnimationView
        android:id="@+id/splash_lottie"
        android:layout_width="wrap_content"
        android:layout_height="@dimen/splash_height"
        app:lottie_autoPlay="true"
        app:lottie_fileName="lottie/mtrva.json"
        app:lottie_imageAssetsFolder="images/mtrva/" />
```

是的，就是这么简单，只要把你的json文件传给LottieAnimationView，它就可以流畅地播放起来，像如何在代码中使用及其它我就不一一贴出来了，大家可以点[这里](http://airbnb.io/lottie/android/android.html)。
#### 扩展
这才是重头戏，面试的时候说自己会实现复杂的动画可能是个卖点，但现在似乎这个工作可以被设计师取代，我们先不谈什么级别的设计师，我们先来说说为啥我们Android程序员不可以来完成这份工作呢？跟我抢饭碗？不可能！

正如项目中Splash页面底下文字`[什么都懂一点，生活更多彩一些][廋金体]`，感兴趣就玩一玩，真的很有意思！

简单分析一下，Splash页面的动画，logo图标沿着s型的路径，淡入，放大。很简单的一个动画，原生实现也是很简单的。重点在于如何开发这样的lottie动画，只需要[Adobe After Effects](https://zh.wikipedia.org/wiki/Adobe_After_Effects)+[bodymovin](https://cdnjs.com/libraries/bodymovin)就可以轻松导出一份这样的json文件。而详细的安装及环境配置可以点[这里](https://github.com/airbnb/lottie-web)，不过英文要好哦，不然只能看看蹩脚的翻译。

简单的开发可以很快入门，我刚玩了一会儿，就开发了这样的一个效果：

![](/img/ae_example.gif)

很可惜，好像导出在Android端运行不太兼容，没有达到预期的效果，可能是我使用姿势不对。就玩了一会儿，弊端就体现出来了，这也是各种跨端的弊端，兼容性问题。一个库开源，使用者总是避免不了踩坑，例如上面xml中lottie\_imageAssetsFolder属性添加是因为json中有图片资源，需要图片资源的路径且图片资源名要改为image\_0，具体原因可以打开json文件瞧一瞧。

那么有同学问有没有什么现成的json吗？AE这玩意好像学起来挺麻烦的，这个肯定有，[lottiefiles](https://www.lottiefiles.com/)挺不错的一个网站，点击preview可以拖拽你的json文件进行预览和简单地编辑。

关于lottie动画介绍差不多就到这里，关键点都说了，剩下的可能是你的AE熟练度了。这玩意没法速成，需要大量经验积累！难点并不是技术，而是创意！创意！创意！

![](/img/kepa.jpg)
### 3D动画
进入首页，最先刺激我眼球的是右下角的妹子，其次是它的动画，这样的效果我最先在百度贴吧上看到的，现在好像去掉了，这也是我第一家公司面试的时候的作品，很有亲切感！

我们简单地拆解一下动画，可以分为这些：3D翻转、平移、阴影渐变和vignette。
#### 3D翻转
这个动画的核心，用的是补间动画，你也可以植入view中，我相信这对你来说并不难。

拆解动画：view分为两面，一面翻上去或者一面翻下去，然后展示另一面，所以分为4个动画TopInAnimation、TopOutAnimation、BottomInAnimation和BottomOutAnimation，Top和Bottom为反向操作，因此这里只分析Top。

如果没图，我猜有些同学不好理解，这里给出一张中间单帧图，画得不好，谅解，谅解。

![](/img/frame_3d_single.png)

不知道大家了解setRotationX这个方法吗，不清楚的可以点[这里](https://developer.android.google.cn/reference/android/view/View#setrotationx)。最常见就是我们的车轮，车轴就是X轴，然后侧着看。这样一想，是不是A和B都在做rotationX动画，但这是不够的，假如A面绕的X轴是高度中间等分线，直到A消失，也是消失在等分线的位置，脑补一下，而事实是A消失于顶部水平线，因此得做平移动画，也就是一边rotationX一边translationY。

了解这个后，我们再来了解两个核心类[Camera](https://developer.android.google.cn/reference/android/graphics/Camera)和[Matrix](https://developer.android.google.cn/reference/android/graphics/Matrix)。篇幅原因，只给出链接，大家可以深入了解，其实就算整片文章都介绍，那也说不完。这里说一下Camera的主要作用是将3D模型投影在2D的屏幕上，而Matrix主要通过一个3x3的矩阵对图像进行平移、旋转、缩放和错切变化。

在上代码之前，补充一个知识，左右手坐标系。

![](/img/lrcoordinatesystem.png)

Camera是基于左手坐标系的，它也应该是基于OpenGL的，OpenGL貌似是右手坐标系，而Android屏幕坐标系的Y轴方向正好与Camera的Y轴方向相反。

![](/img/mengbi_normal.jpg)

Camera比较好理解，你可以想象摄影大哥扛着摄像机对着屏幕左上角（原点），这个挺形象吧！Camera的默认位置是（0,0,-8），单位是英寸。Matrix相对比较复杂一点，大家可以看看这篇文章[Android中图像变换Matrix的原理、代码验证和应用](https://blog.csdn.net/pathuang68/article/details/6991867)，这是我早期学习时收藏的文章，优秀！

简单介绍完这两大核心类，我们来看看在项目中的运用：

```
static class TopOutAnimation extends Animation {
	...
    @Override
    protected void applyTransformation(float interpolatedTime, Transformation t) {
        mCamera.save();
        final float rotate = -DEGREE + DEGREE * interpolatedTime;
        mCamera.rotateX(rotate);
        mCamera.getMatrix(mMatrix);
        mCamera.restore();
        mMatrix.preTranslate(-mWidth / 2, 0);
        mMatrix.postTranslate(mWidth / 2, mHeight - mHeight * interpolatedTime);
        t.getMatrix().postConcat(mMatrix);
    }
}

 static class TopInAnimation extends Animation {
 	...
    @Override
    protected void applyTransformation(float interpolatedTime, Transformation t) {
        mCamera.save();
        final float rotate = DEGREE * interpolatedTime;
        mCamera.translate(0, -mHeight + interpolatedTime * mHeight, 0);
        mCamera.rotateX(rotate);
        mCamera.getMatrix(mMatrix);
        mCamera.restore();
        mMatrix.preTranslate(-mWidth / 2, -mHeight);
        mMatrix.postTranslate(mWidth / 2, 0);
        t.getMatrix().postConcat(mMatrix);
    }
}
```
先来分析一下TopOutAnimation，当interpolatedTime=0的时候，rotate=-90即模型是与屏幕是垂直的，当interpolatedTime=1的时候，rotate=0即正常位置。preTranslate是在旋转前平移模型的位置，至于为什么是-mWidth/2和0，原因也很简单，记住Camera核心作用就行了，就是我上面说的，将3D模型投影在2D的屏幕上，由于Camera的初始位置就是屏幕的原点，做旋转的时候，投影到画布的图形肯定是不正常的，因为不是正向的，那么只要在camera旋转前向左平移宽度的一半即为正向，但要在旋转后回到原来的位置，因此调用postTranslate且x值为mWidth/2。我相信这个比较好理解，所以不贴图了，至于preTranslate的y值是0是因为我们初始化定义的camera的rotateX是-90即与屏幕垂直，是这样的一个变换过程：

![](/img/threed_top_out.png)

有同学问，那我可不可一开始就在底部，这样我postTranslate的时候就不用移动了，从逻辑上一点毛病都没有，但事实上效果诧异，为什么呢？因为Matrix操作的是我们的模型而非屏幕，甚至移动的距离是原来的几倍，还发现变小了，为什么呢？这很简单，你离光源越远，肯定越小，至于为啥移动距离是原来的几倍，我给你画张图：

![](/img/threed_top_out_1.png)

![](/img/soudesinei.jpg)

那么是不是就这一种方法呢？那肯定不是，感兴趣的同学可以自己去尝试，自己动手实践过才印象最深刻。回到我们讨论点，由于我们preTranslate的y值是0使得投影效果在最顶部因此需要最终的view从底部不断往上偏移，故调用postTranslate的y值是mHeight-mHeight*interpolatedTime。

有了TopOutAnimation的基础分析，我们理解TopInAnimation相对容易一点。

上面的代码其实可以改成这样：

```
 static class TopInAnimation extends Animation {
 	...
    @Override
    protected void applyTransformation(float interpolatedTime, Transformation t) {
        mCamera.save();
        final float rotate = DEGREE * interpolatedTime;
        mCamera.rotateX(rotate);
        mCamera.getMatrix(mMatrix);
        mCamera.restore();
        mMatrix.preTranslate(-mWidth / 2, -mHeight);
        mMatrix.postTranslate(mWidth / 2, mHeight - interpolatedTime * mHeight);
        t.getMatrix().postConcat(mMatrix);
    }
}
```

那是不是Camera的y轴平移等同于Matrix的postTranslate平移呢？只能说近似等同。这次我直接画变换过程：

![](/img/threed_top_in_1.png)

最后跟TopOutAnimation一样调用postTranslate的平移就行了。

关于3D翻转就先介绍到这，不懂的可以问我，如果哪里不对的或者有歧义的地方欢迎指正。

#### 平移
这个就比较简单了，直接上代码：

```
public void start(boolean isTop, int index) {
        final float distance = -mTranslationYDistance * index;
    if (isTop) {
        mForegroundView.startAnimation(mTopOutAnimation);
        mBackgroundView.startAnimation(mTopInAnimation);
        ValueAnimator animator = ValueAnimator.ofFloat(0f, 1.0f)
                .setDuration(DURATION);
        animator.setInterpolator(new AccelerateDecelerateInterpolator());
        animator.addUpdateListener(animation -> {
            final float value = (float) animation.getAnimatedValue();
            final float oppositeValue = 1 - value;
            ...
            setTranslationY(distance * oppositeValue);
        });
        animator.start();
    } else {
        mForegroundView.startAnimation(mBottomInAnimation);
        mBackgroundView.startAnimation(mBottomOutAnimation);
        ValueAnimator animator = ValueAnimator.ofFloat(0f, 1.0f)
                .setDuration(DURATION);
        animator.setInterpolator(new AccelerateDecelerateInterpolator());
        animator.addUpdateListener(animation -> {
            final float value = (float) animation.getAnimatedValue();
            final float oppositeValue = 1 - value;
			...
            setTranslationY(distance * value);
        });
        animator.start();
    }
}
```

代码也给出3D翻转的调用，向上平移的时候向下翻转，向下平移的时候向上翻转，平移动画用的是ValueAnimator，因为我们还需要根据value计算阴影的颜色，一起来看看吧！
#### 阴影渐变
为啥要搞个渐变呢？因为要真，当立方体翻滚的时候，由于光线原因，一部分肯定越来越暗，一部分越来越亮。卧槽，物理这么6？还行，也就每次考试90来分。

![](/img/bishi.jpg)

```
//top
int foreStartColor = (int) mArgbEvaluator.evaluate(oppositeValue * 0.8f, 0x00000000, 0xff000000);
int foreCenterColor = (int) mArgbEvaluator.evaluate(oppositeValue * 0.4f, 0x00000000, 0xff000000);
int[] foreColors = {foreStartColor, foreCenterColor, 0x00000000};
int backStartColor = (int) mArgbEvaluator.evaluate(value * 0.8f, 0x00000000, 0xff000000);
int backCenterColor = (int) mArgbEvaluator.evaluate(value * 0.4f, 0x00000000, 0xff000000);
int[] backColors = {backStartColor, backCenterColor, 0x00000000};
mForegroundDrawable.setColors(foreColors);
mBackgroundDrawable.setColors(backColors);
//bottom
int foreStartColor = (int) mArgbEvaluator.evaluate(value * 0.8f, 0x00000000, 0xff000000);
int foreCenterColor = (int) mArgbEvaluator.evaluate(value * 0.4f, 0x00000000, 0xff000000);
int[] foreColors = {foreStartColor, foreCenterColor, 0x00000000};
int backStartColor = (int) mArgbEvaluator.evaluate(oppositeValue * 0.8f, 0x00000000, 0xff000000);
int backCenterColor = (int) mArgbEvaluator.evaluate(oppositeValue * 0.4f, 0x00000000, 0xff000000);
int[] backColors = {backStartColor, backCenterColor, 0x00000000};
mForegroundDrawable.setColors(foreColors);
mBackgroundDrawable.setColors(backColors);
```

mArgbEvaluator是Android提供的ArgbEvaluator类用来根据[0,1]某个值获取两种颜色渐变过程该进度时候的颜色值，好像挺拗口。我们这里操作的是drawable来实现渐变，那么drawable是谁的呢？是用户设置的子view吗？那肯定不是，不然要被打死。是我们在获取用户子view的时候在这基础上添加了一个ImageView与子view平级，上代码：

```
 @Override
protected void onFinishInflate() {
    super.onFinishInflate();
    ...
    View foregroundView = getChildAt(1);

    FrameLayout foregroundLayout = new FrameLayout(context);
    LayoutParams params = new LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, ViewGroup.LayoutParams.MATCH_PARENT);

    removeAllViewsInLayout();
    ...
    ImageView foregroundImg = new ImageView(context);
    ...
    foregroundImg.setImageDrawable(mForegroundDrawable);
    ...
    foregroundLayout.addView(foregroundView);
    foregroundLayout.addView(foregroundImg);

    addViewInLayout(foregroundLayout, 0, params);
    ...
    mForegroundView = (FrameLayout) getChildAt(1);
}
```

这里只给出ForegroundView的代码，其它贴出来毫无意义，代码也很简单，在子view外层套一层FrameLayout，然后添加负责阴影渐变的ImageView，渐变效果与我们平移时进度有关。
#### vignette
之所以加这个，跟我们的阴影渐变一样，要真。什么是vignette？vignette大概就是暗角的意思。举个栗子？

![](/img/example_vignette.jpg)

其主要特征就是在图像四个角有较为显著的亮度下降。这种效果自己去研究又是一堆公式，好在有大佬提供了方便。这里我们使用[GPUImage](https://github.com/CyberAgent/android-gpuimage)来达到这样的效果，该库主要提供各种各样的图像处理滤镜，并且支持照相机和摄像机的实时滤镜。很可惜的是这玩意创意来自于iOS。

那我们如何在Android中使用呢？

```
GPUImageVignetteFilter filter = new GPUImageVignetteFilter();
filter.setVignetteCenter(new PointF(0.5f, 0.5f));
filter.setVignetteColor(new float[]{0.0f, 0.0f, 0.0f});
filter.setVignetteStart(0.0f);
filter.setVignetteEnd(0.75f);
GPUImage gpuImage = new GPUImage(context);
BitmapDrawable bitmapDrawable = (BitmapDrawable) ContextCompat.getDrawable(context, id);
gpuImage.setImage(bitmapDrawable.getBitmap());
gpuImage.setFilter(filter);
Bitmap bitmap = gpuImage.getBitmapWithFilterApplied();
```

id是我们的资源id，主要就是要拿到一个bitmap对象，其它还支持file和uri。vignetteCenter就是我们向外扩展的起始点，一般在中心，例如我们上面的图片例子，vignetteColor是暗角的颜色，vignetteStart和vignetteEnd就是渐变程度。

3D动画是不是So Easy？是不是很炫酷？想不想自己实现一个？

![](/img/jujue.jpg)

关于3D动画先到这结束了，占用篇幅相对较长，看到这挺有耐心。
### 列表侧滑删除
long long ago，这个效果貌似是模仿...模仿啥来着？记不起来了，算了，早期的时候我也写过这个玩意，代码找不到了，哈哈，其实挺简单，主要就是触摸事件的运用，难点可能是用户体验，滑起来卡卡的，那玩个毛。

但现在都什么年代了，实现这种效果更简单了，Android直接给我们提供了[ItemTouchHelper](https://developer.android.google.cn/reference/android/support/v7/widget/helper/ItemTouchHelper)快速实现拖拽和侧滑删除。但需要简单地修改一下，因为Android提供给我们的好像与预期差了那么一点点，由于我比较懒且正好看到一个库[itemtouchhelper-extension](https://github.com/loopeer/itemtouchhelper-extension)是针对侧滑删除的，体验还不错，但还是没法满足我的需求，例如我想用户关闭界面的时候知道菜单是否是打开状态。那我只能使用CV大法然后修改，果然最灵活的还是CV源码进行修改。

本节主要是想介绍ItemTouchHelper，除了这，Android还提供了很多RecyclerView的帮助类，例如DiffUtil[[MTRVA](https://github.com/crazysunj/MultiTypeRecyclerViewAdapter)就是基于这个]，SnapHelper[帮助我们在某一个地方停住，例如你想尝试用RecyclerView实现卡片滑动，不妨试试]，这两个比较常用，当然还有其它的哦，大家不妨去看看这个包[androidx.recyclerview.widget]里面的东西，当然支持包里面也有，说不定有你想要的。
### 列表展开闭合
关于这个点貌似挺尴尬的，因为用[MTRVA](https://github.com/crazysunj/MultiTypeRecyclerViewAdapter)可以轻松实现，我都不知道要不要介绍，关键有几个用户同学会问这个，趁这个机会简单说一下吧。

比如效果图中到联系人列表有展开闭合的效果，怎么实现的呢？首先我们要排除直接操作view，比如让view隐藏等等。这是不可取的，应该用数据去驱动UI，这个我已经强调很多遍了。先看一看代码：

```
private void switchStatus(ContactHeader item, View icon) {
    final int index = mData.indexOf(item);
    final String flag = item.getIndex();
    if (item.isFlod()) {
        icon.animate().rotation(90).setDuration(500).start();
        mHelper.addData(index + 1, item.getChilds());
    } else {
        icon.animate().rotation(-90).setDuration(500).start();
        ...
        childs = mHelper.removeData(index + 1, count);
        item.setChilds(childs);
    }
    item.switchStatus();
    mHelper.setData(index, item);
}
```

很简单的逻辑，如果当前是闭合状态，icon需要选择90度且添加数据，反之，旋转-90度并移除数据。

关键点就在于addData和removeData，底层调用的是adapter的notifyItemRangeInserted和notifyItemRangeRemoved方法，因此是附带动画的，且对数组越界做了优化，例如addData的position大于或等于集合数量，那么直接将应添加的数据直接添加在集合的末尾，而不是抛异常。

本节主要希望大家利用添加和移除数据来达到展开闭合的效果。
### 转场动画进阶
每次玩Material Design产品的时候，总是有很多炫酷的转场动画刺激我。相信也有很多同学跟我一样，很喜欢这种风格。今天我就带大家了解一下转场动画并自定义转场动画，走起！
#### 常见转场动画
* [makeClipRevealAnimation \(View source, int startX, int startY, int width, int height\)](https://developer.android.google.cn/reference/android/support/v4/app/ActivityOptionsCompat.html#makecliprevealanimation)：从一个原点以圆形扩展至满屏，需要传入新Activity动画开始的view，因为其实坐标(startX, startY)都是相对source的，width和height是新Activity的初始宽高
* [makeCustomAnimation \(Context context, int enterResId, int exitResId\)](https://developer.android.google.cn/reference/android/support/v4/app/ActivityOptionsCompat.html#makecustomanimation)：这个类似我们的overridePendingTransition，传入进退动画即可
* [makeScaleUpAnimation \(View source, int startX, int startY, int startWidth, int startHeight\)](https://developer.android.google.cn/reference/android/support/v4/app/ActivityOptionsCompat.html#makescaleupanimation)                         ：与makeClipRevealAnimation相似，文档主要区别是setSourceBounds的设置
* [makeThumbnailScaleUpAnimation \(View source, Bitmap thumbnail, int startX, int startY\)](https://developer.android.google.cn/reference/android/support/v4/app/ActivityOptionsCompat.html#makethumbnailscaleupanimation)：指定一张缩略图缩放转场，参数与makeClipRevealAnimation意义相同
* [makeSceneTransitionAnimation \(Activity activity, View sharedElement, String sharedElementName\)](https://developer.android.google.cn/reference/android/support/v4/app/ActivityOptionsCompat.html#makescenetransitionanimation_4)：可能是最常用的，场景动画，从一个场景转到另一个场景，需要共享元素sharedElement协助过度且其必须指定sharedElementName即目标Activity共享元素的transitionName              
* [makeSceneTransitionAnimation \(Activity activity, Pair...<View, String> sharedElements\)](https://developer.android.google.cn/reference/android/support/v4/app/ActivityOptionsCompat.html#makescenetransitionanimation)：意义与上面相同，只是支持多个共享元素

#### 场景动画核心框架
* [Scene](https://developer.android.google.cn/reference/android/transition/Scene)：用来保存场景应用中View           的属性集合
* [Transition](https://developer.android.google.cn/reference/android/transition/Transition)：负责元素的过渡，你可以在不同场景根据属性值操作元素打造不同的动画
	* 普通过渡
		* [Explode](https://developer.android.google.cn/reference/android/transition/Explode)：从场景中心进入或移除，一种爆炸的感觉
		* [Fade](https://developer.android.google.cn/reference/android/transition/Fade)：最熟悉的淡入淡出
		* [Slide](https://developer.android.google.cn/reference/android/transition/Slide)：从场景边缘进入或移除
		* [AutoTransition](https://developer.android.google.cn/reference/android/transition/AutoTransition)：默认过渡，Fade+ChangeBounds
	* 共享元素过渡
		* [ChangeBounds](https://developer.android.google.cn/reference/android/transition/ChangeBounds)：根据场景前后布局界限变化执行过渡动画
		* [ChangeClipBounds](https://developer.android.google.cn/reference/android/transition/ChangeClipBounds)：根据场景前后getClipBounds变化执行过渡动画
		* [ChangeImageTransform](https://developer.android.google.cn/reference/android/transition/ChangeImageTransform)：根据场景前后ImageView的矩阵变化执行过渡动画
		* [ChangeScroll](https://developer.android.google.cn/reference/android/transition/ChangeScroll)：根据场景前后目标滚动属性的变化执行过渡动画
		* [ChangeTransform](https://developer.android.google.cn/reference/android/transition/ChangeTransform)：根据场景前后视图缩放和旋转变化执行过渡动画，当然也可以根据父视图的改变
	* 场景切换调用[共享元素添加SharedElement]
		* setEnterTransition：A->B，B进入的过渡
		* setExitTransition：A->B，A退出的过渡
		* setReturnTransition：B->A，B返回的过渡
		* setReenterTransition：B->A，A重进的过渡
* [TransitionManager](https://developer.android.google.cn/reference/android/transition/TransitionManager)：把上面Scene和Transition结合起来，常见的有通过setTransition(Scene, Transition)结合

#### 实战
前一小节貌似有点翻译的味道，但很多都是我亲自体验总结的，同时这也是必不可少的，我们要理论结合实践,再从实践中领悟真理。

![](/img/niubimao.jpg)
##### 场景分析
以联系人列表页为出发点，点击条目跳转到联系人详情页。期间发生了什么呢？

列表页条目的头像(其实过渡的时候已经是详情页的头像了）以贝塞尔曲线路径移动且缩放至详情页的头像；地址和名字平滑过渡到详情页；在共享元素过渡的时候，下一个场景也开始了进场动画，右下角的菜单跟小球一样弹跳下来同时背景以屏幕中心以圆形向外扩散。

##### 代码分析
首先我们先分析一下进场动画，因为这个相对比较好理解。

```
public class CircularRevealTransition extends Visibility {

	private static final String PROPNAME_ALPHA = "crazysunj:circularReveal:alpha";
	private static final String PROPNAME_RADIUS = "crazysunj:circularReveal:radius";
	private static final String PROPNAME_TRANSLATION_Y = "crazysunj:circularReveal:translationY";
	
	@Override
	public void captureStartValues(TransitionValues transitionValues) {
	    super.captureStartValues(transitionValues);
	    transitionValues.values.put(PROPNAME_ALPHA, 0.2f);
	    final View view = transitionValues.view;
	    transitionValues.values.put(PROPNAME_RADIUS, 0);
	    transitionValues.values.put(PROPNAME_TRANSLATION_Y, -view.getBottom());
	}
	
	@Override
	public void captureEndValues(TransitionValues transitionValues) {
	    super.captureEndValues(transitionValues);
	    transitionValues.values.put(PROPNAME_ALPHA, 1.0f);
	    final View view = transitionValues.view;
	    int radius = (int) Math.hypot(view.getWidth(), view.getHeight());
	    transitionValues.values.put(PROPNAME_RADIUS, radius);
	    transitionValues.values.put(PROPNAME_TRANSLATION_Y, 0);
	}
	
	@Override
	public Animator onAppear(ViewGroup sceneRoot, View view, TransitionValues startValues, TransitionValues endValues) {
	    if (null == startValues || null == endValues) {
	        return null;
	    }
	    final int id = view.getId();
	    switch (id) {
	        case R.id.satellite_menu:
	            int startTranslationY = (int) startValues.values.get(PROPNAME_TRANSLATION_Y);
	            float startAlpha = (float) startValues.values.get(PROPNAME_ALPHA);
	            int endTranslationY = (int) endValues.values.get(PROPNAME_TRANSLATION_Y);
	            float endAlpha = (float) endValues.values.get(PROPNAME_ALPHA);
	            PropertyValuesHolder translationY = PropertyValuesHolder.ofFloat("translationY", startTranslationY, endTranslationY);
	            PropertyValuesHolder alpha = PropertyValuesHolder.ofFloat("alpha", startAlpha, endAlpha);
	            ObjectAnimator menuAnim = ObjectAnimator.ofPropertyValuesHolder(view, translationY, alpha);
	            menuAnim.setInterpolator(new BounceInterpolator());
	            menuAnim.setDuration(1000);
	            return menuAnim;
	        case R.id.cool_bg_view:
	            int startRadius = (int) startValues.values.get(PROPNAME_RADIUS);
	            int endRadius = (int) endValues.values.get(PROPNAME_RADIUS);
	            Animator coolAnim = new NoPauseAnimator(ViewAnimationUtils.createCircularReveal(view, view.getWidth() / 2, view.getHeight() / 2, startRadius, endRadius));
	            coolAnim.setDuration(1000);
	            coolAnim.setInterpolator(new AccelerateDecelerateInterpolator());
	            return coolAnim;
	        default:
	            return null;
	    }
	}
	
	@Override
	public Animator onDisappear(ViewGroup sceneRoot, View view, TransitionValues startValues, TransitionValues endValues) {
	  ...
	}
	...
}   
```

captureStartValues用来保存动画需要的初始值，key-value的形式，至于key可以模仿系统以两个':'分离，captureEndValues用来保存动画需要的结束值。我们这里，小球透明度从0.2开始，同时从相同x的屏幕顶部掉落，背景初始的半径为0；结束时小球透明度正常，小球回到老地方，背景扩展至满屏，这里半径取了背景宽高的平方和的二次方根，关于值大家可以自己调。 

重点是继承了Visibility，像这种进场动画最好继承Visibility，因为它很好地提供了view的出现和消失方法。

我们再来看看onAppear方法，我们利用id来标识一个view，但这样就不灵活了。我们利用PropertyValuesHolder把translationY和alpha在一个view上同时执行，像这种针对同一个view且需要执行多个属性动画，就可以采用PropertyValuesHolder。背景的圆形扩散可以用[ViewAnimationUtils](https://developer.android.google.cn/reference/android/view/ViewAnimationUtils)来创建，这是Android提供的，必属精品。传入操作的view，扩散点坐标以及起始和结束半径。

onDisappear就是一个相反的过程，就不介绍了。我们再来看看共享元素动画代码：

```
public class BezierTransition extends Transition {
	...
    public BezierTransition() {
        setPathMotion(new PathMotion() {
            @Override
            public Path getPath(float startX, float startY, float endX, float endY) {
                Path path = new Path();
                path.moveTo(startX, startY);
                float controlPointX = (startX + endX) / 4;
                float controlPointY = (startY + endY) * 1.0f / 2;
                path.quadTo(controlPointX, controlPointY, endX, endY);
                return path;
            }
        });
    }

    private void captureValues(TransitionValues transitionValues) {
        Rect rect = new Rect();
        transitionValues.view.getGlobalVisibleRect(rect);
        transitionValues.values.put(PROPNAME_SCREEN_LOCATION, rect);
    }

    @Override
    public void captureStartValues(@NonNull TransitionValues transitionValues) {
        captureValues(transitionValues);
    }

    @Override
    public void captureEndValues(@NonNull TransitionValues transitionValues) {
        captureValues(transitionValues);
    }

    @Override
    public Animator createAnimator(ViewGroup sceneRoot,
                                   TransitionValues startValues, TransitionValues endValues) {
        if (null == startValues || null == endValues) {
            return null;
        }
        final int id = startValues.view.getId();
        if (id <= 0) {
            return null;
        }
        Rect startRect = (Rect) startValues.values.get(PROPNAME_SCREEN_LOCATION);
        Rect endRect = (Rect) endValues.values.get(PROPNAME_SCREEN_LOCATION);
        final View view = endValues.view;
        Path path = getPathMotion().getPath(startRect.centerX(), startRect.centerY(), endRect.centerX(), endRect.centerY());
        return ObjectAnimator.ofObject(view, new PropPosition(PointF.class, "position", new PointF(endRect.centerX(), endRect.centerY())), null, path);
    }
    ...
}
```

首先发现我们继承的是Transition，其次在构造函数里面我们执行了setPathMotion方法。[PathMotion](https://developer.android.google.cn/reference/android/transition/PathMotion)是Transition的一种扩展，提供以某种路径运动，有两个实现类[ArcMotion](https://developer.android.google.cn/reference/android/transition/ArcMotion)和[PatternPathMotion](https://developer.android.google.cn/reference/android/transition/PatternPathMotion)，感兴趣的可以研究下它的算法。前者是三阶贝塞尔曲线，后者是矢量曲线。

我们先来看看我们自己实现的有点low的算法，哈哈，就是一个简单的贝塞尔路径，难点可能是控制点的计算，如何让路径更优雅，这个艰巨的任务就交给你们了。

如果继承Transition，我们实现的是createAnimator，拿到我们创建的path，通过ObjectAnimator传入PropPosition实现的。

```
static class PropPosition extends Property<View, PointF> {
    PointF endPoint;
    PropPosition(Class<PointF> type, String name, PointF endPoint) {
        super(type, name);
        this.endPoint = endPoint;
    }
    @Override
    public void set(View view, PointF value) {
        int x = Math.round(value.x);
        int y = Math.round(value.y);
        int startX = Math.round(endPoint.x);
        int startY = Math.round(endPoint.y);
        int transY = y - startY;
        int transX = x - startX;
        view.setTranslationX(transX);
        view.setTranslationY(transY);
    }
    @Override
    public PointF get(View object) {
        return null;
    }
}

ObjectAnimator ofObject (T target, 
                Property<T, V> property, 
                TypeConverter<PointF, V> converter, 
                Path path)
```

可能ObjectAnimator的这个方法我们不常用，其实也很简单，就是根据传入的path，每个进度会回调一个对象，我们这里是PointF，由于默认的就是PointF，所以我们第三个参数传null就行了。假设我们需要回调的是Point对象，那么我们要实现一个TypeConverter<PointF, Point>，很好理解，就是用来类型转换的。

PropPosition的set方法回调中会返回每个进度根据path计算出来的PointF，这样我们就可以通过结束点计算出view需要平移的距离。

我知道大家很好奇是怎么传值的，这里我单独写了一篇文章[玩一玩Android属性动画源码](http://crazysunj.com/2018/05/19/%E7%8E%A9%E4%B8%80%E7%8E%A9Android%E5%B1%9E%E6%80%A7%E5%8A%A8%E7%94%BB%E6%BA%90%E7%A0%81/)来辅助大家理解。

![](/img/honglingjing.jpg)

回到我们的主题，既然类已经写完了，如何调用的呢？

```
//详情页
private void initTransition() {
    Window window = getWindow();
    TransitionSet set = new TransitionSet();
    AutoTransition autoTransition = new AutoTransition();
    autoTransition.excludeTarget(R.id.ic_head, true);
    autoTransition.addTarget(R.id.tx_name);
    autoTransition.addTarget(R.id.ic_location);
    autoTransition.addTarget(R.id.tx_location);
    autoTransition.setDuration(600);
    autoTransition.setInterpolator(new DecelerateInterpolator());
    set.addTransition(autoTransition);
    
    BezierTransition bezierTransition = new BezierTransition();
    bezierTransition.addTarget(R.id.ic_head);
    bezierTransition.excludeTarget(R.id.tx_name, true);
    bezierTransition.excludeTarget(R.id.ic_location, true);
    bezierTransition.excludeTarget(R.id.tx_location, true);
    bezierTransition.setDuration(600);
    bezierTransition.setInterpolator(new DecelerateInterpolator());
    set.addTransition(bezierTransition);
    
    CircularRevealTransition transition = new CircularRevealTransition();
    transition.excludeTarget(android.R.id.statusBarBackground, true);
    
    window.setEnterTransition(transition);
    window.setReenterTransition(transition);
    window.setReturnTransition(transition);
    window.setExitTransition(transition);
    
    window.setSharedElementEnterTransition(set);
    window.setSharedElementReturnTransition(set);
}
```

根据前面知识普及，我相信不需要解释太多，可能需要解释的地方就是addTarget和excludeTarget。addTarget就是指定参与过渡的view，excludeTarget就是排除过渡的view。

有些小伙伴可能对那个背景很好奇，它并不是一张背景图，中间以贝塞尔曲线分割，下面是白色，上面是前一个场景高斯模糊。贝塞尔曲线虽然不是什么很新鲜的东西，但是运用广泛，比如lottie章节开发用的AE中钢笔的运用就是贝塞尔曲线。

#### 小总结
整个过程下来，是不是发现转场动画也并不难，有些同学看到这可能已经自己写了几个过渡动画了。这我就很开心了，能帮到大家真正运用这些知识。当然了不要忘了添加windowContentTransitions属性哦，还有windowAllowEnterTransitionOverlap和windowAllowReturnTransitionOverlap来控制两个场景的动画要不要同步。NM的，咋不早说？

![](/img/dawoa.jpg)

### 卫星菜单
我记得在我毕业的时候，这玩意挺火的，看起来也很牛皮，实则实现起来很简单。可以说是动画的入门，那为啥要放这里呢？我就放这里，没说我要分析啊，告诉一下大家项目有这个动画。

## 骚聊
又到了紧张刺激的骚聊环节。到这里，肯定有小伙伴质疑我了，你确定这是Android动画全家桶吗？我可以很负责任的告诉你，是。只不过是普通规格的全家桶。无论是动画的种类还是动画的用法肯定是不全的，但是常用的已经八九不离十，重点是给大家总结个大概，感兴趣的可以深入了解。

趁这次机会，我回答一下，很多同学问我的问题，Android到什么地步才算厉害？首先我觉得这种问题很无聊，其次我自己的水平也并不高，不知道有没有资格回答。

鄙人在这发表一得之见，很多同学可能认为懂底层源码的人才是牛皮。懂底层确实很牛皮，但不是最牛皮的，只要你有耐心去阅读分析，不断深入，我相信你也可以和大佬一样写出深刻的源码分析，可能用的时间比大佬长那么一点，那么结果就出来了，学习能力。在这技术不断迭代更新的时代，最重要的是学习能力，因为可能你今天用的技术明天就被弃用了，当然这可能夸张一点，正如今天的Android动画分享，学习能力强的人看一遍自己过一遍已经可以举一反三了，再则，源码也是人写的，也是有迭代的，那你是不是需要重新分析一遍？看源码更多的是看大牛如何写代码，然后学以致用，今天主题动画的难点不是源码也不是如何使用，而是动画的创意！

哦，对了，我答应别人每天只能吹#个牛，祝大家生活愉快，溜了，溜了。有问题可以加我好友，我博客有联系方式，文章的代码都是开源项目[CrazyDaily](https://github.com/crazysunj/CrazyDaily)里面的。

最后，感谢一直支持我的人！
