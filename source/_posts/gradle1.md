title: 使用 Gradle 对应用进行个性化定制
tags:
- Gradle
date: 2017-07-07 16:19:23
categories: Android
toc: true
comments: true
description: 本篇文章主要根据实际开发中遇到的需求，讲解使用 `Gradle` 对应用的不同版本进行个性化定制。避免手动修改一些开关变量，导致误操作等等。
---


啥也不说了，直接进入主题吧。本篇文章主要根据实际开发中遇到的需求，讲解使用 `Gradle` 对应用的不同版本进行个性化定制。

# 场景介绍

1. 一般的应用基本上都有正式服和测试服，这个就不需要多说了。但是有些应用可能还有`超管服务器`专供运营人员使用，对应用内的一些内容进行监管，具有一些管理员才有的操作权限。
2. 开发过程中发布测试服务器的安装包需要在版本号后面增加版本序号，超管服务器的包在版本号后面增加`管理员`文字，线上包则正常显示版本号。
3. 每次打包 `versionCode` 自增，避免发版时忘记手动修改导致老版本不能覆盖安装。
4. 超管包的渠道名为 `admin`，日常运行的 `debug` 包渠道名为 `test`，上线的包使用加固软件进行多渠道加固。
5. `debug` 包和 `release` 包使用同样的签名，避免直接运行的 `debug` 包因为签名问题不能使用需要校验签名的第三方服务，比如：QQ 登录，微信登录，高德地图。
6. `debug` 包打印日志信息，`release` 包不打印日志信息

以上某些场景从我工作以来就一直存在，以前用 `eclipse` 开发时除了每次都手动去修改一些开关变量也没啥好办法，可能是因为当时菜 ╮(╯▽╰)╭（如果你们有什么好方法的话）。后来切换到 `Android Studio` 后使用 `Gradle` 进行依赖管理已经让人很是欣喜，既然如此能不能使用 `Gradle` 将以上问题统统解决，完全自动化呢？答案是：必须的。

# 解决问题
先上完整的 `Gradle` 配置。

```gradle
android {
    compileSdkVersion 25
    buildToolsVersion "25.0.3"
    defaultConfig {
        applicationId "com.imliujun.gradle"
        minSdkVersion 16
        targetSdkVersion 25
        versionCode gitVersionCode()  //获取 git 的 commit 次数
        versionName rootProject.ext.versionName

        manifestPlaceholders = [UMENG_APP_KEY      : "填你的友盟 APP KEY",
                                UMENG_CHANNEL_VALUE: "默认的渠道名"]
    }

    signingConfigs {
	//在这里配置相关的签名信息
        keyStore {
            storeFile file("test.jks")
            storePassword "111111"
            keyAlias "test"
            keyPassword "111111"
        }
    }

    buildTypes {
        release {
            // 不显示Log
            buildConfigField "boolean", "LOG_DEBUG", "false"
            signingConfig signingConfigs.keyStore //设置签名文件
        }

        debug {
            // 显示Log
            buildConfigField "boolean", "LOG_DEBUG", "true"
            versionNameSuffix "-debug"  //设置后缀
            signingConfig signingConfigs.keyStore  //设置签名文件
            manifestPlaceholders.UMENG_CHANNEL_VALUE = "test" //修改渠道名
        }
    }

    productFlavors {
        offline {
            buildConfigField "String", "DOMAIN_NAME", "\"https://offline.domain.com/\""
            versionName getTestVersionName() //修改 versionName
        }

        online {
            buildConfigField "String", "DOMAIN_NAME", "\"https://online.domain.com/\""
        }

        admin {
            buildConfigField "String", "DOMAIN_NAME", "\"https://admin.domain.com/\""
            versionName rootProject.ext.versionName + "-管理员" //修改 versionName
            manifestPlaceholders.UMENG_CHANNEL_VALUE = "admin" //修改渠道名
        }
    }
}

```

根目录下的 `build.gradle` 文件进行如下配置，主要是将版本号和测试包的序号抽取出来：

```gradle
ext {
    versionName = "2.0.2"
    testNum = "0001"
}

def getTestVersionName() {
    return String.format("%s.%s", rootProject.ext.versionName,
            rootProject.ext.testNum)
}

static int gitVersionCode() {
    def count = "git rev-list HEAD --count".execute().text.trim()
    return count.isInteger() ? count.toInteger() : 0
}

```

下面就根据场景来依次介绍对应的配置代码。

## 配置服务器版本

这里创建了三个 flavor，分别是 `offline 测试服`、`online 正式服`、`admin 超管服`。并且通过 `buildConfigField` 动态配置服务器的 URL 常量值到编译后自动生成的 `BuildConfig` 类中。

![自动生成的 BuildConfig 类][image-1]

图中可以看到，不止有 `DOMAIN_NAME` 常量值，还有一个 `FLAVOR` 常量。这个 `FLAVOR` 常量中的值是 `offline`，代表当前在 `offline` 这个版本上面。那怎么切换到其他的服务器呢？

![切换不同的变种版本][image-2]

点开左下角的 `Build Variants`, 可以自由切换当前运行的版本。需要在管理员包中开启一些高级的功能，可以判断 `FLAVOR` 的值是不是 `admin`，如果是的话就显示管理员的操作布局。当然必不可少的要对用户权限进行校验哦。

