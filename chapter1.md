# 音乐播放器

#初体验

说道音乐播放就涉及到两个重要的类`MediaPlayer`、`AuidoManager`。
* [MediaPlayer](https://developer.android.com/reference/android/media/MediaPlayer.html)  这个类提供了核心的播放音频和视频的API
* [AudioManager](https://developer.android.com/reference/android/media/AudioManager.html)  这个类主要管理设备上audio资源和audio的输出

这两个类的关系有点类似Soldier和Commander的关系，soldier只负责打战，至于打哪里，什么时候开火都需要Commander来统一调度。


#清单文件配置
在使用MediaPlayer时，用到一些featrue，需要在清单文件中声明。

* **Internet Permission** 如果播放的是网络资源时需要。
  ```
  <uses-permission android:name="android.permission.INTERNET" />
  ```
* **Wake Lock Permission** 如果需要应用在屏幕变暗或者CPU睡眠的过程中继续运行，可以调 用*MediaPlayer.setScreenOnWilePlaying()* 或者 *MediaPlayer.setWakeMode()*方法，这时就需要这个权限。
  ```
  <uses-permission android:name="android.permission.WAKE_LOCK" />
  ```
  
  



