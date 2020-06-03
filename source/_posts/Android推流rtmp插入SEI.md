---
title: Android推流rtmp插入SEI
date: 2020-06-03 17:38:37
tags: rtmp推流 SEI
---
## Android MediaCodec编码 推流rtmp 插入SEI

### 插入SEI基于rtmp推流开源项目[yasea](https://github.com/begeekmyfriend/yasea)实现

> 当前项封装成flv进行推流。所以要在flv中插入SEI自定义信息  
> 当前实现的是在每个关键帧中插入SEI时间戳（也可插入其他自定义信息）


#### 修改文件为[SrsFlvMuxer.java](https://github.com/begeekmyfriend/yasea/blob/master/library/src/main/java/net/ossrs/yasea/SrsFlvMuxer.java#L876)    

修改的代码段如下

```
public void writeVideoSample(final ByteBuffer bb, MediaCodec.BufferInfo bi) {
            if (bi.size < 4) return;

            long pts = (int) (bi.presentationTimeUs / 1000);
            long dts = pts;
            int type = SrsCodecVideoAVCFrame.InterFrame;
            SrsFlvFrameBytes frame = avc.demuxAnnexb(bb, bi, true);
            int nal_unit_type = frame.data.get(0) & 0x1f;
            if (nal_unit_type == SrsAvcNaluType.IDR || bi.flags == MediaCodec.BUFFER_FLAG_KEY_FRAME) {
                type = SrsCodecVideoAVCFrame.KeyFrame;

		         //增加代码如下
                ByteBuffer sei = ByteBuffer.wrap(muxSEI(System.currentTimeMillis()+""));
                SrsFlvFrameBytes frameSei = new SrsFlvFrameBytes();
                frameSei.size = sei.array().length;
                frameSei.data = sei.duplicate();

                ipbs.add(avc.muxNaluHeader(frameSei));
                ipbs.add(frameSei);
                //增加代码如上
            } else if (nal_unit_type == SrsAvcNaluType.SPS || nal_unit_type == SrsAvcNaluType.PPS) {
                SrsFlvFrameBytes frame_pps = avc.demuxAnnexb(bb, bi, false);
                frame.size = frame.size - frame_pps.size - 4;  // 4 ---> 00 00 00 01 pps
                if (!frame.data.equals(h264_sps)) {
                    byte[] sps = new byte[frame.size];
                    frame.data.get(sps);
                    isPpsSpsSend = false;
                    h264_sps = ByteBuffer.wrap(sps);
                }

                SrsFlvFrameBytes frame_sei = avc.demuxAnnexb(bb, bi, false);
                if (frame_sei.size > 0) {
                    if (SrsAvcNaluType.SEI == (frame_sei.data.get(0) & 0x1f)) {
                        frame_pps.size = frame_pps.size - frame_sei.size - 3;// 3 ---> 00 00 01 SEI
                    }
                }

                if (frame_pps.size > 0 && !frame_pps.data.equals(h264_pps)) {
                    byte[] pps = new byte[frame_pps.size];
                    frame_pps.data.get(pps);
                    isPpsSpsSend = false;
                    h264_pps = ByteBuffer.wrap(pps);
                    writeH264SpsPps(dts, pts);
                }
                return;
            } else if (nal_unit_type != SrsAvcNaluType.NonIDR) {
                return;
            }

            ipbs.add(avc.muxNaluHeader(frame));
            ipbs.add(frame);


            writeH264IpbFrame(ipbs, type, dts, pts);
            ipbs.clear();
        }
```
#### 新增的muxSEI方法

> 该方法实现的是SEI的组装

 ```
 private byte[] muxSEI(String msg) {
        byte[] seiContent = msg.getBytes();
        byte[] seiType = new byte[]{0x06, 0x05};
        byte[] seiUuid = new byte[]{
                0x01, 0x02, 0x03, 0x04, 0x01, 0x02, 0x03, 0x04,
                0x01, 0x02, 0x03, 0x04, 0x01, 0x02, 0x03, 0x04};
        byte[] seiEnd = new byte[]{(byte) 0x80};
        int contentSize = 16 + seiContent.length;

        //数据长度(数据长度减去255，有多少个就写多少个FF，剩下的不为0，再写一个字节)
        int ffCount = 0;
        while (true) {
            if (contentSize >= 255) {
                ffCount++;
            }
            if (contentSize < 255) {
                break;
            }
            contentSize -= 255;
        }

        String size = Integer.toHexString(contentSize);
        byte[] contentLastSize = Hex.decodeHex(size);
        byte[] contentFirstSize = new byte[ffCount];
        for (int i = 0; i < ffCount; i++) {
            contentFirstSize[i] = (byte) 0xff;
        }
        byte[] sei_content_size = combineArrays(contentFirstSize, contentLastSize);

        return combineArrays(seiType, sei_content_size, seiUuid, seiContent, seiEnd);
    }

    private byte[] combineArrays(byte[]... a) {
        int massLength = 0;
        for (byte[] b : a) {
            massLength += b.length;
        }
        byte[] c = new byte[massLength];
        byte[] d;
        int index = 0;
        for (byte[] anA : a) {
            d = anA;
            System.arraycopy(d, 0, c, 0 + index, d.length);
            index += d.length;
        }
        return c;
    }
 ```

#### h264 SEI原理
 
 >  大家可参考该链接https://www.jianshu.com/p/7c6861f0d7fd
 
#### flv封装原理
 
 > 可参考该链接https://blog.evanxia.com/2017/07/1378
