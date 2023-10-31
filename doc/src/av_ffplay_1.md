# ffplay源码研究一

本文针对的是`ffmpeg-release-6.1`源码进行研究。

## 环境搭建

```
# 下载源码
git clone git@github.com:FFmpeg/FFmpeg.git
# 切换到release/6.1分支

```

## 目录结构



## 播放器原理

![](./res/ffplay/demux.webp)

1.对网络媒体数据流进行解封装得到一般的视频封装格式比如MP4等，如果是本地播放的媒体文件就不需要解协议。

2.对媒体文件进行解复用，得到未经过解码的视频、音频或者字幕流数据，在ffmpeg中得到的是AVPacket，在ffplay中会将数据放入Packet Queue之中

3.对字幕、音频和视频数据进行解码，分别得到字幕，PCM，YUV数据，在ffplay中会将数据放入Frame Queue之中。

4.不同数据体积不同解码速度不同，视频解码相对比较慢，如果解码完就立马播放就会出现音频和视频播放不一致的情况因此需要进行音画同步。分别有音频同步到视频，视频同步到音频，外部时钟三种。

5.将字幕，PCM，YUV数据进行播放。

## ffplay.c研究

```
```



