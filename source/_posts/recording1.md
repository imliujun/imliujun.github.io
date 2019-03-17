title: 线程安全引起的录音杂音电流音问题
toc: true
comments: true
date: 2018-03-22 00:29:24
tags:
- 录音
- AudioRecord
categories: Android
description:
---

前段时间写了一个录音模块，需求是：『录音的时候实时语音转文字，实时计算音量大小，实时进行 MP3 转码保存为文件』

首先进行需求分析，确定技术方案：

1. 使用 AudioRecord 进行录音，实时获取原始音频数据
2. 将音频数据传递给第三方语音转文字 SDK 进行处理
3. 对音频数据进行处理，计算出音量大小
4. 对音频数据进行 MP3 编码
5. 将编码后的数据写入 MP3 文件

整个业务流程如上，只不过我们为了效率和解耦，将每个处理逻辑独立开来使用多线程进行并发处理。

具体流程见下图：

{% qnimg Recording/RecordingFlowChart.jpeg title:录音流程图  %}

## 代码撸起来

下面的伪代码全部使用 `kotlin` 展示，不熟悉 `kotlin` 没关系，只需要关注具体的业务逻辑。

### 音频数据采集

首先创建一个线程给 AudioRecord 进行录音采集：

```kotlin
  val emitter: FlowableProcessor<ShortArray>
  val isRecording = AtomicBoolean()

  override fun run() {
    var buffer = ShortArray(bufferSize)
    while (isRecording.get()) {
      val readSize = audioRecord.read(buffer, 0, bufferSize)
      if (readSize > 0) {
        //将数据使用 RxJava 发送出去
        emitter.onNext(buffer)
      }
    }
  }
```

我们在子线程中读取到音频数据，并且通过 RxJava 将数据向下传递。**（用什么传递不重要，重要的是将数据传递给下一层去进行处理）**

### 对数据进行处理

外部接收 RxJava 的事件，对音频数据进行处理 **（再次提醒，不需要在意细节，主要关注业务流程）** ：

```kotlin
  // 使用 observeOnIo() 操作将线程切换到 IO 线程
  recorder?.start()?.observeOnIo()?.subscribe{ it:ShortArray ->
    //此时的代码和录音采集的代码分别执行在不同的线程上了

    //计算音量大小
    val volume = calVolume(it)

  }


  recorder?.start()?.observeOnIo()?.subscribe{ it:ShortArray ->
    //此时的代码和录音采集的代码分别执行在不同的线程上了

    //将 ShortArray 转换为 ByteArray
    var pcmBuffer: ByteArray = ...
    it.toByteArray(pcmBuffer)

    //语音转文字
    ...
  }

  recorder?.start()?.observeOnIo()?.subscribe{ it:ShortArray ->
    //此时的代码和录音采集的代码分别执行在不同的线程上了

    //进行 MP3 编码
    val encode = mp3Encode(it)
    if (encode != null && encode > 0) {
      // 将编码后的数据写入文件
      mp3Stream.write(mp3Buffer, 0, encode)
    }

  }

```

整个业务流程就是这样，我自己使用的手机和公司所有的测试机，试听录制出来的 MP3 文件都没有问题。

开开心心的打包，测试，上线。

然后你懂的，有些用户录制出现杂音、电流音、声音断断续续。😂😂

机智的同学可能通过标题已经猜到了问题的原因，但我当时没有手机进行问题复现，为了解决这个问题可是花了很大的功夫才定位到问题所在。

### 解决杂音问题

因为我们在录音采集时将数据读取到 `buffer` 对象中，然后将 `buffer` 对象通过 RxJava 向下传递，因为 RxJava 的下游都开启了异步线程去处理事件，那么在录音采集的死循环中不等当前的数据进行 MP3 编码完毕就对 `buffer` 对象写入新采集到的音频数据，这个时候 MP3 编码出来的音频数据就被污染了。

```kotlin
  val emitter: FlowableProcessor<ShortArray>
  val isRecording = AtomicBoolean()

  override fun run() {
    var buffer = ShortArray(bufferSize)
    while (isRecording.get()) {
      // 读取音频数据到 buffer 中
      val readSize = audioRecord.read(buffer, 0, bufferSize)
      if (readSize > 0) {
        //将 buffer 发送出去，因为下游是异步处理，所以执行完毕直接开始下次循环
        emitter.onNext(buffer)
      }
    }
  }
```

要解决这个问题很简单：

``` kotlin
  // 将 buffer 数据 copy 一份进行传递，这样就不会修改下游的数据了
  emitter.onNext(buffer.copyOf())
```

但是使用 copy 的方式会频繁的创建、销毁 ShortArray 对象，能不能优化一下呢？

我们可以使用对象池来管理 ShortArray，这样就不会频繁的进行创建、销毁操作。在 Android 的 support.v4 包中有一个 `Pools` 类实现了简单的对象池功能：

```kotlin
  val bufferPool = Pools.SynchronizedPool<ShortArray>(10)

  fun acquireBuffer(bufferSize: Int): ShortArray {
    var buffer = bufferPool.acquire()
    if (buffer == null || buffer.size != bufferSize) {
      buffer = ShortArray(bufferSize)
    }
    return buffer
  }

  fun releaseBuffer(shortArray: ShortArray) {
    try {
      bufferPool.release(shortArray)
    } catch (e: Exception) {
      Timber.e(e)
    }
  }

  override fun run() {
    while (isRecording.get()) {
      //通过对象池获取 buffer 对象
      val buffer = acquireBuffer(bufferSize)
      // 读取音频数据到 buffer 中
      val readSize = audioRecord.read(buffer, 0, bufferSize)
      if (readSize > 0) {
        //将 buffer 发送出去，下游处理完毕后调用 releaseBuffer 对 buffer 对象进行释放
        emitter.onNext(buffer)
      }
    }
  }

```

## 总结

很简单的一个多线程并发问题，但是当我们自己不能复现的时候，还是带来了很大的麻烦。
这种问题在编写 `emitter.onNext(buffer)` 这行代码的时候就应该要考虑到线程安全问题，并且我之前做直播截屏的时候也遇到过类似的问题，截取直播流的画面帧保存为图片，因为截屏的操作不会很频繁，当时是直接 copy 一份画面帧的数据保存为图片。

可是以前没有写博客记录这种小问题，导致遇到类似的问题尽量不记得了。所以这次记录下来😂😂。

<center>欢迎关注微信公众号：**大脑好饿**，更多干货等你来尝</center>
{% qnimg wx_qrcode.gif title:公众号：大脑好饿  %}
