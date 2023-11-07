# ffplay源码研究六（音视频同步部分)

## ffplay整体流程图

![](./res/ffplay/ffplay_framework.png)

便于理解，放一张图，本节主要分析的是音视频时钟同步部分。

首先要明白为啥要做音视频同步？经过上面音频和视频的输出可知道，它们的输出不在同一个线程，那么不一定会同时解出同一个pts的音频帧和视频帧，有时编码或封装的时候可能pts还是不连续的，如果不做处理就直接播放，就会出现声音和画面不同步，因此在进行音频和视频的播放时，需要对音频和视频的播放速度、播放时刻进行控制，以实现音频和视频保持同步。

在ffplay中有3种同步策略：音频为准、视频为准、外部时钟为准，由于人耳朵对声音变化的敏感度比视觉高，一般的同步策略都是以音频为准，即将视频同步到音频，对画面进行适当的丢帧或重复显示，以追赶或等待音频，在ffplay中默认也是以音频时钟为准进行同步。

```
typedef struct Clock {
    double pts;//时钟基础, 当前帧(待播放)显示时间戳，播放后,当前帧变成上一帧
    double pts_drift;//当前pts与当前系统时钟的差值, audio、video对于该值是独立的
    double last_updated;//最后一次更新的系统时钟
    double speed;//时钟速度控制，用于控制播放速度
    int serial;//播放序列，所谓播放序列就是一段连续的播放动作，一个seek操作会启动一段新的播放序列          
    int paused;//= 1 说明是暂停状态
    int *queue_serial; //指向packet_serial
} Clock;
```

### 音频时钟为主时钟

音频时钟设置首先位于sdl_audio_callback，由sdl进行调用。

```
static void sdl_audio_callback(void *opaque, Uint8 *stream, int len)
{
	audio_callback_time = av_gettime_relative(); // audio_callback_time为全局变量
	
	{ // audio_decode_frame函数里面
		audio_clock0 = is->audio_clock;
        /* update the audio clock with the pts */
        if (!isnan(af->pts))
            is->audio_clock = af->pts + (double) af->frame->nb_samples / af->frame->sample_rate;
        else
            is->audio_clock = NAN;
        is->audio_clock_serial = af->serial;
	}
	if (!isnan(is->audio_clock)) {
        set_clock_at(&is->audclk, is->audio_clock - (double)(2 * is->audio_hw_buf_size + is->audio_write_buf_size) / is->audio_tgt.bytes_per_sec, is->audio_clock_serial, audio_callback_time / 1000000.0);
        sync_clock_to_slave(&is->extclk, &is->audclk);
    }
}
```

视频时钟同步音频时钟

```
static void video_refresh(void *opaque, double *remaining_time)
{
	...
	 double last_duration, duration, delay;
     Frame *vp, *lastvp;

    /* dequeue the picture */
    lastvp = frame_queue_peek_last(&is->pictq); // 获取当前显示帧
    vp = frame_queue_peek(&is->pictq); // 获取待显示帧

    // 如果待显示帧序列和当前渲染序列不一致，则进行丢帧，seek之类操作
    if (vp->serial != is->videoq.serial) {
        frame_queue_next(&is->pictq);
        goto retry;
    }

    // 如果当前显示帧和待显示帧序列不一致，seek之类操作，新的播放序列重置当前时间
    if (lastvp->serial != vp->serial)
    is->frame_timer = av_gettime_relative() / 1000000.0;

    if (is->paused) // 暂停则继续渲染当前帧
    	goto display;

    /* compute nominal last_duration */
    // 计算相邻两帧间隔时长 pts相减，以及判断
    last_duration = vp_duration(is, lastvp, vp); 
    //计算上一帧lastvp还需要播放时间 判断同步类型，如果是video master则直接返回last_duration
    // 否则进行时钟同步 同步见下一章
    delay = compute_target_delay(last_duration, is);

    // 获取当前时间，如果当前time在渲染区间内，则进行remain time更新，供av_sleep
    time= av_gettime_relative()/1000000.0;
    if (time < is->frame_timer + delay) {
        *remaining_time = FFMIN(is->frame_timer + delay - time, *remaining_time);
        goto display;
    }

    // 更新frame_timer
    is->frame_timer += delay;
    if (delay > 0 && time - is->frame_timer > AV_SYNC_THRESHOLD_MAX)
    	is->frame_timer = time;

    SDL_LockMutex(is->pictq.mutex);
    if (!isnan(vp->pts))
    	update_video_pts(is, vp->pts, vp->serial); // 更新pts
    SDL_UnlockMutex(is->pictq.mutex);

    // 丢帧处理
    if (frame_queue_nb_remaining(&is->pictq) > 1) {
        Frame *nextvp = frame_queue_peek_next(&is->pictq);
        duration = vp_duration(is, vp, nextvp);
        if(!is->step && (framedrop>0 || (framedrop && get_master_sync_type(is) != AV_SYNC_VIDEO_MASTER)) && time > is->frame_timer + duration){
            is->frame_drops_late++;
            frame_queue_next(&is->pictq); // 丢帧处理
            goto retry;
        }
    }
	...
}
```

