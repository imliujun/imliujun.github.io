title: 提升效率——自动加固并上传到蒲公英
toc: true
comments: true
date: 2019-03-21 16:55:42
tags:
- 自动化
- Gradle
- 蒲公英
- 自动加固
categories: Android
description: 一行命令自动加固多渠道包，并上传到蒲公英
---

在文章 [提升效率——自动打包上传蒲公英](http://www.imliujun.com/automation1.html) 的最后，我们留下了一个问题：

> 我们的超管包是需要发给运营人员去使用的，防止泄露导致的安全风险，我们希望对超管包先进行加固然后再上传到蒲公英。

我们的应用在发布的时候一般都需要进行加固和生成多渠道包，大家通常的做法应该是下载加固客户端，或者将 apk 文件上传到加固服务的管理后台进行加固，然后等着加固完成，再下载安装包文件。

再次引用我的名言：

> 时间是最宝贵的财富，我们的时间得用在刀刃上。

本文基于 [使用 Gradle 实现一套代码开发多个应用](<http://www.imliujun.com/gradle3.html>) 中的 Gradle 配置进行迭代开发，带领大家实现 `360加固` 的自动化 Gradle 脚本。

# 自动加固

我们的目标是全自动化，并且在每个团队成员的电脑上都能够实现一行命令执行，不需要做额外的配置。



## 自动下载 360 加固程序

完整的 `config.gradle` 配置：

```groovy
ext {
    //签名文件配置
    signing = [keyAlias     : 'xxxxx',
               keyPassword  : 'xxxxx',
               storeFile    : '../sign.keystore',
               storePassword: 'xxxxxx']

    //蒲公英配置
    pgy = [apiKey   : "xxxx",
           uploadUrl: "https://www.pgyer.com/apiv2/app/upload"]

    //360加固配置
    jiagu = [name             : 'xxxxx',
             password         : 'xxxxx',
             zipPath          : "../jiagu/360jiagu.zip",
             unzipPath        : "../jiagu/360jiagubao/",
             jarPath          : '../jiagu/360jiagubao/jiagu/jiagu.jar',
             channelConfigPath: '../jiagu/Channel.txt',
             jiagubao_mac     : "http://down.360safe.com/360Jiagu/360jiagubao_mac.zip",
             jiagubao_windows : "http://down.360safe.com/360Jiagu/360jiagubao_windows_64.zip",
    ]

    android = [compileSdkVersion: 28,
               minSdkVersion    : 19,
               targetSdkVersion : 28]

    //版本号管理
    APP1_VERSION_NAME = "2.0.2"
    APP1_TEST_NUM = "0001"
    APP2_VERSION_NAME = "1.0.5"
    APP2_TEST_NUM = "0005"
}

```



新建一个 `jiagu.gradle` 文件：

```groovy
import org.apache.tools.ant.taskdefs.condition.Os

def downloadUrl = Os.isFamily(Os.FAMILY_WINDOWS) ? rootProject.ext.jiagu["jiagubao_windows"] : rootProject.ext.jiagu["jiagubao_mac"]

def zipPath = rootProject.ext.jiagu["zipPath"]
def unzipPath = rootProject.ext.jiagu["unzipPath"]

task download360jiagu() {
    doFirst {
        //如果 Zip 文件不存在就进行下载
        File zipFile = file(zipPath)
        if (!zipFile.exists()) {
            if (!zipFile.parentFile.exists()) {
                zipFile.parentFile.mkdirs()
            }
            exec {
                executable = 'curl'
                args = ['-o', zipPath, downloadUrl]
            }
        }
    }
    doLast {
        //解压 Zip 文件
        ant.unzip(src: zipPath, dest: unzipPath, encoding: "GBK")
        //将解压后的文件开启读写权限，防止执行 Jar 文件没有权限执行
        exec {
            executable = 'chmod'
            args = ['-R', '777', unzipPath]
        }
    }
}
```

执行 `download360jiagu` 就可以自动下载并解压 360 的加固程序啦。



## 根据多渠道文件进行加固

```groovy
import org.apache.tools.ant.taskdefs.condition.Os

def downloadUrl = Os.isFamily(Os.FAMILY_WINDOWS) ? rootProject.ext.jiagu["jiagubao_windows"] : rootProject.ext.jiagu["jiagubao_mac"]

def zipPath = rootProject.ext.jiagu["zipPath"]
def unzipPath = rootProject.ext.jiagu["unzipPath"]


//加固后所有apk的保存路径
def APP1_OUTPUT_PATH = "jiagu/apk/app1/"

def APP1_APK_PATH = "${projectDir.absolutePath}/build/outputs/apk/app1Online/release/${getApkName(rootProject.ext.APP1_VERSION_NAME)}"

/**
 * 加固
 * @param config 配置加固可选项
 * @param apkPath 要加固的文件路径
 * @param outputPath 输出路径
 * @param automulpkg 是否自动生成多渠道包
 */
def jiaGu(String config, String apkPath, String outputPath, boolean automulpkg) {
    //首次使用必须先登录
    exec {
        executable = 'java'
        args = ['-jar', rootProject.ext.jiagu["jarPath"], '-login', rootProject.ext.jiagu["name"], rootProject.ext.jiagu["password"]]
    }
    //升级到最新版本
    exec {
        executable = 'java'
        args = ['-jar', rootProject.ext.jiagu["jarPath"], '-update']
    }
    //显示当前版本号
    exec {
        executable = 'java'
        args = ['-jar', rootProject.ext.jiagu["jarPath"], '-version']
    }

    //导入签名信息
    exec {
        executable = 'java'
        args = ['-jar', rootProject.ext.jiagu["jarPath"], '-importsign',
                rootProject.ext.signing["storeFile"],
                rootProject.ext.signing["storePassword"],
                rootProject.ext.signing["keyAlias"],
                rootProject.ext.signing["keyPassword"]]
    }

    //配置加固可选项
    exec {
        executable = 'java'
        args = ['-jar', rootProject.ext.jiagu["jarPath"], '-config', config]
    }

    //加固命令
    def jiaGuArgs
    if (automulpkg) {
        jiaGuArgs = ['-jar', rootProject.ext.jiagu["jarPath"], '-jiagu',
                     apkPath,
                     outputPath,
                     '-autosign',
                     '-automulpkg',
                     '-pkgparam',
                     rootProject.ext.jiagu["channelConfigPath"]
        ]
    } else {
        jiaGuArgs = ['-jar', rootProject.ext.jiagu["jarPath"], '-jiagu',
                     apkPath,
                     outputPath,
                     '-autosign'
        ]
    }
    exec {
        executable = 'java'
        args = jiaGuArgs
    }
    println "加固的文件路径：${apkPath}"
    println "加固后的文件路径：${outputPath}"
}


/**
 * App1
 * 根据多渠道文件进行加固
 * 执行命令：./gradlew releaseApp1
 */
task releaseApp1(dependsOn: 'assembleApp1OnlineRelease') {
    doFirst {
        //判断加固程序是否存在，不存在则进行下载
        File jarFile = file(rootProject.ext.jiagu["jarPath"])
        if (!jarFile.exists()) {
            download360jiagu.execute()
        }
    }
    group = "publish"
    doLast {
        File apkOutputFile = new File(APP1_OUTPUT_PATH, getCurTime())
        checkOutputDir(apkOutputFile)
        File apkFile = file(APP1_APK_PATH)
        if (!apkFile.exists()) {
            println("apk file is not exists：" + apkFile.absolutePath)
            return
        }
        jiaGu("-", apkFile.absolutePath, apkOutputFile.absolutePath, true)
    }
}

private static void checkOutputDir(File apkOutputFile) {
    if (apkOutputFile.exists()) {
        File[] files = apkOutputFile.listFiles()
        if (files != null) {
            for (File file : files) {
                file.delete()
            }
        }
    } else {
        apkOutputFile.mkdirs()
    }
}


static def getCurTime() {
    return new Date().format("yyyy-MM-dd HH:mm:ss")
}
```

现在我们只需要在命令行执行 `./gradlew releaseApp1` 就可以静待输出了。

在根目录的 `jiagu` 文件夹中创建`Channel.txt`文件，在其中可以配置你需要的多渠道信息。

如果需要配置更多的加固选项，可以在 `jiagu/360jiagubao/jiagu/help.txt`中查看所有的加固命令。



# 加固超管包上传蒲公英

我们的超管包不需要上传应用商店，直接加固上传到蒲公英，然后发送二维码给管理员下载安装。我们把自动加固和自动上传蒲公英整合到一起。

在`jiagu.gradle`中添加单独加固超管包的方法：

```groovy

def APP1_ADMIN_OUTPUT_PATH = "jiagu/apk/app1Admin/"

def APP1_ADMIN_APK_PATH = "${projectDir.absolutePath}/build/outputs/apk/app1Admin/release/${getApkName(getTestVersionName("管理员"))}"


/**
 * 加固超管服包
 * 执行命令：./gradlew jiaGuApp1Admin
 */
task jiaGuApp1Admin(dependsOn: 'assembleApp1AdminRelease') {
    doFirst {
        File jarFile = file(rootProject.ext.jiagu["jarPath"])
        if (!jarFile.exists()) {
            download360jiagu.execute()
        }
    }
    group = "publish"
    doLast {
        File apkOutputFile = new File(APP1_ADMIN_OUTPUT_PATH)
        checkOutputDir(apkOutputFile)
        File apkFile = file(APP1_ADMIN_APK_PATH)
        if (!apkFile.exists()) {
            println("apk file is not exists：" + apkFile.absolutePath)
            return
        }
        jiaGu("-", apkFile.absolutePath, apkOutputFile.absolutePath, false)
    }
}
```

修改蒲公英上传方法：

```groovy
def app1AdminFileDir = "${projectDir.parent}/jiagu/apk/app2Admin/"

/**
 * 执行 “uploadApp1Admin” 命令自动打超管服包，并上传到蒲公英
 */
task uploadApp1Admin(dependsOn: 'jiaGuApp1Admin') {
    group = "publish"

    doLast {
        File dir = new File(app1AdminFileDir)
        if (!dir.exists()) {
            println "Alpha dir not exists：" + dir.path
            return
        }
        File[] files = dir.listFiles(new FileFilter() {
            @Override
            boolean accept(File file) {
                return file.isFile() && file.name.endsWith(".apk")
            }
        })
        if (files == null || files.size() == 0) {
            println "files == null ||  files.size() == 0"
            return
        }
        File apkFile = files[0]

        uploadPGY(apkFile.path)
    }
}
```

在命令行执行 `./gradlew uploadApp1Admin` 就可以静待二维码地址输出。



# 总结

如果你不喜欢执行命令行，我们只点一下鼠标也可以执行自动化命令：

![Gradle 命令](http://images.imliujun.com/static/images/automation/gralde_publish.png)



> demo地址：https://github.com/imliujun/GradleTest



# 相关阅读

* [提升效率——自动打包上传蒲公英](http://www.imliujun.com/automation1.html)
* [使用 Gradle 实现一套代码开发多个应用](http://www.imliujun.com/gradle3.html)


<center>欢迎关注微信公众号：**大脑好饿**，更多干货等你来尝</center>
![公众号：大脑好饿 ](http://images.imliujun.com/static/images/wx_qrcode.gif)
