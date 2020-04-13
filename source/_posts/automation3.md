title: 提升效率——配置 Jenkins 自动加固签名多渠道包
toc: true
comments: true
date: 2020-04-07 01:59:20
tags:
- 自动化
- Jenkins
- 自动加固
- 多渠道包
- Android 签名
categories: Android
description: 一行命令自动打包上传蒲公英，输出二维码地址和版本号信息
---

前两篇文章将我们的打包加固和上传内测平台进行了自动化，节省了开发人员的时间。

但是还有一些问题并没有解决，比如我们正在吭哧吭哧的写代码，测试突然来让你给他打个包，必须要暂停当前的工作，去对应分支执行打包命令，打包期间也只能等着，不能继续写代码。

这时，Jenkins 服务器就派上用场了，我们可以将打包命令在 Jenkins 服务器进行配置，让测试人员自己去选择想要的分支版本进行打包，同时提升双方的效率。

Jenkins 相关的知识网上有很多文章，请大家自行搜索了解，本文只讲解在 Jenkins 上配置以下几点内容：

- 使用梆梆加固命令行加固
- 对加固后的包进行签名
- 使用 VasDolly 进行快速多渠道打包
- 上传安装包到蒲公英

# 配置 Gradle 脚本

由于将 360 加固替换为梆梆加固，而梆梆加固的命令行工具不支持自动签名和本地多渠道包生成，所以增加了重新签名和 VasDolly 多渠道打包的命令。

## 加固脚本

完整的 `config.gradle` 配置：

```groovy
ext {
    curGitCommit = rootProject.properties['GIT_COMMIT'] == null ? getGitCommit() : rootProject.properties['GIT_COMMIT']
    curTime = rootProject.properties["BUILD_TIMESTAMP"] == null ? getCurTime() : rootProject.properties["BUILD_TIMESTAMP"]
    backupPath = "../buildBackup/"

    if (rootProject.properties["isJenkins"].toBoolean()) {
        androidHome = rootProject.properties['ANDROID_HOME']
    } else {
        Properties properties = new Properties()
        properties.load(project.rootProject.file('local.properties').newDataInputStream())
        androidHome = properties.getProperty('sdk.dir')
    }

    //签名文件配置
    signing = [keyAlias     : 'xxxxx',
               keyPassword  : 'xxxxx',
               storeFile    : '../sign.keystore',
               storePassword: 'xxxxxx']

    //蒲公英配置
    pgy = [apiKey   : "xxxx",
           uploadUrl: "https://www.pgyer.com/apiv2/app/upload"]

    //加固配置
    jiagu = [
            app1Online       : "${backupPath}${curGitCommit}/${curTime}/app1Online_jiagu/",
            app1Admin        : "${backupPath}${curGitCommit}/${curTime}/app1Admin_jiagu/",
            app2Online       : "${backupPath}${curGitCommit}/${curTime}/app2Online_jiagu/",
            app2Admin        : "${backupPath}${curGitCommit}/${curTime}/app2Admin_jiagu/",
            channelConfigPath: '../jiagu/Channel.txt',
            VasDollyJar      : "../jiagu/VasDolly.jar",
            banbanJar        : "../jiagu/secapi.jar",
            banbanName       : rootProject.properties['banbanName'] == null ? "banbanName" : rootProject.properties['banbanName'],
            banbanApiKey     : rootProject.properties['banbanApiKey'] == null ? "banbanApiKey" : rootProject.properties['banbanApiKey'],
            banbanSecretKey  : rootProject.properties['banbanSecretKey'] == null ? "banbanSecretKey" : rootProject.properties['banbanSecretKey'],
            banbanIp         : rootProject.properties['banbanIp'] == null ? "banbanIp" : rootProject.properties['banbanIp'],
    ]

    android = [compileSdkVersion: 28,
               buildToolsVersion: "29.0.3",
               minSdkVersion    : 19,
               targetSdkVersion : 28]

    apksignerDir = "${androidHome}/build-tools/${android['buildToolsVersion']}/"

    //版本号管理
    APP1_VERSION_NAME = "2.0.2"
    APP1_TEST_NUM = "0001"
    APP2_VERSION_NAME = "1.0.5"
    APP2_TEST_NUM = "0005"

    dependencies = [
            "androidx-appcompat"       : "androidx.appcompat:appcompat:1.1.0",
            "androidx-constraintlayout": "androidx.constraintlayout:constraintlayout:1.1.3",
            "VasDolly-helper"          : "com.leon.channel:helper:2.0.3",
    ]
}


static def getCurTime() {
    return new Date().format("yyyy-MM-dd_HH-mm-ss")
}


static def getGitCommit() {
    def exceptionStr = "git rev-parse HEAD 执行失败"
    try {
        def cmd = 'git rev-parse HEAD'
        def gitCommit = cmd.execute().text.trim()
        if (gitCommit.isEmpty()) {
            throw new GradleException(exceptionStr)
        }
        return gitCommit
    } catch (Exception e) {
        throw new GradleException(exceptionStr, e)
    }

}
```

