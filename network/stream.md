# stream

由于无法确认目标地址, 一般会使用 `sip` 来**建立会话**, 然后通过 `rtsp` 进行**数据通信**

## video stream 视频

.mp4 格式带有 AVC1(ACC) 和 H264两种编码 [从mp4/flv文件中解析出h264和aac](https://www.cnblogs.com/lihaiping/p/5285166.html)

- AVC1 描述:H.264 bitstream without start codes.一般通过ffmpeg转码生成的视频，是**不带起始码0×00000001**
- H264 描述:H.264 bitstream with start codes.一般对于一下HDVD等电影的压制格式，是**带有起始码0×00000001**

## audio stream 音频