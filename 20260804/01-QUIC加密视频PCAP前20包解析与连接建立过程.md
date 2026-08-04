# QUIC 加密视频 PCAP 前 20 包解析与连接建立过程

本文以一份真实的 YouTube QUIC 加密视频 PCAP 为例，逐包解析前 20 个数据包，还原从 DNS 查询到 QUIC 握手建立的全过程。

## 1. 采集背景

| 属性 | 值 |
| --- | --- |
| 文件 | `YouTube-EncryptedVideo-Android-3a9d9d25...-Wifi-Open-20260613_234015-yb01-ENt4uip1MJ8-auto-YouTube-HongKong.pcap` |
| 采集时间 | 2026-06-13 23:40:41 (中国标准时间) |
| 客户端 IP | 192.168.251.102 (Xiaomi 手机) |
| DNS 服务器 | 8.8.8.8 (Google Public DNS) |
| 目标服务 | YouTube |
| 视频 ID | ENt4uip1MJ8 |
| 采集地 | HongKong |
| 传输协议 | QUIC (UDP:443) + DNS (UDP:53) + 少量 SSDP/TCP |
| 总包数 | 77,212 |
| 总时长 | 335.6 秒 |

## 2. 前 20 包按功能阶段分组

### 2.1 阶段 1：DNS 域名解析 (Pkt 1-5, 7, 9)

DNS 相当于"电话簿"——客户端输入域名，DNS 返回对应 IP 地址。

| 包号 | 方向 | 内容 |
| --- | --- | --- |
| 1 | 手机 → 8.8.8.8 | 查询 `youtubei.googleapis.com` 的 A 记录 |
| 2 | 8.8.8.8 → 手机 | 回复 4 个 IP：216.239.38.223、216.239.36.223、216.239.32.223、216.239.34.223 |
| 3 | 手机 → 8.8.8.8 | 再次查询 `youtubei.googleapis.com`（不同线程重复触发） |
| 4 | 手机 → 8.8.8.8 | 查询 `i.ytimg.com` 的 A 记录（YouTube 缩略图服务器） |
| 5 | 8.8.8.8 → 手机 | 回复 youtubei 的 4 个 IP |
| 7 | 手机 → 8.8.8.8 | 第三次查询 youtubei |
| 9 | 8.8.8.8 → 手机 | 回复 `i.ytimg.com` 的 13 个 IP（Google CDN 大量边缘节点） |

Google 为每个域名返回多个 IP，这是 DNS 轮询加负载均衡策略，客户端可选择最近或最快的 IP 连接。

### 2.2 阶段 2：QUIC 连接 1 — youtubei.googleapis.com (Pkt 6, 8, 16-20)

QUIC 是 Google 设计的下一代传输协议，运行在 UDP 之上，将 TCP 的可靠传输、TLS 的加密和 HTTP/2 的多路复用融为一体，握手只需 1-RTT（甚至 0-RTT），比 TCP+TLS 快很多。

| 包号 | 方向 | 内容 |
| --- | --- | --- |
| 6 | 手机 → 216.239.38.223 | QUIC Initial，DCID=`e7b6584f7221e28d`，PKN=2，CRYPTO 帧（TLS ClientHello）+ PADDING + PING |
| 8 | 手机 → 216.239.38.223 | QUIC Initial，同 DCID，PKN=1（QUIC 允许乱序发送） |
| 16 | 服务器 → 手机 | QUIC Initial，SCID=`e7b6584f7221e28d`，PKN=1，ACK |
| 17 | 服务器 → 手机 | QUIC Initial，PKN=2，ACK + PADDING |
| 18-20 | 服务器 → 手机 | QUIC Initial，PKN=3-5，CRYPTO（TLS ServerHello + 证书）+ PADDING |

关键概念：

- **DCID/SCID**：Destination/Source Connection ID，QUIC 连接标识符。手机生成 DCID 告诉服务器"这是我们的连接 ID"，服务器回复时用 SCID 原样返回。
- **Initial 包**：QUIC 握手的第一个阶段，里面装着 TLS 的 CRYPTO 帧（ClientHello/ServerHello）。
- **PADDING 帧**：填充数据，让包达到一定大小以防止探测和小包攻击。
- **PKN**：Packet Number，QUIC 包号，严格递增（不同于 TCP 的序列号代表字节位置）。

