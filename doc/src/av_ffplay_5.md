# ffplay源码研究四（音视频播放部分)

## ffplay整体流程图

![](./res/ffplay/ffplay_framework.png)

便于理解，放一张图，本节主要分析的是音视频播放部分，即sdl_audio_callback和video_refresh部分。

### sdl_audio_callback(流程图右上角部分，在stream_component_open部分，对于音频有audio_open)

```
static int audio_open(void *opaque, AVChannelLayout *wanted_channel_layout, int wanted_sample_rate, struct AudioParams *audio_hw_params)
{
	... // 省略参数等设置
    wanted_spec.callback = sdl_audio_callback; // 这里设置回调，由SDL触发
    wanted_spec.userdata = opaque; // opaque即为VideoState
    while (!(audio_dev = SDL_OpenAudioDevice(NULL, 0, &wanted_spec, &spec, SDL_AUDIO_ALLOW_FREQUENCY_CHANGE | SDL_AUDIO_ALLOW_CHANNELS_CHANGE))) {
        ... // 省略错误逻辑判断
    }
    ... // 省略参数等设置
}
```

```
/* prepare a new audio buffer */
static void sdl_audio_callback(void *opaque, Uint8 *stream, int len)
{
    VideoState *is = opaque;
    int audio_size, len1;

    audio_callback_time = av_gettime_relative(); // 音视频同步用 时钟相关

    while (len > 0) {
        if (is->audio_buf_index >= is->audio_buf_size) { // 这个也是类似环形队列的
           audio_size = audio_decode_frame(is); // 这个里面有进行resample， resample为支持格式。否则可能有杂音，以及有时钟同步synchronize_audio，主要是从音频环形队列sampq里面获取Frame
           if (audio_size < 0) {
                /* if error, just output silence */
               is->audio_buf = NULL;
               is->audio_buf_size = SDL_AUDIO_MIN_BUFFER_SIZE / is->audio_tgt.frame_size * is->audio_tgt.frame_size;
           } else {
               if (is->show_mode != SHOW_MODE_VIDEO)
                   update_sample_display(is, (int16_t *)is->audio_buf, audio_size);
               is->audio_buf_size = audio_size;
           }
           is->audio_buf_index = 0;
        }
        len1 = is->audio_buf_size - is->audio_buf_index;
        if (len1 > len)
            len1 = len;
        if (!is->muted && is->audio_buf && is->audio_volume == SDL_MIX_MAXVOLUME)
            memcpy(stream, (uint8_t *)is->audio_buf + is->audio_buf_index, len1);
        else {
            memset(stream, 0, len1);
            if (!is->muted && is->audio_buf)
                SDL_MixAudioFormat(stream, (uint8_t *)is->audio_buf + is->audio_buf_index, AUDIO_S16SYS, len1, is->audio_volume); // 拷贝进sdl里面
        }
        len -= len1;
        stream += len1;
        is->audio_buf_index += len1;
    }
    is->audio_write_buf_size = is->audio_buf_size - is->audio_buf_index;
    /* Let's assume the audio driver that is used by SDL has two periods. */
    
    // 时钟同步
    if (!isnan(is->audio_clock)) {
        set_clock_at(&is->audclk, is->audio_clock - (double)(2 * is->audio_hw_buf_size + is->audio_write_buf_size) / is->audio_tgt.bytes_per_sec, is->audio_clock_serial, audio_callback_time / 1000000.0);
        sync_clock_to_slave(&is->extclk, &is->audclk);
    }
}
```

对于audio_open是设置一些参数，比如支持一些什么格式的音视频，以及设置回调函数sdl_audio_callback。供sdl进行回调。sdl_audio_callback会记录回调时间，进行音视频时钟的同步，主要是从环形队列sampq里面获取AVFrame，然后转成支持的格式Resample，以及音频的时钟同步丢帧之类的。然后拷贝进sdl_audio_callback的stream参数里面，供设备播放。

## video_refresh

```
static void event_loop(VideoState *cur_stream)
{
    SDL_Event event;
    double incr, pos, frac;

    for (;;) {
        double x;
        refresh_loop_wait_event(cur_stream, &event);
        ... // 按键处理
    }
}
```

