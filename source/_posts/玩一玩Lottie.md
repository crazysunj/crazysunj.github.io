---
title: 玩一玩Lottie
toc: true
date: 2020-08-09 09:25:17
tags: [lottie]
---
## 背景
我是一名Android开发者，以Android视角讲解。Lottie这玩意，我不记得几年前玩过，当时还是1.x版本，自己还通过AE鼓捣了很多好玩的动画，为此沾沾自喜，这可能就是学习的快感吧！最近项目需使用Lottie配合设计师实现各种复杂动画而再次被捡起。值得一提是Lottie的idea真的很棒，使用简单，是一个非常优秀的动画库。接下来大家跟我一起揭开她神秘的面纱。

![](/img/lottie_ciji.png)
## Lottie
### 简介
> Lottie是Airbnb开源的一个适用于Android、iOS、Web和Windows的动画库，解析Adobe After Effects（简称AE）通过Bodymovin导出的json，并以原生态在移动设备和Web上渲染。

百度翻译是真的不靠谱，还要我这个英语渣渣二次翻译。AE是什么？Bodymovin是什么？这个你不需要了解，让你用Lottie动画的设计师肯定懂 。什么，你主动提出来的？这都不知道，你提个毛线啊。

![](/img/lottie_hehe.jpg)

OK，现在就带你走进Android世界里的Lottie。
### 优势
为什么要用Lottie？传统的动画不香吗？传统动画大致分为帧动画、补间动画、属性动画和组合动画，甚至是gif。

现在我们就来吐槽一下这些动画，都2020年，应该很少人再用补间动画了吧，直接略过；如果是简单的动画属性动画是一个非常不错的选择，但复杂起来，不得不通过属性动画组合实现，而这套动画代码维护起来是非常操蛋的；如果直接使用gif，除开省事外，体验很差不说，占用的大内存和资源使用和回收时机是件让人头疼的事情，稍不小心就会引起内存溢出和卡顿，还有让Android开发大军跪下的分辨率适配工作；最后的帧动画如gif一样糟糕，gif其实就是帧动画。

再来看看我们今天的主角Lottie，Lottie只要一个json文件，文件可以来自网络，也可以来自本地，一份文件可以同时运行在Android、iOS、Web和Windows，实打实的跨平台，极大降低开发成本；几乎不需要额外的适配工作；占用内存小；可以控制动画速度和动态修改动画属性，用过的都说好，真的是碉堡了。

![](/img/lottie_zz.jpeg)

### 配置
Lottie在Android上支持的最低版本是**16**，其中AndroidX版本**2.8.0**及以上，我现在的项目还在使用support，只能用2.8.0以下的，故以下的讲解内容都是基于**2.7.0**版本，无论是哪个版本，原理性的内容是不会变的。

```
// build.gradle依赖对应的版本就可以玩起来了
implementation "com.airbnb.android:lottie:$lottieVersion"
```
### 核心类
结构真的简单，核心三个类。

* LottieAnimationView：展示动画的View
* LottieDrawable: 展示动画的Drawable
* LottieComposition：动画资源管理者

### 使用
使用真的简单。
#### 加载
加载大致分为这么几个场景：

本地Assets目录存放json/zip

```
#LottieAnimationView
public void setAnimation(final String assetName);
```
远程json/zip

```
#LottieAnimationView
public void setAnimationFromUrl(String url);
```
这是最常用的两种场景，但是作为高手，肯定不会用这两个方法，而是组合。Lottie提供这样的方法：

```
#LottieAnimationView
public void setComposition(@NonNull LottieComposition composition);
```

也就是说只要拿到LottieComposition，动画就可以跑起来。LottieComposition产生的唯一渠道，虽然有好多分支，但最终都到一个方法：

```
#LottieCompositionFactory
public static LottieResult<LottieComposition> fromXXXSync(XXX xxx  ...);

#LottieCompositionFactory
// 最终方法
public static LottieResult<LottieComposition> fromJsonReaderSync(JsonReader reader, @Nullable String cacheKey);

#LottieResult<LottieComposition>
@Nullable public LottieComposition getValue();
```

由方法名可知，方法是同步的，放在主线程肯定不行，必须自己实现一套异步加载，优化的时候必须安排各种缓存策略，总不能每次都io操作吧？这谁顶得住啊。