握手流程示意：

```text
手机                           服务器 (216.239.38.223)
  |--- QUIC Initial (CRYPTO: ClientHello) -->|
  |<-- QUIC Initial (ACK) -------------------|
  |<-- QUIC Initial (ACK + PADDING) ---------|
  |<-- QUIC Initial (CRYPTO: ServerHello) ---|
  |<-- QUIC Initial (CRYPTO: Certificate) ---|
```

这就是 QUIC 的 1-RTT 握手：客户端发 1 个包，服务器回几个包，握手基本完成。

### 2.3 阶段 3：SSDP 设备发现 (Pkt 10-12)

| 包号 | 方向 | 内容 |
| --- | --- | --- |
| 10-12 | 手机 → 239.255.255.250 | SSDP M-SEARCH 组播 |

SSDP（Simple Service Discovery Protocol）用于局域网设备互相发现（如投屏设备、打印机）。`239.255.255.250` 是组播地址，端口 1900。这是手机在扫描"局域网里有没有可投屏的设备"。

### 2.4 阶段 4：QUIC 连接 2 — i.ytimg.com (Pkt 13-15)

| 包号 | 方向 | 内容 |
| --- | --- | --- |
| 13 | 手机 → 142.250.197.118 | TCP SYN → 443 端口（传统 HTTPS 尝试） |
| 14-15 | 手机 → 142.250.197.118 | QUIC Initial，DCID=`2c7a5de12c843586`，CRYPTO + PING + PADDING |

Pkt 13 发了一个 TCP SYN（传统 HTTPS 尝试），但紧接着 Pkt 14/15 就发了 QUIC Initial。这说明 Android YouTube 客户端同时尝试 TCP 和 QUIC 连接（竞速策略 / Racing），QUIC 一旦成功，TCP 连接就会被放弃。Pkt 13 的 TCP SYN 没有后续——QUIC 赢了。

## 3. 完整时间线

```text
时间轴 (0~0.17 秒)
────────────────────────────────────────────────────────
0.000s  DNS 查询 youtubei.googleapis.com           ┐
0.047s  DNS 回复 (4 个 IP)                          │ DNS 阶段
0.091s  DNS 查询 youtubei (重复)                    │
0.107s  DNS 查询 i.ytimg.com                        ┘
0.124s  DNS 回复 youtubei                            ┐
0.135s  QUIC Initial → 216.239.38.223 (连接 1)       │ QUIC 握手 1 开始
0.135s  DNS 查询 youtubei (第三次)                   │
0.135s  QUIC Initial → 216.239.38.223 (PKN=1)       ┘
0.139s  DNS 回复 i.ytimg.com (13 个 IP)
0.143s  SSDP 组播 x3 (设备发现)
0.150s  TCP SYN → 142.250.197.118 (竞速)            ┐ QUIC 握手 2 开始
0.150s  QUIC Initial → 142.250.197.118 (连接 2)      │
0.150s  QUIC Initial → 142.250.197.118 (PKN=2)      ┘
0.169s  ← QUIC ACK from 216.239.38.223             ┐ 服务器响应连接 1
0.172s  ← QUIC ACK + CRYPTO x4                     ┘ (ServerHello 等)
────────────────────────────────────────────────────────
```

整个 0.17 秒内：手机打开 YouTube → DNS 解析两个域名 → 同时建立 2 条 QUIC 连接（视频 API + 缩略图 CDN）→ 服务器开始回复握手 → 同时扫描局域网投屏设备。

## 4. 前 20 包中是否有视频数据？

**没有。** 前 20 个包只完成了 DNS 解析和 QUIC Initial 握手。视频数据的传输在约 5.5 秒后才开始（参见 [02-视频数据传输的起止判断与截取规则](02-视频数据传输的起止判断与截取规则.md)）。
