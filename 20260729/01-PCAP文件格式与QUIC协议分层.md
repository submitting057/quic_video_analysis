# PCAP Record、UDP Datagram、QUIC Packet 与协议分层

经典 PCAP 是一种保存原始抓包记录的二进制格式。它不是解析后的 TCP、HTTP 或 QUIC 日志，而是保存“什么时候抓到一个包、抓到了多少字节、原始字节是什么”。

## 整体结构

```text
PCAP Global Header                  24 字节，只出现一次
├── Packet Record Header           16 字节
│   └── Captured Packet Data       incl_len 字节
├── Packet Record Header           16 字节
│   └── Captured Packet Data       incl_len 字节
└── ...
```

全局头字段：

| 字段 | 长度 | 含义 |
| --- | ---: | --- |
| `magic_number` | 4 | 标识字节序和时间戳单位 |
| `version_major` | 2 | 通常为 2 |
| `version_minor` | 2 | 通常为 4 |
| `thiszone` | 4 | 已废弃，通常为 0 |
| `sigfigs` | 4 | 已废弃，通常为 0 |
| `snaplen` | 4 | 每个包允许保存的最大字节数 |
| `network` | 4 | LinkType，即捕获数据的链路层封装类型 |

每个捕获记录的 16 字节头：

| 字段 | 长度 | 含义 |
| --- | ---: | --- |
| `ts_sec` | 4 | Unix 时间戳的秒部分 |
| `ts_fraction` | 4 | 微秒或纳秒部分 |
| `incl_len` | 4 | 文件中实际保存的字节数 |
| `orig_len` | 4 | 数据包在线路上的原始长度 |

如果 `incl_len < orig_len`，说明抓包被 `snaplen` 截断。五元组可能仍能解析，但深层协议字段或载荷可能不完整。

## 为什么有多个数据包头和原始内容

一次抓包期间会出现很多数据包，因此每个被捕获的数据包都需要自己的时间和长度描述：

```text
捕获记录 1：客户端发出 QUIC Initial
捕获记录 2：服务器返回 QUIC Initial
捕获记录 3：服务器发送 Handshake
捕获记录 4：客户端发送应用数据
...
```

需要区分两种“头”：

- **PCAP Record Header**：抓包工具附加的时间和长度信息。
- **协议头**：真实网络数据中的 Ethernet、IP、UDP、QUIC 等头部。

典型层次：

```text
PCAP Record Header
└── Ethernet Header
    └── IPv4 / IPv6 Header
        └── UDP Header
            └── QUIC 数据
```

## 大端和小端

端序描述多字节整数在文件中的字节排列方式。以整数 `0x12345678` 为例：

```text
大端：12 34 56 78
小端：78 56 34 12
```

PCAP 开头四个字节可用于识别端序和时间单位：

| 文件中的字节 | 端序 | 时间单位 |
| --- | --- | --- |
| `D4 C3 B2 A1` | 小端 | 微秒 |
| `A1 B2 C3 D4` | 大端 | 微秒 |
| `4D 3C B2 A1` | 小端 | 纳秒 |
| `A1 B2 3C 4D` | 大端 | 纳秒 |

PCAP 文件头采用小端，不代表包内协议也采用小端。IPv4、UDP 等协议字段仍按照各自规范，通常使用网络大端序。

## 时间精度

微秒格式：

```text
timestamp = ts_sec + ts_usec / 1,000,000
```

纳秒格式：

```text
timestamp = ts_sec + ts_nsec / 1,000,000,000
```

需要区分：

- **分辨率**：文件字段能够表达多细的时间刻度。
- **准确度**：记录时间与真实到达时间相差多少。
- **时钟同步**：多个捕获设备的时钟是否一致。

纳秒格式不保证捕获设备真的具有纳秒级准确度。

## `network` / LinkType

`network` 不是“最高层协议”，而是告诉解析器：Captured Packet Data 从哪种封装格式开始。

| LinkType | 示例结构 |
| --- | --- |
| Ethernet | Ethernet → IP → UDP → QUIC |
| Raw IP | IP → UDP → QUIC |
| Linux SLL | Linux SLL → IP → UDP → QUIC |
| Radiotap | Radiotap → 802.11 → IP → UDP → QUIC |

因此 `network` 不会被设置为 QUIC。经典 PCAP 通常只定义一个 LinkType；需要多个接口和丰富元数据时通常使用 PCAPNG。

## PCAP Record、UDP Datagram 与 QUIC Packet

这三个概念分别属于抓包文件层、UDP 传输层和 QUIC 协议层，不能互相替代。

| 概念 | 所属层级 | 定义 |
| --- | --- | --- |
| PCAP Record | 抓包文件层 | 抓包工具保存的一条捕获记录 |
| UDP Datagram | UDP 传输层 | 一个 UDP Header 加一个 UDP Payload |
| QUIC Packet | QUIC 协议层 | UDP Payload 中一个完整、可独立处理的 QUIC 协议单元 |

### PCAP Record

PCAP Record 是存储概念，不是网络协议本身的报文单位：

```text
PCAP Record
├── Record Header
│   ├── 捕获时间
│   ├── 实际保存长度 incl_len
│   └── 线上原始长度 orig_len
└── Captured Packet Data
    └── 抓包工具保存的原始字节
```

Captured Packet Data 从哪一层开始由 LinkType 决定。Ethernet 抓包通常保存一个 Ethernet Frame；Raw IP 抓包可能直接从 IP Header 开始。

### UDP Datagram

UDP Datagram 是 UDP 协议定义的数据单元：

```text
UDP Datagram
├── UDP Header：8 字节
└── UDP Payload：可变长度
```

