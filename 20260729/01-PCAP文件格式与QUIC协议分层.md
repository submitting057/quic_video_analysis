# PCAP 文件结构、端序、时间精度与 QUIC 协议分层

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

## QUIC 在协议栈中的位置

```text
PCAP 文件层      保存捕获记录
链路层           Ethernet / Wi-Fi
网络层           IPv4 / IPv6
UDP              承载 QUIC
QUIC             可靠传输、多路复用和加密
HTTP/3           可运行在 QUIC 之上
```

一个捕获记录通常对应一个捕获到的链路层帧，其中包含一个 UDP Datagram；一个 UDP Datagram 又可能合并一个或多个 QUIC Packet。因此不能默认：

```text
一个 PCAP Record = 一个 QUIC Packet
```

QUIC 大部分内容受到加密保护。没有会话密钥时，旁路观察者主要使用时间、方向、UDP 长度、五元组路径和少量未加密头部信息进行分析，而不能直接读取 HTTP/3 URL 或媒体内容。

[返回本次讨论总览](README.md)
