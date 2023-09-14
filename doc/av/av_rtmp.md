# 流媒体协议之RTMP

## RTMP协议

RTMP（Real Time Messaging Protocol） 是由 Adobe 公司基于 Flash Player 播放器对应的音视频 flv 封装格式提出的一种，基于`TCP `的数据传输协议。本身具有稳定、兼容性强、高穿透的特点。常被应用于流媒体直播、点播等场景。常用于推推流方（主播）的稳定传输需求。

需知：RTMP基于TCP，因此建立在TCP的connect之后。

## RTMP推流

![RTMP推流流程](./res/RTMP推流流程.png)

* 首先第一步是TCP的connect连接完毕
* 客户端发起握手协议，handshake。
* 服务端接收到连接命令后，发送窗口应答大小确认信息(Window Acknowledgement Size),配置对端带宽(Set Peer Bandwidth), 发送用户控制协议(Stream Begin)告知流开始信息，并发送连接接收响应信息(_result-connect response)_
* 客户端发起创建流通道（createstream）
* _服务器接收到创建流通到后，响应创建流(_result-creatStream response)
* 发布端发起发布命令消息（public）并准备开始传输元数据消息（Metadata）、音频数据(AUdio data)
* 服务端接收到发布命令后，发送响应消息
* 发送端配置chunk size、开始发送视频数据
* 服务端返回发布结果信息，开始接收音视频流

### 其中srs的推流流程如下

```
SrsRtmpConn::do_cycle() // func
rtmp->handshake()
expect_message<SrsConnectAppPacket>(&msg, &pkt) // 等待connect包
SrsRtmpConn::service_cycle() // func
rtmp->set_window_ack_size(out_ack_size)
rtmp->set_in_window_ack_size(in_ack_size)
rtmp->set_peer_bandwidth((int)(2.5 * 1000 * 1000), 2)
rtmp->set_chunk_size(chunk_size)
rtmp->response_connect_app(req, local_ip.c_str())
rtmp->on_bw_done() // onBWDone
SrsRtmpConn::stream_service_cycle() // func
SrsRtmpServer::identify_client // func 
SrsFMLEStartPacket(releaseStream)->result
rtmp->start_fmle_publish(info->res->stream_id)
expect_message<SrsFMLEStartPacket>(&msg, &pkt)
->SrsFMLEStartResPacket // reponse
expect_message<SrsCreateStreamPacket>(&msg, &pkt)
->SrsCreateStreamResPacket // reponse
expect_message<SrsPublishPacket>(&msg, &pkt)
SrsOnStatusCallPacket* pkt = new SrsOnStatusCallPacket(); // onFCPublish
SrsOnStatusCallPacket* pkt = new SrsOnStatusCallPacket(); // onStatus
然后第一个数据是meta数据
```

## RTMP拉流

![RTMP拉流流程](./res/RTMP拉流流程.png)

* 首先第一步是TCP的connect连接完毕
* 客户端发起握手协议，handshake。
* 客户端发送createStream命令
* 服务端接收到命令，发送响应信号
* 客户端发送命令消息（play）
* 服务器端接收到播放命令play后，配置chunk大小，发送用户控制协议（StreamIsRecorded、StreamBegin）通知是否录制流，流已开启标志，之后发送播放命令响应消息（刷新当前状态、通知播放开始），这里如果play命令成功，服务端回复onStatus 命令消息 NetStream.Play.Start和NetStream.Play.Reset，其中NetStream.Play.Reset只有当客户端发送的play命令里设置了reset时才会发送，如果要播放的流没有找到，服务端会发送onStatus消息NetStream.Play.StreamNotFound。
  服务器端发送音视频消息到客户端，客户端开始播放

### srs源码拉流流程如下：

