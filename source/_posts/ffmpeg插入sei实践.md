---
title: ffmpeg插入sei实践
date: 2020-11-09 14:00:18
tags:
---


本文章主要讲述如何修改ffmpeg源码，实现在每个关键帧中插入sei。

## 原始Bsf

BitStream Filter（码流过滤）的缩写为bsf，它的作用是，在不做码流解码的前提下，对已经编码后的比特流做特定的修改、调整。

bsf h264_metadata的调用

使用ffmpeg工具时，可以使用比特流过滤器。基本的filter调用格式如下：

```
ffmpeg -i INPUT -c:v copy -bsf:v filter1[=opt1=str1:opt2=str2][,filter2] OUTPUT
```

可以使用 h264_metadata比特流过滤器添加SEI。下面示例命令添加了类型为未注册的用户数据的SEI，其中uuid为"086f3693-b7b3-4f2c-9653-21492feee5b8"，payload内容为"hello"：

```
./ffmpeg  -I oceans.h264 -c:v copy -bsf:v h264_metadata=sei_user_data='086f3693-b7b3-4f2c-9653-21492feee5b8+hello' oceans.sei.h264
```

## Bsf修改
修改文件为[h264_metadata_bsf.c](https://github.com/FFmpeg/FFmpeg/blob/master/libavcodec/h264_metadata_bsf.c)

#### 查找出关键帧

from
```
    has_sps = 0;
    for (i = 0; i < au->nb_units; i++) {
        if (au->units[i].type == H264_NAL_SPS) {
            err = h264_metadata_update_sps(bsf, au->units[i].content);
            if (err < 0)
                goto fail;
            has_sps = 1;
        }
    }
```
to

```
    int has_idr = 0;
    has_sps = 0;
    for (i = 0; i < au->nb_units; i++) {
        if (au->units[i].type == H264_NAL_SPS) {
            err = h264_metadata_update_sps(bsf, au->units[i].content);
            if (err < 0)
                goto fail;
            has_sps = 1;
        }
        if (au->units[i].type == H264_NAL_IDR_SLICE) {
            has_idr = 1;
        }
    }

```
#### 判断是关键帧
from

```
    // Only insert the SEI in access units containing SPSs, and also
    // unconditionally in the first access unit we ever see.
    if (ctx->sei_user_data && (has_sps || !ctx->done_first_au)) {
```
to

```
    // Only insert the SEI in access units containing IDRs, and also
    // unconditionally in the first access unit we ever see.
    if (ctx->sei_user_data && (has_idr || !ctx->done_first_au)) {

```

#### 在关键帧后插入一帧SEI
当参数为“**-bsf:v h264_metadata=sei_user_data='086f3693-b7b3-4f2c-9653-21492feee5b8+{timestamp}'**”时插入当前时间戳。

解析出的SEI信息为"ts:timestamp"精确到毫秒

from

```
    udu->data        = udu->data_ref->data;
    udu->data_length = len + 1;
    memcpy(udu->data, ctx->sei_user_data + i + 1, len + 1);

```
to

```
    udu->data        = udu->data_ref->data;
    udu->data_length = len + 1;
    memcpy(udu->data, ctx->sei_user_data + i + 1, len + 1);
            
    //convert to Timestamp
    if(strcmp(udu->data, "{timestamp}") == 0){
        char mark[] = "ts:";
        long timestamp = av_gettime() / 1000;
        char result[20];
        sprintf(result, "%s%ld", mark, timestamp);
                
        udu->data        = result;
        udu->data_length = strlen(result) + 1;
    }

```

### 调试方法
使用xcode

- https://www.jianshu.com/p/226c19aa6e42
- https://blog.f-fox.com/2019/03/06/%E4%BD%BF%E7%94%A8Xcode%E6%96%AD%E7%82%B9%E8%B0%83%E8%AF%95ffmpeg/


### 使用方法

```
-bsf:v h264_metadata=sei_user_data='086f3693-b7b3-4f2c-9653-21492feee5b8+{timestamp}'
```
栗子：
```
-stream_loop -1 -i video.mp4  -c:a aac -c:v libx264 -bsf:v h264_metadata=sei_user_data='086f3693-b7b3-4f2c-9653-21492feee5b8+{timestamp}' -f flv rtmp://demo-push.cn/live/11
```


### 参考资料

- https://mp.weixin.qq.com/s/TePsSvMgGDlZ9RgUkqFJ3A
- http://bbs.chinaffmpeg.com/forum.php?mod=viewthread&tid=589&fromuid=3