在`jiagu.gradle`中实现加固签名和多渠道包逻辑：

```groovy
import org.apache.tools.ant.taskdefs.condition.Os


/**
 * 加固
 * @param config 配置加固策略
 * @param apkPath 要加固的文件路径
 * @param outputPath 输出路径
 * @param channelName 渠道名，为 Null 时读取 rootProject.ext.jiagu["channelConfigPath"] 多渠道配置文件
 */
def jiaGu(String config, String apkPath, String outputPath, String channelName) {
    println("开始加固 \nconfig：$config \napkPath:$apkPath \noutputPath:$outputPath \nchannelName:$channelName")
    exec {
        executable = 'java'
        args = ['-jar', rootProject.ext.jiagu["banbanJar"],
                '-i', rootProject.ext.jiagu["banbanIp"],
                '-u', rootProject.ext.jiagu["banbanName"],
                '-a', rootProject.ext.jiagu["banbanApiKey"],
                '-c', rootProject.ext.jiagu["banbanSecretKey"],
                '-f', '0',
                '-t', config,
                '-p', apkPath,
                '-d', outputPath,
        ]
    }

    println "加固的文件路径：${apkPath}"
    println "加固后的文件路径：${outputPath}"

    def signApkPath = apkSigner(outputPath)

    generatingMultipleChannels(signApkPath, outputPath, channelName)

  	//在 Jenkins 上运行时，将构建产物备份到硬盘根目录，
    if (isJenkins.toBoolean()) {
        String sourceDir = file(outputPath).parentFile.absolutePath
        String destinationDir = "/Users/buildBackup/${rootProject.properties["JOB_NAME"]}/${rootProject.ext.curGitCommit}/${rootProject.ext.curTime}/"
        new groovy.util.AntBuilder().copy(todir: destinationDir) {
            fileset(dir: sourceDir)
        }
        println "将构建产物备份到：${destinationDir}"
    }
}


def apkSigner(String apkDir) {
    def apkFile = getApkFile(apkDir)

    if (apkFile == null || !apkFile.exists()) {
        throw GradleException("加固后的 apk 文件不存在")
    }
    def signDir = new File(apkDir, 'sign')
    if (!signDir.exists()) {
        signDir.mkdirs()
    }
    def outputApkPath = new File(signDir, apkFile.name)

    def storeFile = rootProject.ext.signing["storeFile"]
    def storeFilePath = file(storeFile).absolutePath
    //兼容 Windows、Mac、Linux 系统
    def execuTable = Os.isFamily(Os.FAMILY_WINDOWS) ? 'cmd' : 'sh'
    def apksigner = Os.isFamily(Os.FAMILY_WINDOWS) ? 'apksigner.bat' : './apksigner'

    println("对加固包进行签名")

    def parameter = [apksigner, 'sign',
                     '--ks', storeFilePath,
                     '--ks-key-alias', rootProject.ext.signing["keyAlias"],
                     '--ks-pass', "pass:${rootProject.ext.signing["storePassword"]}",
                     '--key-pass', "pass:${rootProject.ext.signing["keyPassword"]}",
                     '--out', outputApkPath, apkFile.absolutePath]
    if (Os.isFamily(Os.FAMILY_WINDOWS)) {
        parameter.add(0, '/c')
    }
    exec {
        workingDir = rootProject.ext.apksignerDir
        executable = execuTable
        args = parameter
    }
    println "apk 签名成功：" + outputApkPath
    return outputApkPath
}

/**
 * 生成多渠道包
 * @param signApkPath 基准包
 * @param outputDir 输出目录
 * @param channelName 渠道名，为 Null 时读取 rootProject.ext.jiagu["channelConfigPath"] 多渠道配置文件
 */
private void generatingMultipleChannels(signApkPath, outputDir, channelName) {
    def channelDir = new File(outputDir, 'channels')
    if (!channelDir.exists()) {
        channelDir.mkdirs()
    }
    println("生成多渠道包")
    def channel = channelName == null ? rootProject.ext.jiagu["channelConfigPath"] : channelName
    exec {
        executable = 'java'
        args = ['-jar', rootProject.ext.jiagu["VasDollyJar"],
                'put', '-c', channel,
                signApkPath, channelDir.absolutePath
        ]
    }
    println("生成多渠道包完毕")
}

def getApkPath(String flavor) {
    if ("app1Online" == flavor) {
        return "${projectDir.absolutePath}/build/outputs/apk/production/release/${getApkName(rootProject.ext.android["versionName"])}"
    } else {
        return "${projectDir.absolutePath}/build/outputs/apk/${flavor}/release/${getApkName(getTestVersionName(flavor))}"
    }
}

private def getApkFile(String fileDir) {
    def dir = file(fileDir)
    if (!dir.exists()) {
        println "dir not exists：" + dir.path
        return null
    }
    File[] files = dir.listFiles(new FileFilter() {
        @Override
        boolean accept(File file) {
            return file.isFile() && file.name.endsWith(".apk")
        }
    })
    if (files == null || files.size() == 0) {
        println "files == null ||  files.size() == 0"
        return null
    }
    return files[0]
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

/**
 * App1
 * 根据多渠道文件进行加固
 * 执行命令：./gradlew releaseApp1
 */
task releaseApp1(dependsOn: ['assembleApp1OnlineRelease']) {
    group = "publish"
    doLast {
        def apkOutputFile = file(rootProject.ext.jiagu["app1OnlineOutputPath"])
        checkOutPutDir(apkOutputFile)
        def apkFile = file(getApkPath("App1Online"))
        if (!apkFile.exists()) {
            println("apk file is not exists：" + apkFile.absolutePath)
            return
        }
        jiaGu("1", apkFile.absolutePath, apkOutputFile.absolutePath, null)
    }
}
```

