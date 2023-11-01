# ffplay源码研究二

## ffplay整体流程图

![](./res/ffplay/ffplay_framework.png)

如上图，ffplay主要分为`read_thread`,`audio_thread`,`video_thread`,`video_refresh`,音频与视频之间同步以及在图中未画的event_loop部分。下面将对这些部分进行分析。

### 初始化

