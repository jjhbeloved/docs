# SDP

[SDP详细介绍](https://blog.csdn.net/longlong530/article/details/9004707)

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