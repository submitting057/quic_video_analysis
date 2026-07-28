# PCAP、QUIC 与 YouTube 视频 ID 识别：会话问题总结

本文整理了围绕 PCAP 文件格式、五元组、数据预处理，以及基于加密 QUIC 流量识别 YouTube 视频 ID 的讨论。重点是建立一条可复现的数据链路：

```text
原始 PCAP
→ UDP Datagram 事件
→ 播放会话
→ 规则指纹或深度表示
→ video_id / unknown
```

> 合规说明：视频流量指纹可能暴露用户的观看兴趣和其他敏感偏好。研究和实验应仅使用自采、获得明确授权或合法公开的数据，并在公开前移除 IP、端口、账号、时间和文件路径等可关联信息。

## 1. PCAP 数据格式

经典 PCAP 是一种保存原始抓包记录的二进制格式。它不是解析后的 TCP、HTTP 或 QUIC 日志，而是保存“什么时候抓到一个包、抓到了多少字节、原始字节是什么”。

### 1.1 整体结构

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

### 1.2 为什么有多个数据包头和原始内容

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

### 1.3 大端和小端

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

### 1.4 时间精度

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

### 1.5 `network` / LinkType

`network` 不是“最高层协议”，而是告诉解析器：Captured Packet Data 从哪种封装格式开始。

| LinkType | 示例结构 |
| --- | --- |
| Ethernet | Ethernet → IP → UDP → QUIC |
| Raw IP | IP → UDP → QUIC |
| Linux SLL | Linux SLL → IP → UDP → QUIC |
| Radiotap | Radiotap → 802.11 → IP → UDP → QUIC |

因此 `network` 不会被设置为 QUIC。经典 PCAP 通常只定义一个 LinkType；需要多个接口和丰富元数据时通常使用 PCAPNG。

### 1.6 QUIC 在哪里

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

## 2. PCAP 五元组

所谓“PCAP 五元组”不是 PCAP 文件头中的固定字段，而是解析器从 IP 和 TCP/UDP 头中提取的流量标识：

```text
(源 IP, 源端口, 目的 IP, 目的端口, 传输层协议)
```

例如：

```text
(192.168.1.20, 53124, 203.0.113.10, 443, UDP)
```

来源：

| 字段 | 来源 |
| --- | --- |
| 源/目的 IP | IPv4 或 IPv6 头 |
| 源/目的端口 | TCP 或 UDP 头 |
| 传输层协议 | IPv4 Protocol 或 IPv6 Next Header |

严格按顺序时，正向和反向是两个不同五元组；进行双向会话分析时，工具通常会将其规范化为一个 Conversation。

### 2.1 与 TCP、UDP 和 QUIC 的关系

- TCP 连接通常可由五元组配合 SYN、FIN、RST、序列号和时间识别。
- UDP 没有连接建立/关闭过程，所谓 UDP Flow 通常是相同五元组加空闲超时的工程定义。
- QUIC 支持连接迁移，因此同一逻辑连接可能跨越多个五元组。
- QUIC Connection ID 也可能轮换，所以被动观察者不能假设一个 CID 永远等于一个完整连接。
- NAT、VPN、隧道和不同抓包位置会改变观察到的外层五元组。

对于 QUIC，比较稳妥的概念是：

```text
五元组       → 当前可观察网络路径
Connection ID → QUIC 端点用于路由和识别连接的标识之一
播放会话     → 研究者需要额外定义和标注的应用级样本
```

## 3. PCAP 数据预处理

预处理目标不是简单转换格式，而是建立可追溯、可复现、不会泄漏标签的数据集。

```text
原始 PCAP
→ 文件质量检查
→ 过滤候选流量
→ 协议字段解析
→ 时间、方向和长度标准化
→ 会话/路径聚合
→ 特征或序列构造
→ 数据划分、匿名化和质量报告
```

### 3.1 文件检查

检查：

- PCAP 或 PCAPNG 类型。
- LinkType、`snaplen` 和时间戳精度。
- 是否截断、损坏、重复或混合多个接口。
- 抓包点是否受 GRO/GSO 等网卡卸载影响；主机侧出现超过 MTU 的“巨型 UDP 包”可能是抓包伪影。

常用工具：

```bash
capinfos input.pcapng
```

### 3.2 过滤和字段提取

候选 QUIC 过滤不能只依靠 `udp.port == 443`：QUIC 不一定使用 443，443/UDP 也不一定都是目标视频流量。

可提取的基础字段：

```bash
tshark -r input.pcapng \
  -T fields \
  -E header=y \
  -E separator=, \
  -e frame.number \
  -e frame.time_epoch \
  -e ip.src -e ipv6.src \
  -e ip.dst -e ipv6.dst \
  -e udp.srcport -e udp.dstport \
  -e frame.len -e ip.len -e udp.length \
  -e quic.dcid -e quic.scid
```