如上，视频以音频时钟为准进行同步，如果视频播放过快，则重复播放上一帧，来等待音频；如果视频播放过慢，则丢帧，追赶音频。



rame_timer，可以理解为帧显示时刻，如更新前，是上 ⼀帧lastvp的显示时刻；对于更新后（ is->frame_timer += delay ），则为当前帧vp显示时刻，上⼀帧显示时刻加上delay（还应显示多久（含帧本身时⻓））即为上⼀帧应结束显示的时刻。

time1：系统时刻小于lastvp结束显示的时刻（frame_timer+dealy），即虚线圆圈位置，此时应该继续显示lastvp；

time2：系统时刻大于lastvp的结束显示时刻，但小于vp的结束显示时刻（vp的显示时间开始于虚线圆圈，结束于黑色圆圈），此时既不重复显示lastvp，也不丢弃vp，即应显示vp；

time3：系统时刻大于vp结束显示时刻（黑色圆圈位置，也是nextvp预计的开始显示时刻），此时应该丢弃vp。

lastvp的显示时⻓delay是如何计算的，这在函数compute_target_delay中实现，compute_target_delay返回值越大，画面越慢，也就是上一帧显示的时间越长。

```
static double compute_target_delay(double delay, VideoState *is)
{
    double sync_threshold, diff = 0;
 
    /* update delay to follow master synchronisation source */
    if (get_master_sync_type(is) != AV_SYNC_VIDEO_MASTER) {
        /* if video is slave, we try to correct big delays by
           duplicating or deleting a frame */
        diff = get_clock(&is->vidclk) - get_master_clock(is);
 
        /* skip or repeat frame. We take into account the
           delay to compute the threshold. I still don't know
           if it is the best guess */
        sync_threshold = FFMAX(AV_SYNC_THRESHOLD_MIN, FFMIN(AV_SYNC_THRESHOLD_MAX, delay));
        if (!isnan(diff) && fabs(diff) < is->max_frame_duration) {
            if (diff <= -sync_threshold)
                delay = FFMAX(0, delay + diff);
            else if (diff >= sync_threshold && delay > AV_SYNC_FRAMEDUP_THRESHOLD)
                delay = delay + diff;
            else if (diff >= sync_threshold)
                delay = 2 * delay;
        }
    }
 
    av_log(NULL, AV_LOG_TRACE, "video: delay=%0.3f A-V=%f\n",delay, -diff);
 
    return delay;
}
```

　(1) diff<= -sync_threshold：视频播放较慢，需要适当丢帧；

　(2) diff>= sync_threshold && delay > AV_SYNC_FRAMEDUP_THRESHOLD：返回 delay + diff，由具体dif决定还要显示多久；

　(3) diff>= sync_threshold：视频播放较快，需要适当重复显示lastvp，返回2*delay，也就是2倍lastvp显示时长，也就是让lastvp再显示一帧的时长；

　(4) -sync_threshold< diff < sync_threshold：允许误差内，按frame duration去显示视频，返回delay。

 　由上可知，并不是每时每刻都在同步，而是有一个“准同步”的差值区域。

### 视频时钟为主时钟

以视频为准进行音频同步视频时，如果对音频使用一样的丢帧和重复显示方案，是不靠谱的，因为⾳频的连续性远⾼于视频，通过重复几百个样本或者丢弃几百个样本来达到同步，会在听觉有很明显的不连贯。ffplay采用的原理：通过做重采样补偿，音频变慢了，重采样后的样本就比正常的少，以赶紧播放下一帧；音频快了，重采样后的样本就比正常的增加，从而播放慢一些。

对于视频流程，video_refresh()-> update_video_pts() 按照着视频帧间隔去播放，并实时地重新矫正video时钟即可，主要在于audio的播放，其主要看audio_decode_frame：