```
SrsRtmpConn::do_cycle() // func
rtmp->handshake()
expect_message<SrsConnectAppPacket>(&msg, &pkt) // 等待connect包
SrsRtmpConn::service_cycle() // func
rtmp->set_window_ack_size(out_ack_size)
rtmp->set_in_window_ack_size(in_ack_size)
rtmp->set_peer_bandwidth((int)(2.5 * 1000 * 1000), 2)
rtmp->set_chunk_size(chunk_size)
rtmp->response_connect_app(req, local_ip.c_str())
rtmp->on_bw_done() // onBWDone
SrsRtmpConn::stream_service_cycle() // func
SrsRtmpServer::identify_client // func 
expect_message<SrsCreateStreamPacket>(&msg, &pkt)
->SrsCreateStreamResPacket // reponse
SrsPlayPacket(); // play
SrsUserControlPacket(); // StreamBegin
SrsOnStatusCallPacket(); // onStatus(NetStream.Play.Reset)
SrsOnStatusCallPacket(); // onStatus(NetStream.Play.Start)
SrsSampleAccessPacket(); // |RtmpSampleAccess(false, false)
SrsOnStatusDataPacket(); // onStatus(NetStream.Data.Start)
```

类似流程如上，具体需要进行抓包，以及具体分析。

## RTMP握手

![rtmp握手](./res/RTMP握手.png)

* 握手分为简单握手，和复杂握手，流程一样，数据有区别。

- 客户端要向服务器发送**C0、C1、C2**(按序)三个 chunk,服务器向客户端发送**S0、S1、S2**(按序)三个 chunk,然后才能进行有效的信息传输。RTMP 协议本身并没有规定这 6 个**Message**的具体传输顺序,但 RTMP 协议的实现者需要保证这⼏点:
  - 客户端要等收到 S1 之后才能发送 C2
  - 客户端要等收到 S2 之后才能发送其他信息(控制信息和真实音视频等数据)
  - 服务端要等到收到 C0 之后发送 S1
  - 服务端必须等到收到 C1 之后才能发送 S2
  - 服务端必须等到收到 C2 之后才能发送其他信息(控制信息和真实音视频等数据)

但一般情况是

|      发送端      | 数据流向 | 接收端 |
| :--------------: | :------: | :----: |
| client（c0+c1）  |    ->    | server |
| server(s0+s1+s2) |    ->    | client |
|    client(c2)    |    ->    | server |

其中srs服务器对c2的校验是没有进行的。

### 简单握手

* c0和s0

  ![RTMP的C0和S0消息](./res/RTMP的C0和S0消息.png)

  c0和s0都是一个字节，值0-2位早期产品，值3为规范版本，值4-31供未来版本使用，32-255不允许使用

* c1和s1

  ![](./res/RTMP的C1和S1消息.png)

  **C1**和**S1**数据包的长度都是**1536**字节。**time**(4 个字节),这个字段包含一个**timestamp**,用于本终端发送的所有后续块的时间起点。这个值可以是 0,或者一些任意值。要同步多个块流,终端可以发送其他块流当前的**timestamp**的值。**zero**(4 个字节),这个字段必需都是 0, 不为0则为复杂协议。**random bytes**(1528 个字节),这个字段可以包含任意值。终端需要区分出响应来自它发起的握手还是对端发起的握手,这个数据应该发送一些足够随机的数。这个不需要对随机数进行加密保护,也不需要动态值。

* c2和s2

  ![RTMP的C2和S2消息](./res/RTMP的C2和S2消息.png)

  C2 和 S2 数据包长度都是 1536 个节，基本就是 S1 和 C1 的副本。S2是C1的复制。 C2是S1的复制。**time** (4字节)这个字段必须包含终端在 S1 (给 C2) 或者 C1 (给 S2) 发的timestamp。**time2** (4字节)这个字段必须包含终端先前发出数据包 (s1 或者 c1) timestamp。**random**(1528字节) 这个字段必须包含终端发的 S1 (给 C2) 或者 S2 (给 C1)的随机数。

### 复杂握手

* c0和s0

  | Field   | Type    | Comment                                                      |
  | ------- | ------- | ------------------------------------------------------------ |
  | version | 8 bytes | 说明是明文还是密文。如果使用的是明文（0X03），同时代表当前使用的rtmp协议的版本号。如果是密文，该位为0x06 |

* c1和s1