![](/img/lottie_ddz.jpg)

资深老Android肯定早就猜到Lottie自身已经提供，不然常见加载岂不是都重新加载io，别人可不这么low，当然你也可以自己实现一套加载机制（手动狗头）。

```
#LottieCompositionFactory
public static LottieTask<LottieComposition> fromXXX(XXX xxx ...);

#LottieTask<LottieComposition
public synchronized LottieTask<LottieComposition> addListener(LottieListener<LottieComposition> listener)

public interface LottieListener<LottieComposition> {
  void onResult(LottieComposition result);
}
```
由代码推导，LottieTask内部线程池启动线程通过前面介绍方法加载LottieComposition，通过LottieListener回调。

#### 基础功能
```
public class LottieDrawable extends Drawable implements Drawable.Callback, Animatable
```
LottieDrawable实现了Animatable，同时提供了动画常见的系列方法：

* addAnimatorUpdateListener：进度监听
* addAnimatorListener：周期监听
* playAnimation：开始动画
* endAnimation：结束动画
* resumeAnimation：恢复动画
* pauseAnimation：暂停动画
* cancelAnimation：取消动画
* setProgress：设置进度

是不是没有看到最常见的setDuration？这个也很好解释，Lottie本身是执行json文件，json本身包括一段动画，起始到结束，当时不需要设置什么时长。那我偏要设置时长呢？虽是有固定时长，说到底还是动画，必然是按帧执行，那么我通过addAnimatorUpdateListener和setProgress，当然方法执行的对象并不是同一个，相信聪明的你已经懂了。

#### 修改属性
Lottie虽支持属性修改，但修改属性有限，可能有些属性一脸懵逼，是AE中名词，设计师肯定懂，讨论动画时，把词抛出来就行了。

* Transform：变换
	* TRANSFORM\_ANCHOR_POINT：锚点
	* TRANSFORM\_POSITION：位置
	* TRANSFORM\_OPACITY：透明度
	* TRANSFORM\_SCALE：缩放
	* TRANSFORM\_ROTATION：旋转
* Fill：填充
	* COLOR：颜色，不支持渐变
	* STROKE\_WIDTH：线宽
	* OPACITY：透明度
	* COLOR\_FILTER：滤镜
* Ellipse：椭圆
	* POSITION：位置
	* ELLIPSE\_SIZE：大小
* Polystar：多边形
	* POLYSTAR\_POINTS：点的集合
	* POLYSTAR\_ROTATION：旋转角度
	* POSITION：位置
	* POLYSTAR\_OUTER\_RADIUS：外圆角
	* POLYSTAR\_OUTER\_ROUNDEDNESS：外圆角度
	* POLYSTAR\_INNER\_RADIUS：内圆角，用于星形
	* POLYSTAR\_INNER\_ROUNDEDNESS：内圆角度，用于星形
* Repeater：类似镜像，复制
	* REPEATER\_COPIES：复制
	* REPEATER\_OFFSET：偏移
	* TRANSFORM\_ROTATION：旋转
	* TRANSFORM\_START\_OPACITY：开始透明度
	* TRANSFORM\_END\_OPACITY：结束透明度   
* Layers：图层
	* 包含Transform所有属性
	* TIME\_REMAP：时间重置

列了这么多，那么代码怎么写呢？

```
#LottieAnimationView
public <T> void addValueCallback(KeyPath keyPath, T property, LottieValueCallback<T> callback);

#LottieAnimationView
// 通过此方法可以获取传参相关的所有KeyPath，可传通配符*和全匹配符**
public List<KeyPath> resolveKeyPath(KeyPath keyPath);
```

#### 修改资源
除了修改属性，还可以修改资源，或者说用自己项目的工具加载资源，Lottie提供了各种各样的代理。
##### 图片
图片是最常用的代理，可以通过LottieImageAsset提供的属性映射创建对应bitmap，不管是本地还是远程，甚至还可以做一下压缩。

```
mPetActionCore.setImageAssetDelegate(new ImageAssetDelegate() {
    @Override
    public Bitmap fetchBitmap(LottieImageAsset asset) {
		// 如果传null，Lottie会不断获取，知道不为null为止，原理在后面会讲解
		return null;
    }
});

// LottieImageAsset属性
public class LottieImageAsset {
  private final int width;
  private final int height;
  private final String id;
  private final String fileName;
  private final String dirName;
  @Nullable private Bitmap bitmap;
}
```

