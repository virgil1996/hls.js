# HLS 协议详解

### 工作原理

> WIKI 解释：
>
> **HTTP Live Streaming**，缩写为**HLS**，是由[苹果公司](https://zh.wikipedia.org/wiki/苹果公司)提出基于[HTTP](https://zh.wikipedia.org/wiki/HTTP)的[流媒体](https://zh.wikipedia.org/wiki/流媒体)[网络传输协议](https://zh.wikipedia.org/wiki/网络传输协议)。是苹果公司[QuickTime X](https://zh.wikipedia.org/w/index.php?title=QuickTime_X&action=edit&redlink=1)和[iPhone](https://zh.wikipedia.org/wiki/IPhone)软件系统的一部分。它的工作原理是把整个流分成一个个小的基于HTTP的文件来下载，每次只下载一些。当媒体流正在播放时，客户端可以选择从许多不同的备用源中以不同的速率下载同样的资源，允许流媒体会话适应不同的数据速率。在开始一个流媒体会话时，客户端会下载一个包含元数据的[扩展 M3U (m3u8)](https://zh.wikipedia.org/wiki/M3U) 播放列表文件，用于寻找可用的媒体流。

简单来说，就是将视频切成一个个的 ts(transport stream) 片段，由 m3u8 文件格式记录其信息。首先下载视频的 m3u8 播放列表，在根据时间下载对应的 ts 片段播放。

### 播放列表(m3u8)

纯文本格式，分别有两种类型：

##### Master Playlist

```sh
#EXTM3U
#EXT-X-STREAM-INF:BANDWIDTH=150000,RESOLUTION=416x234,CODECS="avc1.42e00a,mp4a.40.2"
http://example.com/low/index.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=240000,RESOLUTION=416x234,CODECS="avc1.42e00a,mp4a.40.2"
http://example.com/lo_mid/index.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=440000,RESOLUTION=416x234,CODECS="avc1.42e00a,mp4a.40.2"
http://example.com/hi_mid/index.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=640000,RESOLUTION=640x360,CODECS="avc1.42e00a,mp4a.40.2"
http://example.com/high/index.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=64000,CODECS="mp4a.40.5"
http://example.com/audio/index.m3u8
```

- EXTM3U：标记开头。

- EXT-X-STREAM-INF：标记每一个 URL 的属性，一般展示不同清晰度的信息。其中 BANDWIDTH 是必填的，Int 类型，表示每个媒体文件的总体比特率的上限。

##### Playlist Construction

一个**点播**形式的 m3u8 播放列表格式如下：

```sh
#EXTM3U
#EXT-X-PLAYLIST-TYPE:VOD
#EXT-X-TARGETDURATION:10
#EXT-X-VERSION:4
#EXT-X-MEDIA-SEQUENCE:0
#EXTINF:10.0,
http://example.com/movie1/fileSequenceA.ts
#EXTINF:10.0,
http://example.com/movie1/fileSequenceB.ts
#EXTINF:10.0,
http://example.com/movie1/fileSequenceC.ts
#EXTINF:9.0,
http://example.com/movie1/fileSequenceD.ts
#EXT-X-ENDLIST
```

- EXTM3U：标记开头。
- EXT-X-PLAYLIST-TYPE：VCD，表示信息流不能修改。
- EXT-X-TARGETDURATION：单个媒体文件持续的最大时间，单位秒。
- EXT-X-VERSION：协议版本。
- EXT-X-MEDIA-SEQUENCE：标记当前视频从哪个片段开始播放。
- EXTINF：记录每一段媒体文件的持续时间。
- EXT-X-ENDLIST：标记结尾。

根据参数的配置不同，又分为以下三种模式：

1. 点播： `#EXT-X-PLAYLIST-TYPE: VOD` 表示信息流不允许修改，结尾包含 `#EXT-X-ENDLIST`。

   > VOD: Video on Demand

2. 实时转播：类似于赛事直播，和普通直播的区别是可以看之前的内容， `#EXT-X-PLAYLIST-TYPE: EVENT`表示信息流不允许修改，只允许新增，结尾没有 `#EXT-X-ENDLIST` 字段，表示正在直播，结束后自动添加。

3. 直播：没有 `#EXT-X-PLAYLIST-TYPE`，每播一个文件 `#EXT-X-MEDIA-SEQUENCE` 必须加一。

### 传输流(MPEG2-TS)

> Wiki 解释：
>
> **MPEG2-TS 传输流**（MPEG-2 Transport Stream；又称MPEG-TS、MTS、TS）是一种标准数字封装格式,用来传输和存储视频、音频与频道、节目信息，应用于数字电视广播系统，如[DVB](https://zh.wikipedia.org/wiki/DVB)、[ATSC](https://zh.wikipedia.org/wiki/ATSC)、[ISDB](https://zh.wikipedia.org/wiki/ISDB)[[3\]](https://zh.wikipedia.org/wiki/MPEG2-TS#cite_note-川口-3):118、[IPTV](https://zh.wikipedia.org/wiki/IPTV)等。

TS分组（TS Packet）大小最大为 188 字节，它是多路复用的基本单位。多个不同的 ES 的内容会分别被封装到TSP 中通过同一个TS传输

> ES：(Elementary Stream) 基本码流，不分段的音频、视频或其他信息的连续码流。

##### TS 分组格式

<img src="https://tva1.sinaimg.cn/large/008eGmZEly1gnp4pj3resj30tg0jjtbu.jpg" alt="ts-format" />

- payload：具体的音视频信息。

  > TS 包中 Payload 所传输的信息包括两种类型：视频、音频的 PES 包以及辅助数据；节目专用信息 PSI。
  >
  > PES：分组后的 ES。
  >
  > PSI：(Program Specific Information) 节目专用信息，用来表示这个 TS 包包含哪些信息。

- sync byte：表示一个 TS 片段的开始，固定 0x47。

- transport error indicator：发送时（调制前）值为 0。接收方的解调器在无法成功解调 TS 分组内容时，将该位设置为 1，表示该 TS 分组损坏。

- payload unit start indicator：负载单元起始标示符，一个完整的数据包开始时标记为1, 表示携带的是PSI或PES第一个包。

- transport priority：值为1时，在相同PID的分组中具有更高的优先权。

- PID：用于识别TS分组的ID。

- transport scrambing control：值为 '00' 时表示载荷未加密。其余值由具体系统定义。

- adaptation field control：适配域标志符，第一位表示适配域是否存在，第二位表示 payload 是否存在。

- continuity counter：连续计数器，每当一个 TS 分组中包含 payload 时，该计数器加 1。

- adaptation field：为了传送打包后长度不足188B（包括包头）的不完整TS，或者为了在系统层插入节目时钟参考 PCR 字段，需要在TS 包中插入可变长度字段的调整字段。

  ![ts-adaptation-field](https://tva1.sinaimg.cn/large/008eGmZEly1gnp56nqm5nj30rv07umya.jpg)

光看以上信息还是不够清晰，举个例子，取 https://bitdash-a.akamaihd.net/content/sintel/hls/playlist.m3u8 的第一个 TS Packet：

<img src="https://tva1.sinaimg.cn/large/008eGmZEly1gnp4q84asxj30uk0u0kjm.jpg" alt="image-20210210200746001" style="zoom: 70%;" />

其中固定部分是前四个字节 0x47400011

- 0x47：表示一个 TS Packet 开始。
- 0x4000：0100 0000 0000 0000(B)
  - 0，传输错误指示位，表示正常。
  - 1，载荷单元开始指示位，表示携带的是包头。
  - 0，传输优先级。
  - 0 0000 0000 0000(B)，PID = 0x0000。
- 0x11：0001 0001(B)
  - 00，表示未加密。
  - 01，没有 adaptation field。
  - 0001，连续计数器。
- 剩余部分，payload。

刚才我们说到 payload 包括两种类型，分组后的 ES 数据（PES）或节目关联表（PSI），其中 PSI 又分为以下四种表：

- PAT：(Program Association Table) 节目关联表，列出该 TS 内所有节目，PID 固定为 0x0000。提供传输流中包含哪些节目、节目的编号以及对应节目的 PMT ID，以及 NIT 信息。
- PMT：(Program Map Table) 节目映射表，表示该节目由那些流组成，这些流的类型（音频、视频、数据），以及组成该节目的流的位置，对应 TS 包的 PID 值，每路节目的节目时钟参考（PCR）字段的位置等。
- CAT：(Conditional Access Table) 用于节目的加密与解密。PID 固定为 0x0001。
- NIT：(Network Information Table) 网络信息表，提供 TS 的相关信息，如频率、调制方式，包含多少 TS 流等。

##### PAT 分组格式

![ts-pat-format](https://tva1.sinaimg.cn/large/008eGmZEly1gnp64fylvqj30ur0jjn0c.jpg)



- table id：PID，PAT 固定为 0x00。

- section syntax indicator：固定为1。

  > WiKi: A flag that indicates if the syntax section follows the section length. The PAT, PMT, and CAT all set this to 1

- 0：固定值。

  > WiKi: The PAT, PMT, and CAT all set this to 0. Other tables set this to 1.

- reserved：保留字段，固定为 11。
- section length：12bit，前2bit是保留位，后10位表示此字段之后的整个分段的字节数，包含 CRC 32。
- Transport stream id：用户自定义，用于在一个网络中从其他的多路复用中识别此传送流。
- reserved：保留字段，固定为11。
- version number：PAT 版本号，如果 PAT 改变，+1。
- current next indicator：为 1 时，当前PAT可用。
- section number：分段号，PAT 可能分多段传输，从 0x00 开始开始计数。
- last_section_number：最后一个分段号号数。
- N loop：循环获取 PMT 信息。
  - Program number：节目号，当为 0 时，接下来的 PID 为网络 PID。
  - reserved：3位保留位，固定111。
  - PID：13位 PID信息。
- CRC 32：校验码

继续分析上面的例子：

> 00 00 B0 0D 00 01 C1 00 00 00 01 E1 00 E8 F9 5E 7D FF...

刚才在分析 TS Header 的时候，payload_unit_start_indicator = 1，表示是否是数据包的起始位置。那么 PSI payload 的第一个 bytes 为 pointer_field，表示数据从哪里开始。

- 0x00，说明数据从下一个字节开始。
- 0x00，PAT 的 PID，固定为 0x00。
- 0xB00D，1011 0000 0000 1101(B)
  - 1011(B)，固定值和保留字段
  - 0000 0000 1011(B)，section length = 13，说明后续长度为 13 bytes
- 0x0001，stream id。
- 0xC1，1100 0001(B)
  - 11，保留字段。
  - 00000，版本号。
  - 1，当前 PAT 可用。
- 0x0000，当前分段号0，最后一个分段号也为0，表示只有一个 PAT Packet。
- 0x0001E100，循环信息
  - 0x0001，program number 为 1，表示这是 PMT 信息。
  - 0xE100，1110 0001 0000 0000(B)
    - 111(B)，三位保留位，固定部分。
    - 0 0001 0000 0000(B)，PMT ID = 256。
- 0xE8F95E7D，CRC 信息

##### PMT 信息格式

![ts-pmt-format](https://tva1.sinaimg.cn/large/008eGmZEly1gnrghudgl2j30ug0p4n0y.jpg)

结合第二个 TS Packet 分析

<img src="https://tva1.sinaimg.cn/large/008eGmZEly1gnrgvthxqcj30uk0u0hdu.jpg" alt="image-20210218101441077" style="zoom:70%;" />

0x4100，其中 PID 是后 13 位 001 0000 0000(B) = 256，和 PAT 中拿到的 PMT ID 信息是一致的，说明第二个 Packet 包含了 PMT 信息。PMT 信息片段如下：

> 02 B0 12 00 01 C1 00 00 E1 02 F0 00 1B E1 02 F0 00 A1 4F AD CC

- 0x02：table id，固定值。
- 0xB012：和 PAT 一样，包含几个固定值和 section length 长度，不再赘述。
- 0x0001：program number，表示这是 PMT。
- 0xC10000：和 PAT 一样，包含 version number, section number 等信息，不再赘述。
- 0xE102：1110 0001 0000 0010(B)，前三位是固定值，后13位为 PCR PID = 258。
- 0xF000：1111 0000 0000 0000(B)，program info length = 0，说明后面没有描述信息，开始循环
- 0x1BE102F000：
  - 0x1B：stream type 是 AVC(H264)。
  - 0xE102：1110 0001 0000 0010(B)，前三位保留位，后 13 位表示该流的 ID = 258。
  - 0xF000：前四位保留位，后 12 位表示该流的描述信息占多少字段，为 0。
- 0xA14FADCC：CRC32。

##### PES 信息格式

![ts-pes-format](https://tva1.sinaimg.cn/large/008eGmZEly1gnrjbo5snwj30tu0u9ag7.jpg)

结合第三个 TS Packet 分析

<img src="https://tva1.sinaimg.cn/large/008eGmZEly1gnsm590jxgj30uk0u07wh.jpg" alt="image-20210219100211030" style="zoom:70%;" />

其中 0x4102，说明 PID 是 258，和上面分析的 PMT 的 stream ID 一致。0x30 转为二进制 0011 0000(B)，11(B) 说明即有适配域又有载荷，需要先确定适配域范围。紧跟着后 8 位是 0x07，说明适配域长度是 7 位，跳过适配域（适配域包含了PCR信息，时间是 0，说明是视频刚开始那帧）

> 00 00 01 E0 00 00 84 C0 0A 31 00 01 3A 99 11 00 ... FF ....

- 0x000001：packet start，固定是 0x000001。

- 0xE0：1110 0000(B)，stream id，说明是视频流。音频取值(0xc0-0xdf)；视频取值(0xe0-0xef)。

- 0x0000：包长度，0 表示不限长度。

- 0x84：包含数据是否加密、优先级、版权等信息。

- 0xC0：1100 0000(B)，表示有 PTS 和 DTS。

  >  pts是显示时间戳、dts是解码时间戳，视频数据两种时间戳都需要，音频数据的pts和dts相同，所以只需要pts

- 0x0A：header 剩余长度(bytes)，通常是 5 或 10，PTS 和 DTS 都有就是 10。
- 0xFF：填充字节，表示 PES header 结束
- ES 数据




### 参考

[Wiki: HTTP Live Streaming](https://zh.wikipedia.org/wiki/HTTP_Live_Streaming)

[Apple: Live Streaming](https://developer.apple.com/documentation/http_live_streaming)

[WIKI: MPEG2-TS](https://zh.wikipedia.org/wiki/MPEG2-TS)

[Wiki: PSI](https://en.wikipedia.org/wiki/Program-specific_information)