注意：

```text
udp.length = UDP 头 8 字节 + UDP payload
udp_payload_length = udp.length - 8
```

研究中必须固定长度定义，不能混用 `frame.len`、`ip.len`、`udp.length` 和 UDP payload 长度。

### 3.3 标准化

- 保留原始时间戳，并生成相对时间和包间隔。
- 明确客户端与服务器方向。
- 统一 IPv4/IPv6 字段。
- 保留上下行小包，不要假设它们没有信息。
- 只删除明确的抓包副本，不把真实重传或重复控制行为当作重复数据。
- 为异常记录添加 `is_truncated`、`is_malformed`、`duplicate_candidate` 等质量标记。

### 3.4 样本和数据划分

建议保留三层数据：

```text
packets / datagrams  逐事件记录
paths / flows        五元组路径记录
sessions             一次独立播放实验
```

必须先按完整的 `playback_run_id` 或 `session_id` 划分训练、验证和测试，再从各集合内部生成窗口、Burst 和增强样本。同一次播放产生的不同窗口不能跨集合。

## 4. YouTube 视频 ID 识别问题

在没有会话密钥的被动抓包中，通常无法直接读取 HTTP/3 URL 和 YouTube `video_id`。任务依赖加密流量仍暴露的侧信道：

```text
方向、UDP 载荷长度、到达时间、下载突发和多尺度节奏
```

这是一种统计推断，输出应带置信度，并允许拒识为 `unknown`，不能将预测当作确定事实。

## 5. 指纹库与 Chunk

### 5.1 指纹库

指纹库是已知视频与参考流量模式的映射：

```text
video_id
→ 多次独立参考播放
→ 每次播放的包/Burst 特征序列或学习 embedding
```

同一视频需要多个模板或原型，因为其流量会受到以下因素影响：

- 清晰度和自适应码率轨迹。
- VP9、AV1、H.264 等 Codec。
- 浏览器、设备和客户端版本。
- 带宽、RTT、丢包、MTU 和拥塞。
- CDN、缓存、广告和预加载。
- YouTube 重新编码或内容更新。

### 5.2 三种 Chunk

1. **Media Segment**：应用层真正的视频/音频分段。无密钥被动 QUIC 抓包通常不能准确看到其边界。
2. **Observed Burst**：根据连续下行流量和空闲间隔推测的网络突发。一个 Burst 不一定等于一个 Media Segment。
3. **Model Window**：固定时间或固定事件数的计算窗口，没有应用层语义。

建议分别使用 `media_segment_id`、`burst_id`、`window_id`，不要混用。

## 6. 规则匹配方法

### 6.1 离线建库

1. 每个视频在不同日期、网络、清晰度、设备下重复播放并记录真实标签。
2. 从 PCAP 提取时间、方向和 UDP payload 长度。
3. 识别一次播放会话涉及的候选 QUIC 路径。
4. 根据空闲阈值划分观测 Burst。
5. 为每个 Burst 计算字节数、包数、持续时间、前置间隔和包长分布。
6. 将 Burst 序列保存为视频的多个参考模板。
7. 只使用训练集标定距离权重、匹配阈值和 unknown 拒识阈值。

一个规则指纹可以表示为：

```text
F = [
  (burst_bytes_1, gap_1, duration_1),
  (burst_bytes_2, gap_2, duration_2),
  ...
]
```

### 6.2 在线识别

```text
待识别播放流量
→ 使用相同预处理构造 Burst 序列
→ 用总字节数、前几个 Burst 等进行候选召回
→ 使用 DTW / 编辑距离 / 序列对齐精确匹配
→ 最佳得分不足或与第二名差距太小时输出 unknown
```

输出至少包含：

```text
predicted_video_id
matching_score
second_best_score
confidence
observed_duration
```

### 6.3 数据组织

建议使用以下逻辑表：

```text
sessions:
  session_id, video_id, device, browser, codec,
  network_profile, collection_date, split

datagrams:
  session_id, event_index, relative_time,
  direction, udp_payload_length, path_id

bursts:
  session_id, burst_id, start_time, duration,
  downstream_bytes, upstream_bytes, packet_count, preceding_gap

fingerprints:
  video_id, template_id, source_session_id, burst_feature_sequence
```

原始 PCAP 只读保存，结构化大表优先使用 Parquet，变长模板可以使用 Parquet List、JSON 或 NumPy 数组。

## 7. 深度学习端到端方法

这里的“端到端”建议定义为：从被动可观察的 UDP Datagram 元数据直接学习 `video_id` 或 embedding，而不是将 PCAP 文件头、IP、端口、CID 和加密 payload 原样输入网络。