## 定制 `versionName`

大家看上图 `BuildConfig` 类中 `VERSION_NAME` 常量的值为 `2.0.2.0001-debug`，当前是测试服的 `debug` 包，所以 `versionName` 应该是正常的 2.0.2 版本后面拼上当前出包的序号 0001 ，再拼上 `debug` 的后缀，所以完整的版本号是 `2.0.2.0001-debug`。

看看不同服务器版本的 `VERSION_NAME` ：

* `offlineRelease` 版本为 `2.0.2.0001`
* `offlineDebug`   版本为 `2.0.2.0001-debug`
* `adminRelease` 版本为 `2.0.2-管理员`
* `adminDebug`   版本为 `2.0.2-管理员-debug`
* `onlineRelease` 版本为 `2.0.2`
* `onlineDebug`   版本为 `2.0.2-debug`

如果我们接口需要上传版本号给服务器呢？肯定不能直接上传这些定制化后的 `VERSION_NAME`，那么我们在 `Gradle` 中增加一个 `buildConfigField` 将原始的版本号存起来就好了。

```gradle
buildConfigField "String", "versionNumber", "\"${rootProject.ext.versionName}\""
```

## `versionCode` 自增

这里采用了主流的方式，使用 `git` 的 `commit` 次数作为 `versionCode` 的值。不用担心这个值会超过 `int` 的上限，你得敲烂多少键盘才能提交 2147483648次 `commit`。

```gradle
static int gitVersionCode() {
    def count = "git rev-list HEAD --count".execute().text.trim()
    return count.isInteger() ? count.toInteger() : 0
}
```

## 个性化渠道名

`Gradle` 多渠道打包的文章太多了，相关的基础我就不讲了。简单讲下本文相关的配置吧。

通过定义 `manifestPlaceholders` 键值对，在 `AndroidManifest.xml` 文件中使用占位符的方式动态输入 `UMENG_APPKEY` 和 `UMENG_CHANNEL` 。

然后在 `debug` 的 `buildType` 中修改渠道名为 `test`，在 `admin` 的 `Flavor` 中修改渠道名为 `admin`。如果选择 `adminDebug` 版本，则渠道名为 `test`，`buildType` 中的配置会覆盖掉 `Flavor` 中的配置。

由于我们线上使用第三方加固，所以多渠道包就交给第三方加固软件来生成了。

关于多渠道打包我还有两句话要说，以前使用 `Gradle` 进行多渠道打包，通过代码自定义修改 `apk` 文件的输出路径，`Android Studio` 编译的时候时不时的报一些文件找不到的错误，以前都是通过在 `Gradle` 文件中随便修改一点东西然后刷新一下 `Gradle` 文件来解决。现在我打包不修改输出路径，再也没遇到以前的那些问题了。

建议大家使用 `assemble` 命令来进行打包，比如我要出一个测试包使用 `./gradlew assembleOfflineRelease` 命令，`apk` 文件生成在 `/build/outputs/apk/` 目录下。直接执行 `assemble` 命令是编译 `Build Variants` 中的所有包，如果你要编译指定版本的包，直接在 `assemble` 命令后面拼上指定的 `Build Variant` 就好了。



## `debug` 包使用 `release` 签名

这个问题在 `eclipse` 时代，可以直接在设置里面配置 `debug` 签名文件为 `release` 的签名文件。
用 `Android Studio` 只需要在 `Gradle` 中配置就好了。

首先配置签名文件的信息：

```gradle
signingConfigs {
 	//在这里配置相关的签名信息
  keyStore {
    storeFile file("test.jks")
    storePassword "111111"
    keyAlias "test"
    keyPassword "111111"
  }
}
```

然后在 `buildTypes` 中设置签名信息：

```gradle
buildTypes {
  release {
    signingConfig signingConfigs.keyStore //设置签名文件
  }

  debug {
    signingConfig signingConfigs.keyStore  //设置签名文件
  }
}
```

## 日志开关

这个太简单了，不想单独列出来。不过上面场景里面提出来了，就简单一行代码展示吧。

```gradle
release {
	// 不显示Log
		buildConfigField "boolean", "LOG_DEBUG", "false"
}

debug {
	// 显示Log
	buildConfigField "boolean", "LOG_DEBUG", "true"
}
```

在日志工具类中使用 `BuildConfig` 类中的 `LOG_DEBUG` 常量来判断当前是否应该输出日志。

当然也可以用这个开关来控制开启严格模式等其他只适合在 `debug` 模式下开启的设置。

# 结语

唧唧歪歪说了这么多，在懂的人眼中自然很简单，在对 `gradle` 一点都不了解的人眼中就可以直接复制过去用了。当然我是不建议直接复制，毕竟需求稍微一改，你可能就束手无策了。建议大家还是以理解为主，掌握其原理自然一通百通。

<center>欢迎关注微信公众号：**大脑好饿**，更多干货等你来尝</center>
{% qnimg wx_qrcode.png title:公众号：大脑好饿  %}

[image-1]:	http://images.imliujun.com/static/images/Gradle/BuildConfig.png "BuildConfig"
[image-2]:	http://images.imliujun.com/static/images/Gradle/SwitchBuildVariants.png "BuildVariants"