| Field   | Type      | Comment                                                      |
| ------- | --------- | ------------------------------------------------------------ |
| time    | 4 bytes   | 说明是明文还是密文。如果使用的是明文（0X03），同时代表当前使用的rtmp协议的版本号。如果是密文，该位为0x06 |
| version | 4 bytes   | 非0值，如果是0则表示简单握手。                               |
| key     | 764 bytes | random-data：长度由这个字段的最后4个byte决定，即761 - 764        key-data：128个字节。Key字段对应C1和S1有不同的算法。发送端（C1）中的Key应该是随机的，接收端（S1）的key需要按照发送端的key去计算然后返回给发送端。        random-data：（764 - offset - 128 - 4）个字节        key_offset：4字节, 最后4字节定义了key的offset（相对于KeyBlock开头而言，相当于第一个random_data的长度） |
| digest  | 764 bytes | offset：4字节, 开头4字节定义了digest的offset        random-data：长度由这个字段起始的4个byte决定        digest-data：32个字节        random-data：（764 - 4 - offset - 32）个字节 |

其中key和digest有先后顺序，分别为`schema0`和`schema1`。

**C1的key为128bytes随机数**。C1_32bytes_digest= HMACsha256(P1+P2, 1504, FPKey, 30) ，其中P1为digest之前的部分，P2为digest之后的部分，P1+P2是将这两部分拷贝到新的数组，共1536-32长度。**S1的key根据 C1的key算出来。**

* c2和s2

| Field  | Type       | Comment                                                      |
| ------ | ---------- | ------------------------------------------------------------ |
| time   | 4 bytes    | 这个字段必须包含终端在 S1 (给 C2) 或者 C1 (给 S2) 发的timestamp。 |
| time2  | 4 bytes    | 这个字段必须包含终端先前发出数据包 (s1 或者 c1) timestamp。  |
| random | 1504 bytes | random-data：1504字节                                        |
| digest | 32 bytes   | digest-data：32个字节                                        |

// client generate C2, or server valid C2
temp-key = HMACsha256(FPKey, 62, s1-digest)//是s1-digest，不是s1-digest-data
c2-digest-data = HMACsha256(c2-random-data, temp-key, 32)

// server generate S2, or client valid S2
temp-key = HMACsha256(FMSKey, 68, c1-digest)//是c1-digest，不是c1-digest-data
s2-digest-data = HMACsha256(s2-random-data, temp-key, 32)

### RTMP代理

关于RTMP代理的协议规范。RTMP是字节协议，第一个包是c0，1个字节，一般是03表示是明文的RTMP。所以如果需要做RTMP代理，如果直接转发RTMP客户端的消息，是没法传递额外的信息的，譬如HTTP代理在Header中传递的`X-Real-IP`，即客户端的IP，就没法给RTMP的后端了。

因此，RTMP的Proxy协议必须使用私有协议，c0的意义必须改写了，譬如另外一个值表示是代理，后面跟随了一些协议信息，这个协议就是RTMP Proxy协议。

| Field     | Type | Comment                                        |
| --------- | ---- | ---------------------------------------------- |
| F3        | 1B   | 常量0xF3，表示RTMP代理协议。                   |
| Size      | 2B   | 表示代理数据的长度，即Size和C0之间的数据。     |
| X-Real-IP | 4B   | 表示客户端的真实IP。                           |
| C0        | 1B   | 原始客户端的C0，方便代理直接转发客户端的数据。 |

如

```
F3            // 表示是RTMP代理
00 04         // 表示Extra有4字节
C0 A8 01 67   // 表示客户端IP，C0.A8.01.67，即192.168.1.103
03            // 客户端原始的C0数据。从这个数据（包括它本身）开始，就是客户端发送的消息了，譬如C1C2。
```

### SRS的Handshake源码如下：

```
srs_error_t SrsRtmpServer::handshake()
{
	...
    SrsComplexHandshake complex_hs;
    if ((err = complex_hs.handshake_with_client(hs_bytes, io)) != srs_success) {
            if ((err = simple_hs.handshake_with_client(hs_bytes, io)) != srs_success) {
                return srs_error_wrap(err, "simple handshake");
            }
    }
	...
}

srs_error_t SrsComplexHandshake::handshake_with_client(SrsHandshakeBytes* hs_bytes, ISrsProtocolReadWriter* io)
{
	hs_bytes->read_c0c1(io)
    c1.parse(hs_bytes->c0c1 + 1, 1536, srs_schema0) // 代理协议
    c1.c1_validate_digest // 验证c1 schema0 和schema1
    s1.s1_create(&c1) // s1生成，用了s1生成算法
    s1.s1_validate_digest(is_valid) // 进行校验
    s2.s2_create(&c1) // 生成s2
    s2.s2_validate(&c1, is_valid)
    // 数据拷贝 发送
    hs_bytes->read_c2(io) // 读取c2，但没校验
}
```