### 7.1 一个训练样本是什么

```text
一个样本 = 一次独立播放会话在固定观察时间内的全部目标 Datagram
一个标签 = 该播放会话的 video_id
```

不是一个包对应一个视频标签，也不是一个五元组必然对应一个完整视频会话。

### 7.2 基础输入：事件序列

两位独立 agent 的共同判断是：严格无密钥旁路场景应以 UDP Datagram 为基本事件，而不是假设已准确分离 QUIC Packet 或 HTTP/3 Stream。

每个事件推荐输入：

```text
[
  signed_udp_payload_length,
  log_global_iat,
  log_path_iat,
  relative_time
]
```

方向约定可以任意，但必须全数据集一致。本文采用：

```text
服务器 → 客户端：正
客户端 → 服务器：负
```

归一化示例：

```text
length_feature = signed_udp_payload_length / 1500
global_iat     = zscore_train(log1p(clip(global_iat_ms, 0, 10000)))
path_iat       = zscore_train(log1p(clip(path_iat_ms, 0, 10000)))
relative_time  = t_rel / observation_duration
```

不能按每个会话单独缩放包长，因为绝对流量规模可能正是视频指纹的一部分。所有均值和标准差只能由训练集计算。

以 30 秒、最多 8192 个 Datagram 为例：

```text
X_event: [B, 8192, 4]
path_id: [B, 8192]
mask:    [B, 8192]
y:       [B]
```

`path_id` 不是原始五元组或 CID，而是会话内的匿名编号，只表达事件属于同一可观察路径。模型可使用共享路径 embedding，但不能记忆真实 IP、端口或 CID。

优点：保留信息完整，适合 TCN、局部 Transformer 或 CNN + Transformer。缺点：序列长，并且容易受到网络环境和捕获机制影响。

### 7.3 主鲁棒输入：多尺度时间 Bin

将 30 秒观察窗按 20 ms 划分。每个路径、每个 Bin 统计：

```text
[
  downstream_bytes,
  upstream_bytes,
  downstream_small_count,
  upstream_small_count,
  downstream_medium_count,
  upstream_medium_count,
  downstream_large_count,
  upstream_large_count
]
```

示例大小分组：

```text
small  ≤ 160 B
medium = 161～1000 B
large  > 1000 B
```

保留流量最大的 `K=4` 条候选路径，其余路径合并为 `other`。对于 30 秒观察窗：

```text
X20:  [B, 5, 1500, 8]
X100: [B, 5,  300, 8]
X500: [B, 5,   60, 8]
Mpath:[B, 5]
```

`X100` 和 `X500` 可以由 `X20` 确定性池化得到。不同路径应使用共享编码器，再通过 Set Attention 或无序池化融合，避免模型赋予“第一个路径”固定语义。

多尺度分别侧重：

```text
20 ms  → 局部包团和请求响应
100 ms → 下载 Burst 形状
500 ms → 较长下载节奏和 ABR 周期
```

这种输入牺牲 Bin 内精确顺序，但比固定前 N 个包更能抵抗包速率变化、轻微乱序和时间抖动。

### 7.4 双视图输入

性能上限模型可以融合：

```text
事件分支：X_event + path_id + mask
聚合分支：X20 + X100 + X500
```

```text
事件序列 → TCN / 局部 Transformer ─┐
                                    ├→ Session Embedding
多尺度 Bin → CNN / TCN ─────────────┘
```

实验顺序应先分别验证两个分支，再验证融合，避免无法判断提升来自哪种信息。

### 7.5 Burst 序列输入

可解释基线：

```text
X_burst: [B, M, 7]

每个 Burst：
[
  direction,
  log_bytes,
  log_packet_count,
  duration,
  preceding_gap,
  large_packet_ratio,
  small_packet_ratio
]
```

纯 Burst 特征依赖人工阈值，严格来说属于深度模型学习人工表示；如果仅使用间隔确定边界，而 Burst 内部仍由包级编码器学习，则可视为分层端到端。

## 8. 会话起点和多连接问题

训练时最好通过浏览器自动化或实验日志记录真实点击播放时间：

```text
t0 = 点击播放或确认媒体开始的时间
T  = 10 / 30 / 60 秒观察窗
```

不能简单使用 QUIC 连接建立时间，因为浏览器可能复用已有连接，其中还可能承载页面、音频、视频、广告和控制请求。

如果部署时拿不到准确 `t0`，应使用：

```text
较长观测区间
→ 多个重叠滑动窗口
→ 每个窗口编码为 embedding
→ Attention / Multiple Instance Learning 聚合
```

训练时也必须加入起点偏移，例如 `-5、0、+5、+10 秒`，不能训练时使用完美起点、测试时从任意位置开始。

