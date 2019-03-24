title: 提升效率——自动打包上传蒲公英
toc: true
comments: true
date: 2019-03-17 20:07:02
tags:
- 自动化
- Gradle
- 蒲公英
categories: Android
description: 一行命令自动打包上传蒲公英，输出二维码地址和版本号信息
---

> 时间是最宝贵的财富，我们的时间得用在刀刃上。

在文章 [使用 Gradle 对应用进行个性化定制](http://www.imliujun.com/gradle1.html) 中，我们能够针对一个应用的正式服、测试服、超管服等其他版本，进行个性化定制。所以经常会有测试跑过来说，帮我打个测试服的包吧、帮我打个正式服的包吧、帮我打个超管服的包吧。我打你一脸包😤😤🤯。

我们的目标是**一行命令**完成：

1. 自动编译 apk 上传到蒲公英
2. 上传完成后输出二维码地址和版本号
3. 支持多环境包上传



# 自动上传蒲公英

直接使用蒲公英的上传 [API](<https://www.pgyer.com/doc/view/api#uploadApp>) ，在 Gradle 中封装如下方法：

```groovy
private def uploadPGY(String filePath) {
    def stdout = new ByteArrayOutputStream()
    exec {
        executable = 'curl'
        args = ['-F', "file=@${filePath}", '-F', "_api_key=${rootProject.ext.pgy["apiKey"]}", rootProject.ext.pgy["uploadUrl"]]
        standardOutput = stdout
    }
    String output = stdout.toString()
    def parsedJson = new groovy.json.JsonSlurper().parseText(output)
    println parsedJson.data.buildQRCodeURL
    println "版本号：" + parsedJson.data.buildVersion
}
```

好了，执行以上方法就可以实现我们上面的两个目标啦。看不懂的我们继续，包教包会。

> [curl](http://curl.haxx.se/) 是一种命令行工具，作用是发出网络请求，然后得到和提取数据，显示在"标准输出"（stdout）上面。

我们使用 curl 命令来调用蒲公英的接口上传 apk 文件。将蒲公英的 `apiKey` 和 `uploadUrl` 放在 config.gradle 中进行统一的管理。

```groovy
ext{
    pgy = [apiKey   : "xxxxxxxxxxx",
           uploadUrl: "https://www.pgyer.com/apiv2/app/upload"]
}
```

`stdout` 变量是用来获取网络请求返回的数据流，我们解析成 JSON 对象后打印出二维码地址和版本号信息。如果你还想上传或者输出更多信息请查看蒲公英的上传 [API](<https://www.pgyer.com/doc/view/api#uploadApp>) 。

这个时候大家要问了，apk 的文件路径从哪来呢？



# 实现不同环境包上传

我们有正式环境、测试环境和超管环境，并且每个环境生成的 apk 文件名也不同，那么我们怎么获取到 apk 文件的路径呢？

## 统一文件命名规则

在`config.gradle` 中增加版本信息

```groovy
ext{
    pgy = [apiKey   : "xxxxxxxxxxx",
           uploadUrl: "https://www.pgyer.com/apiv2/app/upload"
          ]

    android = [compileSdkVersion: 28,
               buildToolsVersion: "28.0.3",
               minSdkVersion    : 16,
               targetSdkVersion : 28,
               versionCode      : 1,
               versionName      : "1.0.0"
              ]
}
```

在根目录的 `build.gradle` 中增加获取 `getApkName` 方法

```groovy
def getTestVersionName(String suffix) {
    def testVersion = "001"
    if (suffix == null || suffix.isEmpty()) {
        return String.format("%s.%s", rootProject.ext.android["versionName"], testVersion)
    } else {
        return String.format("%s.%s.%s", rootProject.ext.android["versionName"], testVersion, suffix)
    }
}

def getApkName(String versionName) {
    return String.format("我是一个包-v%s.apk", versionName)
}

```

在 application 工程的 `build.gradle` 中修改 apk 文件名

```groovy
productFlavors {
        offline {
            buildConfigField "String", "DOMAIN_NAME", "\"https://offline.domain.com/\""
            versionName getTestVersionName("offline") //修改 versionName
        }

        online {
            buildConfigField "String", "DOMAIN_NAME", "\"https://online.domain.com/\""
            versionName rootProject.ext.android["versionName"]
        }

        admin {
            buildConfigField "String", "DOMAIN_NAME", "\"https://admin.domain.com/\""
            versionName getTestVersionName("管理员") //修改 versionName
        }
}

android.applicationVariants.all { variant ->
    variant.outputs.all {
        outputFileName = getApkName(variant.versionName)
    }
}
```



## 实现多环境上传

```groovy
def offlineFile = "${projectDir.absolutePath}/build/outputs/apk/offline/release/${getApkName(getTestVersionName("offline"))}"
def adminFile = "${projectDir.absolutePath}/build/outputs/apk/admin/release/${getApkName(getTestVersionName("管理员"))}"
def onlineFile = "${projectDir.absolutePath}/build/outputs/apk/online/release/${getApkName(rootProject.ext.android["versionName"])}"


/**
 * 执行 “uploadOfflineApk” 命令自动打测试服包，并上传到蒲公英
 */
task uploadOfflineApk(dependsOn: 'assembleOfflineRelease') {
    group = "publish"
    doLast {
        uploadPGY(offlineFile)
    }
}

/**
 * 执行 “uploadOnlineApk” 命令自动打正式服包，并上传到蒲公英
 */
task uploadOnlineApk(dependsOn: 'assembleOnlineRelease') {
    group = "publish"
    doLast {
        uploadPGY(onlineFile)
    }
}

/**
 * 执行 “uploadAdminApk” 命令自动打超管服包，并上传到蒲公英
 */
task uploadAdminApk(dependsOn: 'assembleAdminRelease') {
    group = "publish"
    doLast {
        uploadPGY(adminFile)
    }
}
```

一行命令打3个包上传：`./gradlew uploadOfflineApk uploadOnlineApk uploadAdminApk`



# 下篇预告

我们的超管包是需要发给运营人员去使用的，防止泄露导致的安全风险，我们希望对超管包先进行加固然后再上传到蒲公英。



# 相关阅读

* [使用 Gradle 对应用进行个性化定制](http://www.imliujun.com/gradle1.html)
* [使用 Gradle 实现一套代码开发多个应用](http://www.imliujun.com/gradle3.html)


<center>欢迎关注微信公众号：**大脑好饿**，更多干货等你来尝</center>
![公众号：大脑好饿 ](http://images.imliujun.com/static/images/wx_qrcode.gif)