##### 字体
```
mPetActionCore.setFontAssetDelegate(new FontAssetDelegate(){
    // 可以通过fontFamily加载对应Typeface
    public Typeface fetchFont(String fontFamily) {
    }
    // 可以通过fontFamily返回assets的字体文件路径
    public String getFontPath(String fontFamily) {
    }
});
```

##### 文本
```
mPetActionCore.setTextDelegate(new TextDelegate(mPetActionCore) {
    // 一些列替换文本的方法
    ...
});
```

### 性能
无论玩什么，这都是无法逃避的话题。除了Android日常开发所使用的性能检测工具外，Lottie还给咱们提供建议和工具。

Lottie建议尽量不要在动画中加入遮罩/蒙版，这会大大降低性能，如果非要使用，最好开启硬件加速。说到这，顺便提一下，目前Lottie支持AE中的哪些特性呢？点击[这里](http://airbnb.io/lottie/#/supported-features)。

Lottie还给咱们提供了渲染性能检测工具。

```
// 开启性能检测，最好判断环境，例如在测试环境才开启
// 这里需要注意，该设置必须在设置LottieComposition之后，源码是最好的老师，不懂就去看看，这个后面源码分析就不作分析了
PerformanceTracker performanceTracker = mPetActionCore.getPerformanceTracker();
if (performanceTracker != null) {
    mPetActionCore.setPerformanceTrackingEnabled(true);
}

// 通过该方法可以打印每个图层渲染所需要的时间
// 注意，该方法必须在执行动画之后调用
PerformanceTracker performanceTracker = mPetActionCore.getPerformanceTracker();
if (performanceTracker != null) {
    List<Pair<String, Float>> sortedRenderTimes = performanceTracker.getSortedRenderTimes();
    Log.d(TAG, "各图层渲染时间:");
    for (int i = 0; i < sortedRenderTimes.size(); i++) {
        Pair<String, Float> layer = sortedRenderTimes.get(i);
        Log.d(TAG, String.format("\t\t%30s:%.2f", layer.first, layer.second));
    }
}

// 这里传授一个动画帧率是否稳定的黑科技，就是打印动画执行进度，如果打印的内容比平时少，说明就是丢帧了
// 也可以通过Choreographer，Lottie内部就是通过它刷新的
mPetActionCore.addAnimatorUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
    @Override
    public void onAnimationUpdate(ValueAnimator animation) {
        Log.d(TAG, "ActionType:" + mRunningActionType + " onAnimationUpdate--->" + animation.getAnimatedFraction());
    }
});
```

### 注意点
Lottie本身已经做的非常优秀了，如果真出了什么问题，也是无能为力的事情，只能放弃动画或者换动画，甚至放弃通过Lottie加载，不过咱们还是可以对一些小细节做优化的。

* 最常见的是首次执行丢帧。这个别无它法，资源就这么多，大家都在抢占，必然会引起卡顿或者丢帧。要么你把引起Lottie丢帧的罪魁祸首找出来，要么你就作延迟 ，一定时间后再执行。一般会选择后者，中间空档期会通过首帧占位图掩盖过去，首帧图自己想办法。
* 还有一个比较常见的就是动画素材本身有问题，不是设计师导出的文件有问题就是动画采用了Lottie不支持的特性，直接找设计师就完事了，常见的有元素丢失，空档期，效果与预览差异，元素位置偏移等等。
* 在动画的高频使用中可能会出现闪一下消失的情况，这往往是Lottie在加载资源时候，做了动画操作，所以维护好周期和状态是非常费脑的事情。
* 动画元素可能在执行的时候出现乱码情况，一般是多套json之间切换切位图存放目录不一样导致的，处理这种情况可在切换的时候通过setImageAssetDelegate来重新设置图片获取方式。

```
Log.w(VIEW_LOG_TAG, getClass().getSimpleName() + " not displayed because it is"
            + " too large to fit into a software layer (or drawing cache), needs "
            + projectedBitmapSize + " bytes, only "
            + drawingCacheSize + " available");
```
* 出现上面这种情况一般是图片太大了，超出了规范的大小，有两种做法，一种做法是减小图片尺寸，还有一种做法是通过硬件加速绕过检测。好奇为什么做的宝宝可以去看源码，这属于原生的范畴，不止是lottie。

像这种涉及资源使用的玩意，必然逃不掉RecyclerView的禁忌，尽量不要用，不然你会疯了的。非要作死？那你必须好好思考资源的使用和回收，动画的执行和取消时机，看似简单，其实要命。

![](/img/lottie_ym.png)

### 源码解析
其实对于Lottie来说，只要知道上面的内容就行了，源码理解的意义并不大，如果你是跟我一样是个有追求的程序员，那么咱们继续。

我分析源码还是喜欢从一个简单例子切入，整理各个分支流程，最后总结整个流程。当然知道结果再去看源码心境会不一样，这不是屁话么？我知道答案去考试和不知道答案去考试，这心境能一样吗？哪个更深刻呢？你应该知道答案了。

![](/img/lottie_ryss.jpg)

#### 加载
```
lottieAnimationView.setAnimation("xxx.json");
lottieAnimationView.playAnimation();
```
最简单的示例。xxx.json是assets目录下文件，OK，就从这入手。

一个多年开发经验的同学肯定立马判断setAnimation方法是初始化资源的。How？How？How？

```
#LottieAnimationView
public void setAnimation(final String assetName) {
    ...
    setCompositionTask(LottieCompositionFactory.fromAsset(getContext(), assetName));
}

private void setCompositionTask(LottieTask<LottieComposition> compositionTask) {
    ...
    this.compositionTask = compositionTask
            .addListener(loadedListener)
            .addFailureListener(failureListener);
}
```
从命名以及结构上大致判断，LottieTask是个资源加载任务，addListener和addFailureListener分别回调资源加载成功和失败。

要论证自己的猜想正确与否还是得从LottieTask的实际结构和创建流程出发。

```
#LottieCompositionFactory
public static LottieTask<LottieComposition> fromAsset(Context context, final String fileName) {
    final Context appContext = context.getApplicationContext();
    // 这里大致判断是个异步任务，call里是个同步获取
    return cache(fileName, new Callable<LottieResult<LottieComposition>>() {
      @Override public LottieResult<LottieComposition> call() {
        return fromAssetSync(appContext, fileName);
      }
    });
}
#LottieCompositionFactory
private static LottieTask<LottieComposition> cache(
          @Nullable final String cacheKey, Callable<LottieResult<LottieComposition>> callable) {
    final LottieComposition cachedComposition = LottieCompositionCache.getInstance().get(cacheKey);
    //  LottieCompositionCache是一个根据cacheKey存储LottieComposition的缓存类，内部实现是LruCache
    if (cachedComposition != null) {
      return new LottieTask<>(new Callable<LottieResult<LottieComposition>>() {
        @Override
        public LottieResult<LottieComposition> call() {
          // 读取到缓存直接封装成LottieResult，LottieTask直接返回
          return new LottieResult<>(cachedComposition);
        }
      });
    }
    // 可以认为是二级缓存，但不完全是，taskCache是用于防止LottieTask重复创建的缓存
    if (taskCache.containsKey(cacheKey)) {
      // 如果当前cacheKey已存在创建的LottieTask正在执行任务就直接返回
      return taskCache.get(cacheKey);
    }
    // 如果都没缓存的情况，创建LottieTask，传入callable
    LottieTask<LottieComposition> task = new LottieTask<>(callable);
    task.addListener(new LottieListener<LottieComposition>() {
      @Override public void onResult(LottieComposition result) {
        if (cacheKey != null) {
          // 对应开头，任务加载成功根据cacheKey缓存
          LottieCompositionCache.getInstance().put(cacheKey, result);
        }
        // 任务结束，正在执行任务缓存移除
        taskCache.remove(cacheKey);
      }
    });
    task.addFailureListener(new LottieListener<Throwable>() {
      @Override public void onResult(Throwable result) {
        // 任务失败，正在执行任务缓存移除
        taskCache.remove(cacheKey);
      }
    });
    // 把当前任务存储正在执行列表，保证正在执行不重复创建
    taskCache.put(cacheKey, task);
    return task;
}
```
从上面看lottie在任务管理做的不错，so，没必要自己再建一套。LottieTask创建方式我们已经很清楚，下一步是结构和任务执行。

```
#LottieTask
public LottieTask(Callable<LottieResult<T>> runnable) {
    this(runnable, false);
}
#LottieTask
LottieTask(Callable<LottieResult<T>> runnable, boolean runNow) {
    task = new FutureTask<>(runnable);
    if (runNow) {
      try {
        setResult(runnable.call());
      } catch (Throwable e) {
        setResult(new LottieResult<T>(e));
      }
    } else {
      // 必然会走else，通过线程池执行任务
      EXECUTOR.execute(task);
      startTaskObserverIfNeeded();
    }
 }
```
正常情况下，task执行完就宣布任务结束，产生结果，但这里不一样，task是通过外部传入，需要一个检测线程不断去检测是否已经执行完毕，选用FutureTask不就是这个目的吗？而下面的startTaskObserverIfNeeded检测的开始。

![](/img/lottie_nb.jpg)

```
#LottieTask
private synchronized void startTaskObserverIfNeeded() {
    // 如果线程存活或者已经知道结果，直接return
    if (taskObserverAlive() || result != null) {
      return;
    }
    taskObserver = new Thread("LottieTaskObserver") {
      private boolean taskComplete = false;
      @Override public void run() {
        // 死循环
        while (true) {
          // 线程被中断或者任务完成直接结束循环
          if (isInterrupted() || taskComplete) {
            return;
          }
          // 判断任务是否执行完成
          if (task.isDone()) {
            try {
              // 赋值结果
              setResult(task.get());
            } catch (InterruptedException | ExecutionException e) {
              setResult(new LottieResult<T>(e));
            }
            taskComplete = true;
            // 停止检测任务
            stopTaskObserverIfNeeded();
          }
        }
      }
    };
    // 线程开始执行
    taskObserver.start();
}
#LottieTask
private void setResult(@Nullable LottieResult<T> result) {
    // 本地缓存并通知回调
    this.result = result;
    notifyListeners();
}
#LottieTask
private void notifyListeners() {
    // 执行是在子线程，所以需要切到主线程
    handler.post(new Runnable() {
      @Override public void run() {
        if (result == null || task.isCancelled()) {
          return;
        }
        // 通知回调
        LottieResult<T> result = LottieTask.this.result;
        if (result.getValue() != null) {
          notifySuccessListeners(result.getValue());
        } else {
          notifyFailureListeners(result.getException());
        }
      }
    });
}
#LottieTask
private void notifySuccessListeners(T value) {
    // 通知回调
    List<LottieListener<T>> listenersCopy = new ArrayList<>(successListeners);
    for (LottieListener<T> l : listenersCopy) {
      l.onResult(value);
    }
}
#LottieTask
public synchronized LottieTask<T> addListener(LottieListener<T> listener) {
    if (result != null && result.getValue() != null) {
      listener.onResult(result.getValue());
    }
    successListeners.add(listener);
    // 这里解释下为什么addListener也会开启检测任务，在stopTaskObserverIfNeeded会判断successListeners为空就会中断检测任务，作者想的挺多
    startTaskObserverIfNeeded();
    return this;
}
```

整个资源的加载还是很巧妙的，值得学习。接下来介绍资源到底如何解析的。

#### 解析
```
#LottieCompositionFactory
public static LottieResult<LottieComposition> fromAssetSync(Context context, String fileName) {
	...
	// 创建cacheKey
	String cacheKey = "asset_" + fileName;
	if (fileName.endsWith(".zip")) {
		// 通过zip加载，其实内部还是fromJsonInputStreamSync
		return fromZipStreamSync(new ZipInputStream(context.getAssets().open(fileName)), cacheKey);
	}
	return fromJsonInputStreamSync(context.getAssets().open(fileName), cacheKey);
	...
}
#LottieCompositionFactory
public static LottieResult<LottieComposition> fromJsonInputStreamSync(InputStream stream, @Nullable String cacheKey) {
    return fromJsonInputStreamSync(stream, cacheKey, true);
}
#LottieCompositionFactory
private static LottieResult<LottieComposition> fromJsonInputStreamSync(InputStream stream, @Nullable String cacheKey, boolean close) {
	...
	// 无论你是通过什么类型加载，最后都是通过fromJsonReaderSync
	return fromJsonReaderSync(new JsonReader(new InputStreamReader(stream)), cacheKey);
}
#LottieCompositionFactory
public static LottieResult<LottieComposition> fromJsonReaderSync(JsonReader reader, @Nullable String cacheKey) {
	...
	// 开始解析，核心部分
	LottieComposition composition = LottieCompositionParser.parse(reader);
	// 这里又一次缓存
	LottieCompositionCache.getInstance().put(cacheKey, composition);
	return new LottieResult<>(composition);
}
#LottieCompositionParser
// JsonReader可能很多人很少用，这是Android提供用来解析json
public static LottieComposition parse(JsonReader reader) throws IOException {
    // 屏幕密度，用来适配
    float scale = Utils.dpScale();
    // 首帧
    float startFrame = 0f;
    // 结束帧
    float endFrame = 0f;
    // 帧率
    float frameRate = 0f;
    // 根据Layer的id存储Layer
    final LongSparseArray<Layer> layerMap = new LongSparseArray<>();
    // 首层能解析出来的Layer
    final List<Layer> layers = new ArrayList<>();
    // 动画宽高
    int width = 0;
    int height = 0;
    // 根据assets层id存储Layer集合
    Map<String, List<Layer>> precomps = new HashMap<>();
    // 根据assets层id存储assets相关信息
    Map<String, LottieImageAsset> images = new HashMap<>();
    // 根据字体名字存储字体相关信息，不作介绍
    Map<String, Font> fonts = new HashMap<>();
    // 根据编码hashcode存储编码相关信息，不作介绍
    SparseArrayCompat<FontCharacter> characters = new SparseArrayCompat<>();
    // 初始化LottieComposition
    LottieComposition composition = new LottieComposition();
    reader.beginObject();
    while (reader.hasNext()) {
      // 根据json解析，没写注解的参考上方注解
      switch (reader.nextName()) {
        case "w":
          width = reader.nextInt();
          break;
        case "h":
          height = reader.nextInt();
          break;
        case "ip":
          startFrame = (float) reader.nextDouble();
          break;
        case "op":
          endFrame = (float) reader.nextDouble() - 0.01f;
          break;
        case "fr":
          frameRate = (float) reader.nextDouble();
          break;
        case "v":
          // 解析bodymovin版本
          break;
        case "layers":
          // 解析图层，不作分析，意义不大，重要的类Layer
          parseLayers(reader, composition, layers, layerMap);
          break;
        case "assets":
          // 解析位图信息，重要的类LottieImageAsset
          parseAssets(reader, composition, precomps, images);
          break;
        case "fonts":
          parseFonts(reader, fonts);
          break;
        case "chars":
          parseChars(reader, composition, characters);
          break;
        default:
          reader.skipValue();
      }
    }
    reader.endObject();
    // 根据屏幕密度重新调整宽高
    int scaledWidth = (int) (width * scale);
    int scaledHeight = (int) (height * scale);
    Rect bounds = new Rect(0, 0, scaledWidth, scaledHeight);
    // composition开始初始化，至此composition结构也不用看了，就是这些
    composition.init(bounds, startFrame, endFrame, frameRate, layers, layerMap, precomps,
        images, characters, fonts);
    return composition;
}
// 具体怎么解析出来不贴代码，真的很繁琐，看看上面就懂了，而且这个意义并不大
public class Layer {
	// 就是shape，各种形状
	private final List<ContentModel> shapes;
	// 内部引用LottieComposition，可能是想单layer获取全局信息，只看到了字体有用到
	private final LottieComposition composition;
	// 图层命名
	private final String layerName;
	// 图层id
	private final long layerId;
	// 图层类型，目前有组合，固态（纯色填充），位图，Null（不作任何事情），Shape，文本，Unknown（不认识，可能手动改的吧）
	private final LayerType layerType;
	// 父层id
	private final long parentId;
	// 引用id，用于获取位图
	@Nullable private final String refId;
	// 蒙版信息，支持3种模式，相加，相减，交集模式
	private final List<Mask> masks;
	// 变化属性，锚点，位置，旋转，缩放，透明度
	private final AnimatableTransform transform;
	// 填充宽度
	private final int solidWidth;
	// 填充高度
	private final int solidHeight;
	// 填充色
	private final int solidColor;
	// 快进比例
	private final float timeStretch;
	// 开始帧
	private final float startFrame;
	// 组合层宽度
	private final int preCompWidth;
	// 组合层高度
	private final int preCompHeight;
	// 文本关键帧动画
	@Nullable private final AnimatableTextFrame text;
	// 文本颜色，边框宽，边框颜色，追踪关键帧动画
	@Nullable private final AnimatableTextProperties textProperties;
	// 时间重置关键帧动画
	@Nullable private final AnimatableFloatValue timeRemapping;
	// 起始结束关键帧动画
	private final List<Keyframe<Float>> inOutKeyframes;
	// 遮罩类型，支持两种，正常和反转
	private final MatteType matteType;
}

public class LottieImageAsset {
	// 位图宽度
	private final int width;
	// 位图高度
	private final int height;
	// 位图id
	private final String id;
	// 位图命名
	private final String fileName;
	// 位图相对目录名，这个是相对设计师给你文件
	private final String dirName;
	// 当前已缓存的bitmap
	@Nullable private Bitmap bitmap;
}
```

上面一系列信息的管家是Composition。而拥有这些信息就可以轻松实现复杂的动画。

![](/img/lottie_mb.jpg)

#### 设置
```
#LottieAnimationView
private final LottieListener<LottieComposition> loadedListener = new LottieListener<LottieComposition>() {
    @Override public void onResult(LottieComposition composition) {
      // 突然到这里，一直云里雾里的同学可能更加懵逼了
      // 加载并解析构建成LottieComposition，LottieAnimationView设置
      setComposition(composition);
    }
};

#LottieAnimationView
public void setComposition(@NonNull LottieComposition composition) {
    ...
    // lottieDrawable设置composition
    boolean isNewComposition = lottieDrawable.setComposition(composition);
    ...
    // LottieAnimationView设置drawable
    setImageDrawable(lottieDrawable);
    ...
}
```
万事俱备，只欠渲染。

#### 渲染
很显然我们直接跟踪lottieDrawable的渲染逻辑即可。

```
#LottieDrawable
@Override public void draw(@NonNull Canvas canvas) {
    ...
    // 核心渲染，compositionLayer怎么来的？貌似LottieDrawable的setComposition还有一些文章
    compositionLayer.draw(canvas, matrix, alpha);
    ...
}
#LottieDrawable
public boolean setComposition(LottieComposition composition) {
    ...
    buildCompositionLayer();
    ...
}
#LottieDrawable
private void buildCompositionLayer() {
    // 又构建了个新玩意CompositionLayer，这就是组合/合成图层
    compositionLayer = new CompositionLayer(
        this, LayerParser.parse(composition), composition.getLayers(), composition);
}
// CompositionLayer集成BaseLayer
public class CompositionLayer extends BaseLayer {
	...
	private final List<BaseLayer> layers = new ArrayList<>();
	...
	// layerModel是手动构建命名为__container，当然指的是LottieDrawable里边的
	public CompositionLayer(LottieDrawable lottieDrawable, Layer layerModel, List<Layer> layerModels,
      LottieComposition composition) {
		...
		// 把所有layer转化成BaseLayer
		for (int i = layerModels.size() - 1; i >= 0; i--) {
		  Layer lm = layerModels.get(i);
		  BaseLayer layer = BaseLayer.forModel(lm, lottieDrawable, composition);
		  ...
		  layers.add(0, layer);
		  ...
		}
		...
      }
      ...
}
#BaseLayer
static BaseLayer forModel(
    Layer layerModel, LottieDrawable drawable, LottieComposition composition) {
    // 这些类型之前已经介绍过了
    switch (layerModel.getLayerType()) {
      case Shape:
        return new ShapeLayer(drawable, layerModel);
      case PreComp:
        return new CompositionLayer(drawable, layerModel,
            composition.getPrecomps(layerModel.getRefId()), composition);
      case Solid:
        return new SolidLayer(drawable, layerModel);
      case Image:
        return new ImageLayer(drawable, layerModel);
      case Null:
        return new NullLayer(drawable, layerModel);
      case Text:
        return new TextLayer(drawable, layerModel);
      case Unknown:
      default:
        return null;
    }
}
#BaseLayer
// 再回到LottieDrawable的draw方法，里边通过CompositionLayer调用draw方法，而CompositionLayer继承BaseLayer
@Override
public void draw(Canvas canvas, Matrix parentMatrix, int parentAlpha) {
	...
	drawLayer(canvas, matrix, alpha);
	...
}
#CompositionLayer
@Override void drawLayer(Canvas canvas, Matrix parentMatrix, int parentAlpha) {
    ...
    for (int i = layers.size() - 1; i >= 0 ; i--) {
		...
		// 这里BaseLayer就是上边通过layer转化后的
		BaseLayer layer = layers.get(i);
		layer.draw(canvas, parentMatrix, parentAlpha);
		...
    }
    ...
}
// 这里只挑选场景Image作讲解，主要为了解释前面的坑，其它的感兴趣可以看源码
public class ImageLayer extends BaseLayer {
	...
	@Override public void drawLayer(@NonNull Canvas canvas, Matrix parentMatrix, int parentAlpha) {
	    // 获取bitmap
	    Bitmap bitmap = getBitmap();
	    ...
	    // 最后作draw，很简单
	    canvas.drawBitmap(bitmap, src, dst , paint);
	    ...
	}
	@Nullable
	private Bitmap getBitmap() {
		// refId前面介绍的引用id
		String refId = layerModel.getRefId();
		return lottieDrawable.getImageAsset(refId);
	}
	...
}
#LottieDrawable
@Nullable public Bitmap getImageAsset(String id) {
	ImageAssetManager bm = getImageAssetManager();
	if (bm != null) {
	  return bm.bitmapForId(id);
	}
	return null;
}
#LottieDrawable
private ImageAssetManager getImageAssetManager() {
	...
	// 注意这，一旦创建过，不会第二次构建
	if (imageAssetManager == null) {
	  imageAssetManager = new ImageAssetManager(getCallback(),
	      imageAssetsFolder, imageAssetDelegate, composition.getImages());
	}
	return imageAssetManager;
}
#ImageAssetManager
@Nullable public Bitmap bitmapForId(String id) {
	// 解析的时候存储的
	LottieImageAsset asset = imageAssets.get(id);
	// 如果LottieImageAsset已经有缓存bitmap直接返回
	Bitmap bitmap = asset.getBitmap();
	if (bitmap != null) {
	  return bitmap;
	}
	// 如果有设置代理直接从代理里面取
	// 这也为什么声称可以一直取直到取到，不停绘制不停取，不是废话吗，太让我失望了（手动狗头）
	if (delegate != null) {
	  bitmap = delegate.fetchBitmap(asset);
	  if (bitmap != null) {
	    // 就是上面的缓存
	    putBitmap(id, bitmap);
	  }
	  return bitmap;
	}
	...
	if (filename.startsWith("data:") && filename.indexOf("base64,") > 0) {
	  // 解析到base64格式创建返回
	  byte[] data = Base64.decode(filename.substring(filename.indexOf(',') + 1), Base64.DEFAULT);
	  ...
	  bitmap = BitmapFactory.decodeByteArray(data, 0, data.length, opts);
	  return putBitmap(id, bitmap);
	}
	...
	// 默认通过assets目录
	InputStream is = context.getAssets().open(imagesFolder + filename);
	...
	bitmap = BitmapFactory.decodeStream(is, null, opts);
	// 综上可知，通过外部设置一共就两种，通过设置aseets图片目录，设置图片代理，那么在这两种方式切换就很容易出问题，出现乱码，为什么？因为ImageAssetManager只有setDelegate方法而没有setImagesFolder方法
	// imagesFolder只能通过构造方法传入，只能一次机会，所以你前一次通过delegate方式获取图片，而第二次通过imagesFolder方式获取图片是有问题的，imagesFolder并不会改变
	return putBitmap(id, bitmap);
}
```

#### 感慨
整个源码分析到这就结束了，省略了很多烦杂的源码，主要是解析那部分，我们只分析了主架构的解析。其本身是没有意义的，什么时候你必须去研究研究呢？那就是出问题的时候！设计师只负责效果，发现根本跑不起来，这时候就到你装逼的时候了，前提是你也得略懂AE知识。
## 结束语
今天讲解的Lottie圆满结束，完结撒花。能从头看到这的同学我也挺佩服的，而那些直接拉到这的我只能嗤之以鼻。本篇文章对lottie知识点算是罗列的非常详细了，当你忘了就过来看看。当然，你放心，我不会根据lottie升级而调整的。我想核心点是不会变的，机制说改就改的，rubbish！！！

![](/img/lottie_bd.jpg)