title: Lottie库实现直播礼物动画
tags: Lottie 直播 动态加载 礼物 动画
date: 2017-04-25 07:27:11
categories:
---

[Lottie](https://github.com/airbnb/lottie-android) 库我就不介绍了，不了解的可以看下这篇文章[Airbnb 开源炫酷动画库 Lottie（译）－看看 Airbnb 的工程师怎么说](https://juejin.im/entry/589ba3fc128fe1005801704b)

`Lottie 支持 Android、iOS、React Native 平台，支持实时渲染 After Effects 动画，使得 app 中使用动画可以像使用静态资源一样简单。`

## 使用场景

众所周知，做直播软件肯定少不了各种礼物的动画效果，当上线新的礼物时，不仅 Android、ios 需要实现新的动画效果，还很难兼容老版本。现在有了 Lottie，可以让设计师使用 After Effects 进行动画设计，通过 Bodymovin 插件导出 json 文件，将动画资源打包上传到服务器后，客户端通过动态下载资源文件来执行动画。这样上线新的礼物，只需要将资源文件上传，客户端不需要发版完全可以执行新礼物的动画效果。

## 使用详解

先看下动画效果：

![幽灵动画效果](https://mmbiz.qlogo.cn/mmbiz_gif/5MJZPMAX957mRHzicibvoEHbIUichBfvMKHRMwCFBzOQsZSLPY2GibsKkUF0iaVvnXicOz0uia7EOGDAost6ibib7ialQGiaA/0?wx_fmt=gif)

具体怎么实现呢？别着急，先来了解一下动画资源文件的结构。由于直播间送出的大礼物一般都是飞机、跑车、航母、花瓣雨等物品，这些物品不是简单的线条、色块能够绘制的出来，所以使用图片文件来实现动画效果。

这样一来设计师导出来的动画资源包括一个 json 文件和一组图片文件：

![资源文件目录](https://mmbiz.qlogo.cn/mmbiz_png/5MJZPMAX957mRHzicibvoEHbIUichBfvMKH1QICgdxYRibg3k2Q15rhPtamoxD2Ae59XtjtQ6QxAjweVhoCQic4nVWQ/0?wx_fmt=png)

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

![json文件](https://mmbiz.qlogo.cn/mmbiz_jpg/5MJZPMAX957mRHzicibvoEHbIUichBfvMKHdKmIT8UtrmrKMwzrsCvbyb8micqOYj4BwXFR9Koq4rwZE8Ihpibc3e0g/0?wx_fmt=jpeg)

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

这几行代码就实现了从SD卡动态加载动画，那么这样就算完工了吗？看看上面的动画是不是感觉有什么地方不对劲？  
好吧，作为一个Android软件工程师，一定要记住2个字 **适配** **适配** **适配**

## 源码解析

动画是全屏的效果，小幽灵也是从屏幕外飞进来的没有问题，为什么背景图离屏幕两边有空隙呢？

![没有适配](https://mmbiz.qlogo.cn/mmbiz_png/5MJZPMAX957mRHzicibvoEHbIUichBfvMKHXxiaFpU9EkWEM8NgVROH3Ia5OHl6vMj9xEoJqKnr3VmEXDJXtaJZic0g/0?wx_fmt=png)

再来看看这个动图，**为什么隐藏虚拟按键就全屏了呢？**

![隐藏虚拟按键](https://mmbiz.qlogo.cn/mmbiz_gif/5MJZPMAX957mRHzicibvoEHbIUichBfvMKHfrsFSSYqOCKoXMoZficF2ibBoJRmJvgxgafA3e3ibw02nrY98IEvCoe1w/0?wx_fmt=gif)

再来看看 json 文件里面的内容：

![json文件](https://mmbiz.qlogo.cn/mmbiz_jpg/5MJZPMAX957mRHzicibvoEHbIUichBfvMKHbvopnE0xKiaibxSU9QdNvES3WQBp2g46RO288PEovlavjAwW6uP6dDqg/0?wx_fmt=jpeg)

背景图的宽高和画布的宽高是一样的，那么为什么有虚拟按键的时候背景图就不全屏呢？原因其实很简单（然而找了半天才找到原因😅），我们来看一下源码，LottieAnimationView 加载 Composition 时判断了画布的宽高如果大于手机屏幕的宽高就进行等比例缩小：

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
      setScale(Math.max(xScale, yScale));
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

计算出缩放比后，调用 setScale 方法将缩放比设置到 lottieDrawable 上：

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

绘制的时候对 matrix 设置缩放比，然后调用了 compositionLayer 去进行具体的绘制，这个 compositionLayer 中有一个集合 `private final List<BaseLayer> layers = new ArrayList<>();` 就是 json 文件里面的 `layers` 节点解析出来的图层列表，其中还包含一些属性动画。`compositionLayer.draw(canvas, matrix, alpha);` 方法中主要调用了 drawLayer 抽象方法由图层的具体实现类执行绘制：

```java
void drawLayer(Canvas canvas, Matrix parentMatrix, int parentAlpha) {
    for (int i = layers.size() - 1; i >= 0 ; i--) {
      layers.get(i).draw(canvas, parentMatrix, parentAlpha);
    }
}
```

CompositionLayer 类是复合层，遍历 layers 集合一层一层的进行绘制，而我们是使用图片文件做的动画，对应的 Layer 实现类为 ImageLayer。 我们看下 ImageLayer 的绘制方法：

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

getBitmap() 方法会调用到一开始设置的代理类 ImageAssetDelegate ，获取到从 SD 卡读取到的图片。  

看代码中调用了 canvas 的 `save`、`restore` 方法来进行图层的叠加绘制，lottieDrawable 的 `draw` 方法传递下来的 `matrix` 用到了 `concat` 方法上，对 bitmap 进行了等比压缩。现在我们知道画布和 bitmap 都根据屏幕的宽高进行了等比缩放，而我们隐藏虚拟按键时，背景图片缩放的大小和屏幕宽高是吻合的，显示虚拟按键时，背景图片缩放后的高度和屏幕高度一样，但是宽度比屏幕的宽度要小，所以两边就有间距。

归根结底就是一句话：
**背景图的长宽比是 16 : 9，手机屏幕的长宽比也是 16 : 9，但是因为底部的虚拟按键占了一部分的高度，屏幕的可用空间的长宽比大约为 3 : 2**
**16 : 9 的图片再怎么等比缩放也不能铺满 3 : 2 的屏幕啊**

## 全屏适配

因为画布的长宽比是 16 : 9 ，那么只有在手机分辨率不是 16 : 9 或者带虚拟按键的手机上会有这个问题。

### 方案一

在执行动画的界面隐藏虚拟按键，或者将虚拟按键设置为透明浮在布局上面，这样屏幕的长宽比和画布的长宽比一样就没有问题。目前市面上的手机基本上都是 720P、1080P、2K 等分辨率，这些分辨率都是 16 : 9 的尺寸。

设置主题：

```xml
<style name="Theme">
        <item name="android:windowTranslucentStatus">true</item>
        <item name="android:windowTranslucentNavigation">true</item>
</style>
```

代码设置：

```java
Window window = getWindow();
int visibility = View.SYSTEM_UI_FLAG_LOW_PROFILE |
    View.SYSTEM_UI_FLAG_HIDE_NAVIGATION;
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
    window.clearFlags(WindowManager.LayoutParams.FLAG_TRANSLUCENT_STATUS);
    window.addFlags(WindowManager.LayoutParams.FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS);
    window.setStatusBarColor(ContextCompat.getColor(this, android.R.color.transparent));
    visibility |= View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN |
                    View.SYSTEM_UI_FLAG_LAYOUT_STABLE |
                    View.SYSTEM_UI_FLAG_IMMERSIVE_STICKY;
} else if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT) {
    window.addFlags(WindowManager.LayoutParams.FLAG_TRANSLUCENT_STATUS);
    visibility |= View.SYSTEM_UI_FLAG_IMMERSIVE_STICKY;
}
window.getDecorView().setSystemUiVisibility(visibility);
```

### 方案二

如果铁了心要适配不是 16 : 9 屏幕的手机，那么也是能够实现的😆😆 先解析 JSONObject 然后对比屏幕的长宽比和画布的长宽比相差超过 0.01，就根据屏幕长宽比修改画布的高度。

```java
InputStream stream = null;
try {
    stream = getAssets().open("79.json");
    int size = stream.available();
    byte[] buffer = new byte[size];
    stream.read(buffer);
    String json = new String(buffer, "UTF-8");
    JSONObject jsonObject = new JSONObject(json);
    DisplayMetrics displayMetrics = new DisplayMetrics();
    WindowManager wm = (WindowManager) getApplicationContext().getSystemService(Context.WINDOW_SERVICE);
    wm.getDefaultDisplay().getMetrics(displayMetrics);
    float displayScaled = displayMetrics.heightPixels / (float) displayMetrics.widthPixels;
    int w = jsonObject.optInt("w");
    int h = jsonObject.optInt("h");
    float scaled = h / (float) w;
    if (Math.abs(displayScaled - scaled) > 0.01F) {
        jsonObject.put("h", (int) (w * displayScaled));
    }
    mLottieAnimationView.setAnimation(jsonObject);
    mLottieAnimationView.setImageAssetsFolder("images");
} catch (IOException e) {
    throw new IllegalStateException("Unable to find file.", e);
} catch (JSONException e) {
    throw new IllegalStateException("Unable to load JSON.", e);
} finally {
    closeQuietly(stream);
}
```
