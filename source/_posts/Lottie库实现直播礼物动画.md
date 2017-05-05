title: Lottie -- 轻松实现动态加载直播礼物动画
tags:
- Lottie
- 直播
- 动态加载
- 礼物
- 动画
date: 2017-04-25 07:27:11
categories: Android
toc: true
comments: true
description: Lottie，轻松实现炫酷动画，让 app 加载动画像加载图片一样简单。几分钟时间让你学会动态加载资源实现炫酷礼物动画，完美适配安卓各机型。
---
>本文已授权微信公众号：鸿洋（hongyangAndroid）原创首发。

本文主要讲解 [Lottie](https://github.com/airbnb/lottie-android) 库动态加载 SD 卡上带图片资源的动画，并对各种机型做全屏适配。

Lottie 的优点：

* 跨平台，支持 Android、iOS、React Native 平台

* 支持实时渲染 After Effects 动画，让 app 加载动画像加载图片一样简单。

* 资源动态下载，减小 APP 体积，上线新的动画效果不需要发版

* 更多优点等你发现

## 使用场景

* 做直播软件肯定少不了各种礼物的动画效果，当上线新的礼物时，不仅 Android、ios 客户端需要实现新的动画效果，还很难兼容老版本。

* 平常过节日的时候，很多 APP 都会做各种活动，修改启动页的图片，更改应用内的按钮图标，如果涉及到动画，那么肯定需要提前一两个版本将节日的动画实现代码预制到应用内

* 更多应用场景等你探索

现在有了 Lottie，可以让设计师使用 After Effects 进行动画设计，通过 Bodymovin 插件导出 json 文件，将动画资源打包上传到服务器后，客户端通过动态下载资源文件来执行动画。这样上线新的礼物，只需要将资源文件上传，客户端不需要发版完全可以执行新礼物的动画效果。流程如下：

{% qnimg Lottie/Lottie_9.jpeg title:流程图  %}  

## 使用详解

本文就以直播间播放动画为例子来讲解具体的实现方案，先看下动画效果：

{% qnimg Lottie/Lottie_1.gif title:动画效果  %}  

直播软件的大礼物一般都是飞机、跑车、航母、花瓣雨等，这些物品不是简单的线条、色块所能绘制，所以使用图片文件来实现动画效果。  
导出的动画资源包括一个 json 文件和一组图片文件：

{% qnimg Lottie/Lottie_2.jpg title:资源文件目录  %}

将这些文件打成压缩包上传到后台，客户端下载压缩包进行解压，使用 Lottie 加载本地资源执行动画：

```java
File jsonFile = new File(giftDir, "79.json");
File imagesDir = new File(giftDir, "images");
FileInputStream fis = null;
if (jsonFile.exists()) {
    try {
        fis = new FileInputStream(jsonFile);
    } catch (FileNotFoundException e) {
        e.printStackTrace();
    }
}
if (fis == null || !imagesDir.exists()) {
    showLocalAnimation(gift);
    return;
}
final String absolutePath = imagesDir.getAbsolutePath();
//提供一个代理接口从 SD 卡读取 images 下的图片
mLottieAnimationView.setImageAssetDelegate(new ImageAssetDelegate() {
    @Override
    public Bitmap fetchBitmap(LottieImageAsset asset) {
        BitmapFactory.Options opts = new BitmapFactory.Options();
        opts.inScaled = true;
        opts.inDensity = 160;
        return BitmapFactory.decodeFile(absolutePath + File.separator +
                asset.getFileName(), opts);
    }
});
//从文件流中加载 json 数据
LottieComposition.Factory.fromInputStream(this, fis, new OnCompositionLoadedListener() {
    @Override
    public void onCompositionLoaded(LottieComposition composition) {
        mLottieAnimationView.setVisibility(View.VISIBLE);
        mLottieAnimationView.setComposition(composition);
        mLottieAnimationView.playAnimation();
    }
});

```

关键的代码就是设置图片资源代理，去 SD 卡解析图片文件，那么怎么知道该解析哪一张图片呢？咱们来看看 json 文件里面的内容：

{% qnimg Lottie/Lottie_7.jpg title:json文件  %}  

`assets` 字段是图片资源的数组，具体的解析的源码如下：

```java
private LottieImageAsset(int width, int height, String id, String fileName) {
    this.width = width;
    this.height = height;
    this.id = id;
    this.fileName = fileName;
}

static class Factory {
    private Factory() {}

    static LottieImageAsset newInstance(JSONObject imageJson) {
        return new LottieImageAsset(
                  imageJson.optInt("w"),     //width
                  imageJson.optInt("h"),     //height
                  imageJson.optString("id"), //id
                  imageJson.optString("p")   //fileName
               );
    }
}
```

直接根据 `ImageAssetDelegate` 代理类的 `fetchBitmap(LottieImageAsset asset)` 方法中的 `LottieImageAsset` 参数获取当前需要解析的图片文件名，去 images 文件夹下面解析对应的文件就OK啦。

这几行代码就实现了从SD卡动态加载动画，那么这样就算完工了吗？看看上面的动画是不是感觉有什么地方不对劲？好吧，作为一个 Android 软件工程师，一定要记住2个字 **适配** **适配** **适配**

## 源码解析

动画是全屏的效果，小幽灵也是从屏幕外飞进来的没有问题，为什么背景图离屏幕两边有空隙呢？

{% qnimg Lottie/Lottie_4.jpg title:没有屏幕适配的效果  %}

再来看看这个动图，**为什么隐藏虚拟按键就全屏了呢？**

{% qnimg Lottie/Lottie_5.gif title:隐藏虚拟按键  %}

再来看看 json 文件里面的内容：

{% qnimg Lottie/Lottie_7.jpg title:json文件  %}

背景图的宽高和画布的宽高是一样的，那么为什么有虚拟按键的时候背景图就不全屏呢？原因其实很简单，来看一下 Lottie 是怎么解析 json 数据：

```java
static LottieComposition fromJsonSync(Resources res, JSONObject json) {
      Rect bounds = null;
      float scale = res.getDisplayMetrics().density;
      int width = json.optInt("w", -1);
      int height = json.optInt("h", -1);

      if (width != -1 && height != -1) {
        int scaledWidth = (int) (width * scale);
        int scaledHeight = (int) (height * scale);
        bounds = new Rect(0, 0, scaledWidth, scaledHeight);
      }

      long startFrame = json.optLong("ip", 0);
      long endFrame = json.optLong("op", 0);
      int frameRate = json.optInt("fr", 0);
      LottieComposition composition =
          new LottieComposition(bounds, startFrame, endFrame, frameRate, scale);
      JSONArray assetsJson = json.optJSONArray("assets");
      parseImages(assetsJson, composition);
      parsePrecomps(assetsJson, composition);
      parseLayers(json, composition);
      return composition;
}
```

解析出了动画的宽高、帧率等信息，这里将解析出来的宽高乘上了屏幕的像素密度，然后设置渲染区域的边界，为了便于理解，本文将其称为画布。我这台手机是 1080P 的分辨率，density = 3，scaledWidth = 2250，scaledHeight = 4002，现在缩放后的画布宽高比手机屏幕大了太多，如果动画在这种尺寸下进行渲染肯定不行。所以 LottieAnimationView 加载 Composition 时判断了画布的宽高如果大于手机屏幕的宽高就进行等比例缩小：

```java
public void setComposition(@NonNull LottieComposition composition) {
    if (L.DBG) {
      Log.v(TAG, "Set Composition \n" + composition);
    }
    lottieDrawable.setCallback(this);

    boolean isNewComposition = lottieDrawable.setComposition(composition);
    if (!isNewComposition) {
      // We can avoid re-setting the drawable, and invalidating the view, since the composition
      // hasn't changed.
      return;
    }
    //重点在这里，根据屏幕宽高对画布进行等比例缩放
    int screenWidth = Utils.getScreenWidth(getContext());
    int screenHeight = Utils.getScreenHeight(getContext());
    int compWidth = composition.getBounds().width();
    int compHeight = composition.getBounds().height();
    //如果画布的宽高大于屏幕宽高，计算缩放比
    if (compWidth > screenWidth ||
        compHeight > screenHeight) {
      float xScale = screenWidth / (float) compWidth;
      float yScale = screenHeight / (float) compHeight;
      //按比例缩小
      setScale(Math.min(xScale, yScale));
      Log.w(L.TAG, String.format(
          "Composition larger than the screen %dx%d vs %dx%d. Scaling down.",
          compWidth, compHeight, screenWidth, screenHeight));
    }


    // If you set a different composition on the view, the bounds will not update unless
    // the drawable is different than the original.
    setImageDrawable(null);
    setImageDrawable(lottieDrawable);

    this.composition = composition;

    requestLayout();
}
```

计算出宽和高的缩放比后，为了让画布小于屏幕，所以取较小的一个比例，调用 setScale 方法将缩放比设置到 lottieDrawable 上：

```java
public void setScale(float scale) {
    lottieDrawable.setScale(scale);
    if (getDrawable() == lottieDrawable) {
      setImageDrawable(null);
      setImageDrawable(lottieDrawable);
    }
}
```

lottieDrawable 的 setScale 方法保存了缩放比，并且更新了绘制的矩形范围：

```java
public void setScale(float scale) {
    this.scale = scale;
    updateBounds();
}
```

这里可以看到矩形的范围是根据画布的宽高进行了等比例的缩放

```java
private void updateBounds() {
    if (composition == null) {
      return;
    }
    setBounds(0, 0, (int) (composition.getBounds().width() * scale),
        (int) (composition.getBounds().height() * scale));
}
```

现在画布被缩放了，然而背景图呢？来看一下 lottieDrawable 的绘制代码：

```java
public void draw(@NonNull Canvas canvas) {
    if (compositionLayer == null) {
      return;
    }
    matrix.reset();
    matrix.preScale(scale, scale);
    compositionLayer.draw(canvas, matrix, alpha);
  }
```

lottieDrawable 在绘制的时候对 matrix 设置了缩放比，然后调用了 compositionLayer 去进行具体的绘制。这个 compositionLayer 就是所有图层的一个组合，它有一个`List<BaseLayer> layers`属性， 这个属性就是 json 文件里面的`layers`节点解析出来的图层列表，每个图层中间还包含一些属性动画。`compositionLayer.draw(canvas, matrix, alpha)`方法中主要调用了 drawLayer 抽象方法由图层的具体实现类执行绘制：

```java
void drawLayer(Canvas canvas, Matrix parentMatrix, int parentAlpha) {
    for (int i = layers.size() - 1; i >= 0 ; i--) {
      layers.get(i).draw(canvas, parentMatrix, parentAlpha);
    }
}
```

可以看到 CompositionLayer 类的 drawLayer 方法遍历 layers 集合进行循环绘制，这里是使用图片文件做的动画，对应的 Layer 实现类为 ImageLayer。 看下 ImageLayer 的绘制方法：

```java
public void drawLayer(@NonNull Canvas canvas, Matrix parentMatrix, int parentAlpha) {
    Bitmap bitmap = getBitmap();
    if (bitmap == null) {
      return;
    }
    paint.setAlpha(parentAlpha);
    canvas.save();
    canvas.concat(parentMatrix);
    src.set(0, 0, bitmap.getWidth(), bitmap.getHeight());
    dst.set(0, 0, (int) (bitmap.getWidth() * density), (int) (bitmap.getHeight() * density));
    canvas.drawBitmap(bitmap, src, dst , paint);
    canvas.restore();
}
```

getBitmap() 方法会调用到一开始设置的代理类 ImageAssetDelegate ，从 SD 卡加载图片。  

代码中调用了 canvas 的`save`、`restore`方法来进行图层的叠加绘制，从 lottieDrawable 的`draw`方法传递下来的`matrix`用到了`concat`方法上，对 bitmap 进行了等比缩放。

整个流程跑下来，Lottie 库的动画渲染机制已经基本了解，背景图没有全屏展示的原因如下：

**背景图的长宽比是 16 : 9，手机屏幕的长宽比也是 16 : 9，但是因为底部的虚拟按键占了一部分的高度，屏幕可用空间的长宽比大约为 3 : 2，所以导致背景图不能铺满屏幕**

## 全屏适配

动画不能全屏有两种情况，一种是手机长宽比和画布的长宽比是相同的，但是状态栏、导航栏占了屏幕一部分空间导致不能全屏，使用方案一可以解决问题。还有一种情况是手机屏幕长宽比和画布的长宽比就是不一样，毕竟 Android 机型这么多，有几台奇葩手机很正常，那么使用方案二可以实现全屏。

### 方案一

在执行动画的界面隐藏虚拟按键，或者将虚拟按键设置为透明浮在布局上面，这样屏幕的长宽比和画布的长宽比一样就没有问题。目前市面上的手机基本上都是 720P、1080P、2K 等分辨率，这些分辨率都是 16 : 9 的尺寸。

状态栏和虚拟按键透明悬浮在布局上面，设置样式：

```xml
<style name="Theme">
        <item name="android:windowTranslucentStatus">true</item>
        <item name="android:windowTranslucentNavigation">true</item>
</style>
```

隐藏虚拟按键通过代码设置：

```java
Window window = getWindow();
int visibility = View.SYSTEM_UI_FLAG_HIDE_NAVIGATION |
                View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN |
                View.SYSTEM_UI_FLAG_LAYOUT_STABLE |
                View.SYSTEM_UI_FLAG_LAYOUT_HIDE_NAVIGATION;
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
    window.clearFlags(WindowManager.LayoutParams.FLAG_TRANSLUCENT_STATUS);
    window.addFlags(WindowManager.LayoutParams.FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS);
    window.setStatusBarColor(ContextCompat
                    .getColor(getActivity(), android.R.color.transparent));
    visibility |= View.SYSTEM_UI_FLAG_IMMERSIVE_STICKY;
} else if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT) {
    window.addFlags(WindowManager.LayoutParams.FLAG_TRANSLUCENT_STATUS);
    visibility |= View.SYSTEM_UI_FLAG_IMMERSIVE_STICKY;
}
window.getDecorView().setSystemUiVisibility(visibility);
```

使用代码隐藏虚拟按键需要注意一点：**界面的切换会导致 setSystemUiVisibility() 的设置被清空，最好是在 onResume() 或者 onWindowFocusChanged() 方法中进行设置。**

### 方案二

如果要适配其他长宽比的屏幕，咋办呢？两行代码解决问题，只不过图片有一部分会被裁剪。设置控件的宽高为`match_parent`，设置`android:scaleType`为`centerCrop`

```xml
<com.airbnb.lottie.LottieAnimationView
        android:id="@+id/lottieAnimationView"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:scaleType="centerCrop"/>
```

## 总结

Lottie 发布才几个月，很多功能还不够完善，缓存机制也比较弱，像这种从 SD 卡动态加载的方式，需要自己去实现缓存逻辑。但是这点小瑕疵掩盖不了牛逼的事实，就目前这个需求来说，已经大大的降低了开发成本。只不过设计师们需要好好练练 AE 了，动画炫不炫就看设计师给不给力啦😆😆

本文是作者的处女作，如果对大家有帮助，希望大家多多支持，给予作者更多的创作动力，提供更好的作品给大家。  
<center>欢迎关注微信公众号：**大脑好饿**，更多干货等你来尝</center>
{% qnimg wx_qrcode.png title:公众号：大脑好饿  %}
