
[TOC]
#音乐播放器

#Basic
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
  
#开始使用

如何使用MediaPlayer播放音频或者视频资源可以参考官网[MediaPlayer类](https://developer.android.com/reference/android/media/MediaPlayer.html)，播放的控制类似于一个状态机，这里我还是觉得这个张图对整个播放状态描述比较直接。


![](mediaplayer_state_diagram.gif)


* 使用MediaPlayer时prepare的时候一般会用异步等待，避免ANR。如果是一些比较小的本地音频就没有这个必要了。
* 对于Error状态，必须要先reset，然后才能继续使用。
* 使用过程中我们只要记住一点MediaPlayer是基于状态的，进行下一个操作严格依赖上一个状态，否则会抛异常。
* MediaPlayer是非常消耗系统资源的，再不用的时候要及时释放。

#后台播放

在应用退出后依然想让APP在后台继续播放，音乐播放器都会有这种需求，这时候需要开始一个后台服务来处理播放，以下是后台播放需要了解和处理的情况。

##使用wake lock
当设备进入休眠状态时，系统会关掉一些不需要的特性，包括CPU和WIFI模块。然而当你的音乐在后台播放时，为了防止系统影响你的播放。你需要使用`wake locks`，`wake locks`是一种在系统处于空闲状态时告诉系统你需要一些特性。


> *Notice*: 要保证只有在需要的时候使用wake locks，因为这样可以减少对电池的伤害

为了在播放过程中保证CPU一直运行，在MediaPlayer初始化的时候调用`setWakeMode()`。MediaPlayer就会持有这种锁直到暂停或者停止播放。
```
mMediaPlayer = new MediaPlayer();
// ... other initialization here ...
mMediaPlayer.setWakeMode(getApplicationContext(), PowerManager.PARTIAL_WAKE_LOCK);
```

如果你需要使用WIFI，你也可以持有WifiLock，但是这时候需要手动去管理。
```
WifiLock wifiLock = ((WifiManager) getSystemService(Context.WIFI_SERVICE))
    .createWifiLock(WifiManager.WIFI_MODE_FULL, "mylock");

wifiLock.acquire();
```
当播放结束不在需要访问网络时，释放WifiLock
```
wifiLock.release();
```

##前台服务
为什么选择前台服务？是因为前台服务有着更高的等级，系统几乎不会杀死该服务，而这也是我们需要的。开启一个前台服务必须创建`Notification`，然后调用`startForeground()`方法。
```
String songName;
// assign the song name to songName
PendingIntent pi = PendingIntent.getActivity(getApplicationContext(), 0,
                new Intent(getApplicationContext(), MainActivity.class),
                PendingIntent.FLAG_UPDATE_CURRENT);
Notification notification = new Notification();
notification.tickerText = text;
notification.icon = R.drawable.play0;
notification.flags |= Notification.FLAG_ONGOING_EVENT;
notification.setLatestEventInfo(getApplicationContext(), "MusicPlayerSample",
                "Playing: " + songName, pi);
startForeground(NOTIFICATION_ID, notification);
```
你只能在你需要的时候维持前台状态，一旦不需要的时候要及时释放
```
stopForeground();
```

##处理audio焦点
对于一个音乐播放器，焦点的处理显得尤为重要。否则就是个坑。在播放音乐之前要先申请焦点，确认是否可以播放
```
AudioManager audioManager = (AudioManager) getSystemService(Context.AUDIO_SERVICE);
int result = audioManager.requestAudioFocus(this, AudioManager.STREAM_MUSIC,
    AudioManager.AUDIOFOCUS_GAIN);

if (result != AudioManager.AUDIOFOCUS_REQUEST_GRANTED) {
    // could not get audio focus.
}
```
在`requestAudioFocus()`方法中的第一个参数是`AudioManager.OnAudioFocusChangeListener`，在焦点发生改变的时候回调用`onAudioFocusChange()`，所以服务要实现该方法
```
class MyService extends Service
                implements AudioManager.OnAudioFocusChangeListener {
    // ....
    public void onAudioFocusChange(int focusChange) {
        // Do something based on focus change...
    }
}
```

`focusChange`参数会返回当前的焦点状态

* AUDIOFOCUS_GAIN 当前获取到焦点
* AUDIOFOCUS_LOSS 当前失去焦点，你需要停止当前音乐播放，并且释放资源
* AUDIOFOCUS_LOSS_TRANSIENT 当前失去焦点，你需要停止当前播放，可以不释放资源
* AUDIOFOCUS_LOSS_TRANSIENT_CAN_DUCK 当前失去焦点，你只需要将声音调低即可

```
public void onAudioFocusChange(int focusChange) {
    switch (focusChange) {
        case AudioManager.AUDIOFOCUS_GAIN:
            // resume playback
            if (mMediaPlayer == null) initMediaPlayer();
            else if (!mMediaPlayer.isPlaying()) mMediaPlayer.start();
            mMediaPlayer.setVolume(1.0f, 1.0f);
            break;

        case AudioManager.AUDIOFOCUS_LOSS:
            // Lost focus for an unbounded amount of time: stop playback and release media player
            if (mMediaPlayer.isPlaying()) mMediaPlayer.stop();
            mMediaPlayer.release();
            mMediaPlayer = null;
            break;

        case AudioManager.AUDIOFOCUS_LOSS_TRANSIENT:
            // Lost focus for a short time, but we have to stop
            // playback. We don't release the media player because playback
            // is likely to resume
            if (mMediaPlayer.isPlaying()) mMediaPlayer.pause();
            break;

        case AudioManager.AUDIOFOCUS_LOSS_TRANSIENT_CAN_DUCK:
            // Lost focus for a short time, but it's ok to keep playing
            // at an attenuated level
            if (mMediaPlayer.isPlaying()) mMediaPlayer.setVolume(0.1f, 0.1f);
            break;
    }
}
```

这里有涉及到一个`remote service` or `local service`，以下是我自己使用过程中的体会（以下都是基于Nexus4，OSVsersion 4.4）：

###Remote service
 这里会涉及几个问题：
 * 跨进程通讯<br/>
    跨进程通讯方式我就不讨论，可以用AIDL，Message。
 * 多个进程共享数据<br/>
    共享数据一般用的是 SharedPreferences 和 DataBase。<br/>
    如果是SharedPreferences，这需要将MODE设置为`MODE_MULTI_PROCESS`，但是你会发现这个模式已经被废弃掉了，所以被抛弃了；如果使用`DataBase`则需要使用`Observable`将自己的数据接口暴露出去。
 * 通过远程服务在应用退出或者从后台remove的时候可以继续播放，当然国产的手机remove的时候依然会kill掉所有相关的服务。（具体的效果可以依照网易云音乐来实现，我想他们应该会测试很多种情况，如果他都没办法继续播放就别折腾了）

总得来说用这种方式实现的话会进程通讯和数据同步需要写很多代码。

###Local Service

本地服务只需要保证APP从前台退出的时候依然播放，这里通过`startService()`是可以实现，唯一要处理的地方就是从最近打开应用列表REMOVE掉的时候，后台服务也会被kill掉。有兴趣的可以看看这个帖子[Foreground service killed when receiving broadcast after acitivty swiped away in task list](https://code.google.com/p/android/issues/detail?id=53313)。









