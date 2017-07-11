title: Android Studio 3.0 上 Gradle 改动
toc: true
comments: true
date: 2017-07-11 15:02:19
tags: Gradle
categories: Android
description:
---
上一篇文章：[使用 Gradle 对应用进行个性化定制](http://www.imliujun.com/gradle1.html) 中使用到了 `productFlavors`，有同学评论在 Android Studio 3.0 上编译不了。

[官方文档](https://developer.android.com/studio/build/gradle-plugin-3-0-0-migration.html)：

![官方说明](http://images.imliujun.com/static/images/Gradle/AndroidStudio3_update.png)

简单解释一下，`'com.android.tools.build:gradle:3.0.0-alpha5'` 插件 3.0.0 版本包含一个新的依赖机制，强制所有的 `flavor` 必须配置一个 `flavor dimension`。

在上一篇文章的基础上，稍作修改：

```gradle
//配置一个默认的 flavorDimensions
flavorDimensions "SERVER"
    productFlavors {
        offline {
            dimension "SERVER" //设置
            buildConfigField "String", "DOMAIN_NAME", "\"https://offline.domain.com/\""
            versionName getTestVersionName() //修改 versionName
        }

        online {
            dimension "SERVER"
            buildConfigField "String", "DOMAIN_NAME", "\"https://online.domain.com/\""
        }

        admin {
            dimension "SERVER"
            buildConfigField "String", "DOMAIN_NAME", "\"https://admin.domain.com/\""
            versionName rootProject.ext.APP1_VERSION_NAME + "-管理员" //修改 versionName
            manifestPlaceholders.UMENG_CHANNEL_VALUE = "admin" //修改渠道名
        }
    }
```

主要就是给 `flavor` 设置默认的 `Dimension` ，这样编译就没有问题了。

# 相关阅读

[使用 Gradle 对应用进行个性化定制](http://www.imliujun.com/gradle1.html)

[使用 Gradle 实现一套代码开发多个应用](http://www.imliujun.com/gradle3.html)


<center>欢迎关注微信公众号：**大脑好饿**，更多干货等你来尝</center>
![公众号：大脑好饿 ](http://images.imliujun.com/static/images/wx_qrcode.png)
