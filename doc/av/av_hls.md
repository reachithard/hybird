# 流媒体协议之HLS协议

[TOC]

## HLS概述

作为**Apple**提出的一种基于**HTTP**的协议,**HLS(HTTP Live Streaming)用于解决实时音视频流的传输。尤其是在移动端,由于iOS/H5**不支持**flash**,使得**HLS**成了移动端实时视频流传输的首选。**HLS**经常用在直播领域,一些国内的直播云通常用**HLS**拉流。

![](.\res\HLS框架.png)

- **流分割器(Stream Segmenter)负责将编码器输出的MPEG-2 TS**流分割为一系列**连续的、长度均等的**小**TS**文件,并依次发送至内容分发组件中的**Web**服务器进行存储。
- 为了跟踪播放过程中媒体文件的可用性和当前位置,流分割器还需创建一个含有指向这些小**TS**文件指针的索引文件,同样放置于**Web**服务器之中。m3u8文件
- **TS**文件中必须仅包含一个**MPEG-2**节目,在每个文件的开头应包含一个**节目关联表(PAT)和一个节目映射表(PMT)**,包含视频的文件中还必须含有至少一个**关键帧和其他足够信息(如序列头)用以完成解码器的初始化。索引文件采用扩展的M3U**播放列表格式,后缀名为 **.m3u8** 。

## HLS之M3U8

m3u8为索引文件。

|        配置项         |                             作用                             |
| :-------------------: | :----------------------------------------------------------: |
|        #EXTM3U        |         每个 M3U 文件第一行必须是这个 tag,起标示作用         |
|    #EXT-X-VERSION     |        该属性可以没有,目前主要是 version 3,最新的是 7        |
| #EXT-X-MEDIA-SEQUENCE | 每一个 media URI 在 PlayList 中只有唯一的序号,相邻之间序号+1,一个 media URI 并不是必须要包含的,如果没有,默认为 0 |
| #EXT-X-TARGETDURATION | 所有切片的最大时长。有些 Apple 设备这个参数不正确会无法播放。SRS 会自动计算出 ts 文件的最大时长,然后更新 m3u8 时会自动更新这个值。用户不必自己配置 |
|        #EXTINF        | ts 切片的实际时长,SRS 提供配置项 hls_fragment,但实际上的 ts 时长还受 gop 影响 |
|     ts 文件的数目     | SRS 可配置 hls_window(单位是秒,不是数量),指定 m3u8 中保存多少个切片。譬如,每个 ts 切片为 10 秒,窗口为 60 秒,那么 m3u8 中最多保存 6 个 ts 切片,SRS 会自动清理旧的切片 |
|    livestream-0.ts    | SRS 会自动维护 ts 切片的文件名,在编码器重推之后,这个编号会继续增长,保证流的连续性。直到 SRS 重启,这个编号才重置为 0 |

## HLS之TS

### ts，pes，es关系图



![](C:\workspace\hybird\doc\av\res\TS文件分层.png)

* **ts包为188字节。**
* ts包包含header+adaptation field+payload，header用于描述Payload中数据的类型，payload既可以是PSI(Program Specific Information)元数据 也可以是音视频数据
* PSI/SI数据种类做了精简 只保留了两种最基本的PSI数据，也就是PAT(Program Association Table)和PMT(Program Map Table)，其中PAT指明PMT使用的PID，PMT指明音视频流的pid

### Ts header

![](.\res\ts_header.png)

* 由上可知，假设adaptation为x，则`4+x+payload length = 188`, 当adaptation_filed_control为11B时，有该公式。

* pid为payload的数据类型，比如0x00为元数据信息。

以下为各字段信息：

![](.\res\ts_header_desc.png)

适配域如下：

![](.\res\adaption_field.png)

适配域含义如下：

![](.\res\adaption_field_desc.png)

### PAT和PMT的作用：

* PAT 和 PMT 表是没有 adaptation field 的，不够的⻓度直接补 0xff 即可。

* 播放器播放TS流的基本过程 ：解析TS流时，首选获得PAT信息，之后通过PAT找到PMT，在通过PMT找到音视频流，PAT数据的PID是固定值0x00解析TS流时，首先需要对每个传过来的TS包进行遍历，直到找到TS Header中的PID值为0x00的包，然后从Payload中找到PAT数据，再找到流。**注TS Header中adaptation_field_control域指明存在adaption_field域，则在Payload中要跳过该域；另外，如果在TS Header中payload_unit_start_indicator置位，则说明真实数据之前还有一个字节的指针域，跳过这个域才能拿到真正的数据**
* 视频流和⾳频流都需要加 adaptation field，通常加在⼀个帧的第⼀个 ts包和最后⼀个 ts 包⾥，中间的 ts 包不加。

#### PAT格式如图

![](C:\workspace\hybird\doc\av\res\pat.png)

开始循环和结束循环的意思为中间字段为一组按组内顺序重复。

#### PMT格式如图

![](C:\workspace\hybird\doc\av\res\pmt.png)

开始循环和结束循环的意思为中间字段为一组按组内顺序重复。

### pes

![](.\res\pes_packet.png)

* PES 由Base Header，Optional Header以及Payload组成。

* **Base Header 分为3个域，分别为start_code 用于标识PES包的开始，stream_id用于标识是音频流还是视频流**

* stream id 0xCD~0xDF标识音频 0xE0~0xEF标识视频，packet_length PES的长度，这个长度是从packet_length 字段后的第一个字节开始算起，此值为0表示长度不受限制

![](.\res\pes_base_header.png)

#### pes optional header

![](.\res\pes_option_header.png)

header_info_length （1字节）标识了Optional Header 中是否有信息区域，如果值不为0的话说明后面还跟着信息区。而信息区中的内容就是根据前面Optional Header的几个标志位来决定 比如是否包含PTS DTS

###### 点播视频 dts 算法：

dts = 初始值 + 90000 / video_frame_rate ，初始值任意，但建议不为 0，video_frame_rate 就是帧率，⽐如 23、30。

pts 和 dts 是 以  time scale 为 单 位 的 ， 1s = 90000 time scale ， ⼀ 帧 就 应 该 是90000/video_frame_rate 个 timescale。

⽤⼀帧的 timescale 除以采样频率就可以转换为⼀帧的播放时⻓。

###### 点播⾳频 dts 算法：

dts = 初 始 值 + (90000 * audio_samples_per_frame) / audio_sample_rate ，audio_samples_per_frame 这 个 值 与 编 解 码 相 关 ， aac 取 值 1024 ， mp3 取 值 1158 ，audio_sample_rate 是采样率，⽐如 24000、41000. AAC ⼀般解码出来是每声道 1024 个 sample，也就是说⼀帧的时⻓为 1024/sample_rate 秒。所以每⼀帧时间戳依次 0,1024/sample_rate, ...,1024*n/sample_rate 秒 。

注：直播视频的 dts 和 pts 应该直接⽤直播数据流中的时间，不应该按公式计算。

## SRS的HLS源码解析