## RTMP消息格式

### Chunk

RTMP 传输的数据称为Message，Message包含音视频数据和信令，传输时不是以Message为单位的，而是把Message拆分成Chunk发送，而且必须在一个Chunk发送完成之后才能开始发送下一个Chunk，每个Chunk中带有msg stream id代表属于哪个Message，接受端也会按照这个id来将chunk组装成Message。
![](.\res\RTMP块格式.png)

#### Basic Header

* chunk type（2位）+chunk stream id（6位，14位，22位），basic header有三种类型，分别为1字节，2字节，3字节。其中最高二位`chunk type`的值代表整个chunk的类型，后续补充，**chunk stream id**一般被简写为**CSID**,用来唯一标识一个特定的流通道
* **RTMP**协议最多支持**65597**个用户自定义**chunk stream ID**,范围为 **[3,65599]**,**ID 0、1、2**被协议规范直接使用,其中**ID 值为 0、1 分表表示了 Basic Header 占用 2 个字节和 3 个字节**。
* **CSID 值 0**:代表**Basic Header**占用**2**个字节,**CSID**在 **[64,319]** 之间。**CSID 值 1**:代表**Basic Header**占用**3**个字节,**CSID**在 **[64,65599]** 之间。**CSID 值 2**:代表该**chunk**是**控制信息和一些命令信息**。

以下见basic header的类型。

![](C:\workspace\hybird\doc\av\res\RTMP块基础头1.png)

如上，该类型为1字节类型，高二位为chunk type，低六位为chunk stream id。2的六次方为64，其中**CSID 值 0**:代表**Basic Header**占用**2**个字节,。**CSID 值 1**:代表**Basic Header**占用**3**个字节,。**CSID 值 2**:代表该**chunk**是**控制信息和一些命令信息**。因此可自定义的在[3, 63]闭区间内。

![](C:\workspace\hybird\doc\av\res\RTMP块基础头2.png)

如上，该类型为2字节类型，**CSID**只占**8**位，第一个字节除**chunk type**占用的 bit 都置为**0**,第二个字节用来表示**CSID - 64**,8 位可以表示 **[0, 255]** 共 256 个数,ID 的计算方法为(第二个字节+64),范围为 **[64,319]**。

![](C:\workspace\hybird\doc\av\res\RTMP块基础头3.png)

当**Basic Header**为**3**个字节时,**ID 的计算方法为(第三字节\*256+第二字节+64)**(Basic Header 是采用小端存储的方式),范围为 **[64,65599]**。

##### SRS的Basic Header解析

```
srs_error_t SrsProtocol::read_basic_header(char& fmt, int& cid)
{
    srs_error_t err = srs_success;
    
    if ((err = in_buffer->grow(skt, 1)) != srs_success) {
        return srs_error_wrap(err, "basic header requires 1 bytes");
    }
    
    fmt = in_buffer->read_1byte();
    cid = fmt & 0x3f;
    fmt = (fmt >> 6) & 0x03;
    
    // 2-63, 1B chunk header
    if (cid > 1) {
        return err;
    // 64-319, 2B chunk header
    } else if (cid == 0) {
        if ((err = in_buffer->grow(skt, 1)) != srs_success) {
            return srs_error_wrap(err, "basic header requires 2 bytes");
        }

        cid = 64;
        cid += (uint8_t)in_buffer->read_1byte();
    // 64-65599, 3B chunk header
    } else {
        srs_assert(cid == 1);

        if ((err = in_buffer->grow(skt, 2)) != srs_success) {
            return srs_error_wrap(err, "basic header requires 3 bytes");
        }
        
        cid = 64;
        cid += (uint8_t)in_buffer->read_1byte();
        cid += ((uint8_t)in_buffer->read_1byte()) * 256;
    }
    
    return err;
}
```

#### Message Header

**Message Header**的格式和长度取决于**Basic Header**的**chunk type**,共有**4**种不同的格式,由上面所提到的**Basic Header**中的 **fmt**字段控制。



### Message