```
static int audio_decode_frame(VideoState *is)
{
    .......................................
    //1.根据与video clock的差值，计算应该输出的样本数
    wanted_nb_samples = synchronize_audio(is, af->frame->nb_samples);
   //2.判断是否需要重新初始化重采样
    if (af->frame->format        != is->audio_src.fmt            ||
        dec_channel_layout       != is->audio_src.channel_layout ||
        af->frame->sample_rate   != is->audio_src.freq           ||
        (wanted_nb_samples       != af->frame->nb_samples && !is->swr_ctx)) {
        swr_free(&is->swr_ctx);
    //设置输出、输入目标参数
        is->swr_ctx = swr_alloc_set_opts(NULL,
                                         is->audio_tgt.channel_layout, is->audio_tgt.fmt, is->audio_tgt.freq,
                                         dec_channel_layout,           af->frame->format, af->frame->sample_rate,
                                         0, NULL);
        if (!is->swr_ctx || swr_init(is->swr_ctx) < 0) {
            av_log(NULL, AV_LOG_ERROR,
                   "Cannot create sample rate converter for conversion of %d Hz %s %d channels to %d Hz %s %d channels!\n",
                    af->frame->sample_rate, av_get_sample_fmt_name(af->frame->format), af->frame->channels,
                    is->audio_tgt.freq, av_get_sample_fmt_name(is->audio_tgt.fmt), is->audio_tgt.channels);
            swr_free(&is->swr_ctx);
            return -1;
        }
    //保存最新的参数
        is->audio_src.channel_layout = dec_channel_layout;
        is->audio_src.channels       = af->frame->channels;
        is->audio_src.freq = af->frame->sample_rate;
        is->audio_src.fmt = af->frame->format;
    }
    //3.重采样，利用重采样进行样本的添加或剔除
    if (is->swr_ctx) {
        const uint8_t **in = (const uint8_t **)af->frame->extended_data;
        uint8_t **out = &is->audio_buf1;
        int out_count = (int64_t)wanted_nb_samples * is->audio_tgt.freq / af->frame->sample_rate + 256;
        int out_size  = av_samples_get_buffer_size(NULL, is->audio_tgt.channels, out_count, is->audio_tgt.fmt, 0);
        int len2;
        if (out_size < 0) {
            av_log(NULL, AV_LOG_ERROR, "av_samples_get_buffer_size() failed\n");
            return -1;
        }
        if (wanted_nb_samples != af->frame->nb_samples) {
            if (swr_set_compensation(is->swr_ctx, (wanted_nb_samples - af->frame->nb_samples) * is->audio_tgt.freq / af->frame->sample_rate,
                                        wanted_nb_samples * is->audio_tgt.freq / af->frame->sample_rate) < 0) {
                av_log(NULL, AV_LOG_ERROR, "swr_set_compensation() failed\n");
                return -1;
            }
        }
        av_fast_malloc(&is->audio_buf1, &is->audio_buf1_size, out_size);
        if (!is->audio_buf1)
            return AVERROR(ENOMEM);
    //音频重采样：返回值是重采样后得到的音频数据中单个声道的样本数
        len2 = swr_convert(is->swr_ctx, out, out_count, in, af->frame->nb_samples);
        if (len2 < 0) {
            av_log(NULL, AV_LOG_ERROR, "swr_convert() failed\n");
            return -1;
        }
        if (len2 == out_count) {
            av_log(NULL, AV_LOG_WARNING, "audio buffer is probably too small\n");
            if (swr_init(is->swr_ctx) < 0)
                swr_free(&is->swr_ctx);
        }
        is->audio_buf = is->audio_buf1;
        resampled_data_size = len2 * is->audio_tgt.channels * av_get_bytes_per_sample(is->audio_tgt.fmt);
    } else {
    //不用重采样，则将指针指向frame中音频数据
        is->audio_buf = af->frame->data[0];
        resampled_data_size = data_size;
    }
    ...............................................
     
    return resampled_data_size;//返回audio_buf的数据大小
}
```

(1) 根据与video clock的差值，计算应该输出的样本数。由函数 synchronize_audio 完成，音频变慢了则样本数减少，音频变快了则样本数增加；

(2) 判断是否需要重采样：如果要输出的样本数与frame的样本数不相等，也就是需要适当减少或增加样本；

(3) 利⽤重采样库进行平滑地样本添加或剔除，在获知要调整的⽬标样本数 wanted_nb_samples 后，通过 swr_set_compensation 和 swr_convert 的函数组合完成重采样。

### 外部时钟为主时钟