UDP Header 包含源端口、目的端口、UDP Length 和 Checksum。普通情况下：

```text
udp_payload_length = udp.length - 8
```

在 QUIC 通信中，UDP Payload 用来承载一个或多个 QUIC Packet。

### QUIC Packet

QUIC Packet 是 QUIC 协议自己的数据单元：

```text
QUIC Packet
├── QUIC Header
└── Protected Payload
    └── 一个或多个 QUIC Frame
```

QUIC Frame 才是 ACK、STREAM、CRYPTO、PADDING 等结构化协议信息。一个 QUIC Packet 可以包含多个 QUIC Frame。

## 完整嵌套关系

```text
PCAP 文件
└── PCAP Record
    ├── PCAP Record Header
    └── Ethernet Frame
        ├── Ethernet Header
        └── IP Packet
            ├── IP Header
            └── UDP Datagram
                ├── UDP Header
                └── UDP Payload
                    ├── QUIC Packet 1
                    │   └── 一个或多个 QUIC Frame
                    └── QUIC Packet 2
                        └── 一个或多个 QUIC Frame
```

RFC 9000 明确定义，一个 UDP Datagram 可以封装一个或多个 QUIC Packet。将多个 QUIC Packet 放入同一个 UDP Datagram 称为 Packet Coalescing，常见于握手期间把 Initial 和 Handshake Packet 一起发送。[RFC 9000](https://www.rfc-editor.org/rfc/rfc9000.html)

说明性示例：

```text
PCAP Record #420
└── Ethernet Frame
    └── IPv4 Packet
        └── UDP Datagram
            ├── UDP Header：8 字节
            └── UDP Payload：1350 字节
                ├── QUIC Initial Packet：1100 字节
                └── QUIC Handshake Packet：250 字节
```

此时关系是：

```text
1 个 PCAP Record
→ 1 个 UDP Datagram
→ 2 个 QUIC Packet
```

因此不能默认：

```text
一个 PCAP Record = 一个 QUIC Packet
```

## PCAP Record 是否等于一个 UDP Datagram

普通 Ethernet 抓包且没有分片、截断和网卡卸载时，包含 UDP 的 PCAP Record 通常可以近似看作：

```text
一个 PCAP Record
→ 一个 Ethernet Frame
→ 一个 IP Packet
→ 一个完整 UDP Datagram
```

但以下情况会破坏这种近似关系。

### IP 分片

一个 UDP Datagram 可能被 IP 层拆成多个 Fragment：

```text
一个 UDP Datagram
├── IP Fragment 1 → PCAP Record 1
├── IP Fragment 2 → PCAP Record 2
└── IP Fragment 3 → PCAP Record 3
```

只有完成 IP 重组后，三个捕获记录才对应一个完整 UDP Datagram。后续 Fragment 不能被当成独立的 UDP 事件。

### 抓包截断

如果 `snaplen` 太小，PCAP Record 可能只保存 Datagram 的前一部分：

```text
incl_len < orig_len
```

Record 仍然存在，但 Captured Packet Data 不完整。

### 网卡卸载

在终端主机抓包时，GRO、GSO、USO 等卸载机制可能使抓包工具看到经过聚合或尚未分段的数据。此时 PCAP Record 中的单位可能与线上实际发送的数据单位不同。

### 非 UDP 捕获记录

一个包含 QUIC 流量的 PCAP 文件还可能同时保存 ARP、DNS、TCP、ICMP 和其他后台通信，因此并非每个 PCAP Record 都包含 UDP Datagram。

## “QUIC PCAP”的含义

“QUIC PCAP”不是协议标准定义的数据单元，通常只是指：

> 一个包含 QUIC 流量的 PCAP 或 PCAPNG 文件。

该文件仍然使用普通 PCAP/PCAPNG 格式。Wireshark 将某个捕获记录显示为 QUIC，只表示解析器认为其中的 UDP Payload 符合 QUIC 格式，并不表示文件变成了另一种“QUIC PCAP 格式”。

应当区分：

```text
QUIC PCAP   → 包含 QUIC 流量的抓包文件，非正式称呼
QUIC Packet → QUIC 协议定义的数据单元
```

## 对 packet-level 训练的影响

“Packet-level”仍然需要明确 Packet 指的是哪一层：

```text
PCAP Record-level
UDP Datagram-level
QUIC Packet-level
```

对于无密钥、被动分析的 YouTube QUIC 流量，推荐把训练事件定义为一个完整 UDP Datagram：

```text
event_i = {
    timestamp,
    direction,
    udp_payload_length,
    path_id
}
```

预处理应执行：

```text
读取 PCAP Record
→ 判断是否包含 UDP
→ 处理 IP 分片、截断和卸载异常
→ 恢复一个完整 UDP Datagram
→ 为每个有效 UDP Datagram 生成一条模型事件
```

不应直接执行：

```text
读取一个 PCAP Record
→ 无条件当成一个 QUIC Packet
```

选择 UDP Datagram 作为基础事件的原因是：

- UDP 边界和 UDP Length 在无密钥旁路场景中稳定可见。
- 一个 UDP Datagram 可能包含多个 QUIC Packet。
- QUIC Short Header 和受保护字段使逐 QUIC Packet 解析更依赖解析器和密钥条件。
- 这种定义比直接使用 PCAP Record 更容易统一处理分片、截断和捕获异常。

QUIC 大部分内容受到加密保护。没有会话密钥时，旁路观察者主要使用时间、方向、UDP 长度、五元组路径和少量未加密头部信息进行分析，而不能直接读取 HTTP/3 URL 或媒体内容。

[返回本次讨论总览](README.md)
