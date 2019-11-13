# SDP

[SDP详细介绍](https://blog.csdn.net/longlong530/article/details/9004707)

SDP 完全是一种**会话描述格式**, 它不属于传输协议, 它只使用不同的适当的传输协议，包括会话通知协议（SAP）、会话初始协议（SIP）、实时流协议（RTSP）、MIME 扩展协议的电子邮件以及超文本传输协议（HTTP）。SDP协议是也是基于文本的协议，这样就能保证协议的可扩展性比较强，这样就使其具有广泛的应用范围。SDP 不支持会话内容或媒体编码的协商，所以在流媒体中只用来描述媒体信息。媒体协商这一块要用RTSP来实现

## 语法

1. SDP中的video**必须**携带PS属性
2. SDP中**不能**携带audio
3. 必须包含subject头域

``` go
v=0 //SDP version
o=<username> <sess-id> <sess-version> <nettype> <addrtype> <unicast/multicast-address>
s=<session-name> // “Play”代表实时点播；“Playback”代表历史回放； “Download”代表文件下载
u=<IPC国标ID.>// Uri
c=IN IP4 <invitation.caller.IP> //connect 的信息，分别描述了：网络协议，地址的类型，连接地址
t=0 0 //时间信息，分别表示开始的时间和结束的时间，一般在流媒体的直播的时移中见的比较多
m=video %d TCP/RTP/AVP 96 // // TCP media name and transport address
m=video 19690 RTP/AVP 126 125 99 34 96 // 可以对接多个 payload, 与 rtpmap:对接
m=video %d RTP/AVP 96   // UDP
m=<media> <port> <proto> <fmt> ...
m=<media> <port>/<number of ports> <proto> <fmt> // <media>可以是，"audio","video", "text", "application" and "message"。<port>是媒体传送的端口号，它依赖于c=和<proto>。<proto> 可以是，udp，RTP/AVP和RTP/SAVP。
a=setup:active // 主动 client
a=setup:passive // 被动 server
a=connection:new // 新开链接
a=<key>:<val> // attributes
a=<key>
a=fmtp:99 profile-level-id=3
a=fmtp:125 profile-level-id=42e01e
a=fmtp:126 profile-level-id=42e01e
a=rtpmap:96 MPEG4-GENERIC/32000/2 //rtpmap的信息，表示音频为AAC的其sample为32000
a=rtpmap:96 PS/90000
a=rtpmap:99 MP4V-ES/90000
a=rtpmap:125 H264S/90000
a=rtpmap:126 H264/90000
a=rtpmap:<payload type> <encoding name>/<clock rate> [/<encoding   parameters>] // 利用该属性携带编码器厂商名称(如：大华或海康编码名称DAHUA或HIKVlSlON)
m=audio 0
a=rtpmap:96 MPEG4-GENERIC/32000/2 //rtpmap的信息，表示音频为AAC的其sample为32000
a=fmtp:96 profile-level-id=15;mode=AAC-hbr;sizelength=13;indexlength=3;indexdeltalength=3;config=1210 //config为AAC的详细格式信息
a=mimetype:string;"audio/MPEG4-GENERIC"
a=recvonly
a=sendrecv
a=sendonly
```

1. Session Description
2. Timing Description
3. Media Description

### h264

1. "m=" 行中的媒体名必须是 "video"
2. "a=rtpmap" 行中的编码名称必须是 "H264".
3. "a=rtpmap" 行中的时钟频率必须是 90000.
4. 其他参数都包括在 "a=fmtp" 行中.

``` text
m=video 49170 RTP/AVP 98
a=rtpmap:98 H264/90000
a=fmtp:98 profile-level-id=42A01E; packetization-mode=1; sprop-parameter-sets=Z0IACpZTBYmI,aMljiA==
```

`sprop-parameter-sets`: SPS,PPS

这个参数可以用于传输 H.264 的**序列参数集**和**图像参数** NAL 单元. 这个参数的值采用 Base64 进行编码. 不同的参数集间用","号隔开.

`profile-level-id`
这个参数用于指示 H.264 流的 profile 类型和级别. 由 Base16(十六进制) 表示的 3 个字节. 第一个字节表示 H.264 的 Profile 类型, 第三个字节表示 H.264 的 Profile 级别:

`max-mbps`
这个参数的值是一个整型, 指出了每一秒最大的宏块处理速度.