我们只需要在命令行执行 `./gradlew releaseApp1` 就会执行编译—>加固—>签名—>生成多渠道包

上传蒲公英的代码和前文一样，只需要改动一下 apk 路径，取生成的渠道包进行上传。

# Jenkins 配置

首先我们创建好任务，配置 task 命令参数：

![Jenkins project parameterized](http://images.imliujun.com/static/images/automation/Jenkins1.jpg)

再配置 Gradle 插件执行上面选择的命令，并配置 Project properties：

![Jenkins Gradle 插件配置](http://images.imliujun.com/static/images/automation/Jenkins2.jpg)

如果执行上传蒲公英的任务，我们希望直接显示蒲公英的二维码，增加如下`Set build description`配置：

![Jenkins build description](http://images.imliujun.com/static/images/automation/Jenkins3.png)

我们保存配置，执行任务：

![Build with Parameters](http://images.imliujun.com/static/images/automation/Jenkins5.png)执行成功后的效果：

![Jenkins build description](http://images.imliujun.com/static/images/automation/Jenkins4.png)

# 总结

全文没有做太多讲解，直接上代码上配置图片，简单粗暴，如果 Gradle 部分有疑惑的建议看看前面几篇文章。

对于 Jenkins 的配置本文只是一笔带过，相信大家简单了解一下相关的概念和原理就能快速上手，基本流程跑通以后可以举一反三实现更多功能。

> 最后上 demo 地址：https://github.com/imliujun/GradleTest/tree/Jenkins

# 相关阅读

- [提升效率——自动加固并上传到蒲公英](http://www.imliujun.com/automation2.html)
- [使用 Gradle 实现一套代码开发多个应用](http://www.imliujun.com/gradle3.html)

<center>欢迎关注微信公众号：**大脑好饿**，更多干货等你来尝</center>

![公众号：大脑好饿 ](http://images.imliujun.com/static/images/wx_qrcode.gif)