## 9. 分类、检索与开放集

### 闭集分类

所有查询视频都属于已知集合：

```text
Session Encoder → Softmax → video_id
```

适合作为概念验证，但不符合现实中海量未知视频的情况。

### 开放集检索

更推荐：

```text
Session Encoder
→ embedding
→ 与视频 Gallery 中的多个原型比较
→ 最近邻候选
→ 拒识阈值
→ video_id 或 unknown
```

训练可使用分类损失加监督对比损失。同一视频在不同日期、设备、网络、清晰度和 Codec 下的会话构成正样本，不同视频构成负样本。

## 10. 禁止作为主输入的字段

以下字段容易形成与视频内容无关的捷径：

- IP、MAC、真实五元组和端口。
- CID 原始字节、UDP 校验和、QUIC 密文。
- TTL、IPv6 Flow Label 和其他路径/设备标识。
- 文件名、目录、绝对时间和采集顺序。
- CDN 地址或可以直接映射采集批次的字段。

QUIC 版本、长/短头和 CID 长度建议只作为消融项，因为它们可能更多反映客户端版本、采集日期或服务部署。

## 11. 推荐实验矩阵

### 输入对照

1. 极简统计：30 秒总上下行字节、包数和持续时间。
2. 包长序列：仅 `signed_udp_payload_length`。
3. 包长 + 全局 IAT。
4. 包长 + 全局/路径 IAT + 相对时间 + 匿名路径。
5. 20/100/500 ms 多尺度时间 Bin。
6. Burst 序列。
7. 事件序列与多尺度 Bin 双视图融合。

### 观察时长

```text
10 秒：早期识别能力
30 秒：推荐主实验
60 秒：信息充分时的上限
```

### 反捷径和鲁棒性实验

- 跨日期、CDN、网络、设备、浏览器和 Codec 测试。
- 同一个视频覆盖多组网络 Trace；不同视频共享相同网络 Trace。
- 包长、时间、匿名路径逐项消融。
- 打乱短窗口内包顺序，检查模型是否只使用总流量。
- 交换不同会话的包长序列和 Timing 序列。
- 模拟或真实改变带宽、RTT、丢包、重排和 MTU。
- 测试冷/热缓存、广告/无广告和不同 Seek 起点。
- 对起点施加 `±5/±10 秒` 偏移并绘制性能曲线。
- 使用线性探针从 embedding 预测设备、日期、CDN 和网络条件；预测过强说明环境信息仍然泄漏。

### 数据划分

- 同一次播放的所有连接、窗口、Burst 和增强样本必须位于同一 split。
- Gallery 只能由训练数据生成，测试会话只能作为 Query。
- 闭集测试包含相同 video ID 的独立播放会话。
- 开集测试包含从未进入训练和 Gallery 的完整视频 ID。
- 不能先切成大量窗口后随机拆分，否则会出现严重的数据泄漏。

## 12. 最终建议

推荐按以下顺序实现和验证：

```text
第一阶段：事件序列基线
X_event = [signed UDP payload, global IAT, path IAT, relative time]

第二阶段：多尺度时间 Bin 主模型
X20 + X100 + X500，多路径共享编码和无序融合

第三阶段：开放集 embedding Gallery
分类损失 + 监督对比损失 + unknown 阈值

第四阶段：双视图融合与跨环境验证
事件细节 + 多尺度下载节奏
```

研究成功不能只由同分布准确率证明。核心问题是：模型能否在跨网络、跨日期、跨设备、跨清晰度和跨 Codec 条件下，仍然利用视频特有的多尺度下载结构，而不是记住 CDN、网络环境或采集批次。

## 13. 参考资料

- [RFC 9000 — QUIC: A UDP-Based Multiplexed and Secure Transport](https://www.rfc-editor.org/rfc/rfc9000.html)
- [RFC 9001 — Using TLS to Secure QUIC](https://www.rfc-editor.org/rfc/rfc9001.html)
- [RFC 9114 — HTTP/3](https://www.rfc-editor.org/rfc/rfc9114.html)
- [RFC 9308 — Applicability of the QUIC Transport Protocol](https://www.rfc-editor.org/rfc/rfc9308.html)
- [RFC 9312 — Manageability of the QUIC Transport Protocol](https://www.rfc-editor.org/rfc/rfc9312.html)
- [I Know What You Saw Last Minute — Encrypted HTTP Adaptive Video Streaming Title Classification](https://arxiv.org/abs/1602.00490)
- [Inferring Streaming Video Quality from Encrypted Traffic](https://arxiv.org/abs/1901.05800)
- [Wireshark User’s Guide](https://www.wireshark.org/docs/wsug_html_chunked/)
