# PCAP 预处理总体架构与数据语义

## 1. 目标

本方案把原始 PCAP 转换为可审计、可复现、可供多类深度网络重复物化的统一事件数据，同时满足以下约束：

1. 不依赖 QUIC 载荷解密或 HTTP/3 Stream 恢复。
2. 不把某个模型的张量形状、归一化或截断策略永久写入基础数据。
3. 保留足够的审计信息，使每个模型事件都能回溯到原始捕获记录和处理决定。
4. 不向模型暴露 IP、端口、CID、文件名、设备编号、绝对时间等高风险 shortcut。
5. 同时支持离线完整会话、固定时间观察窗、固定包前缀和在线早期识别。

## 2. 三层数据架构

```text
Layer 0：原始证据
  PCAP/PCAPNG、metadata、operate.log、debug_info、采集清单

Layer 1：审计层
  原始地址与长度、协议证据、质量标志、处理原因、帧级回溯信息

Layer 2：模型安全基础层
  匿名路径、方向、UDP Payload 长度、相对时间、IAT、窗口索引

Layer 3：模型适配层
  固定长度序列、时间 Bin、路径集合、Burst、字节视图、多视图组合
```

Layer 0 和 Layer 1 属于敏感数据区。Layer 2 仍需受控使用，但已移除直接网络身份。Layer 3 可以按实验重复生成，不应成为唯一的数据真相来源。

## 3. 标识与关联单位

### 3.1 Capture

`capture_id` 对应一个独立抓包文件及其配套 metadata 和日志。它必须与原始文件名解耦，且不能由 `video_id` 直接拼接生成。

### 3.2 Playback Session

`session_id` 对应一次独立播放行为。通常一个 capture 对应一个 session，但程序必须允许：

- 一个 capture 含多个明确可切分的播放阶段；
- 一个 session 因采集轮转跨多个 capture；
- capture 只有抓包而缺少日志，session 边界只能降级推断。

无法可靠切分多个目标视频的 capture 不进入主训练集，应标记为 `ambiguous_multi_playback`。

### 3.3 Path

`path` 是规范化后的双向五元组：

```text
(client_ip, client_port, server_ip, server_port, UDP)
```

正反向 Datagram 必须映射到同一 `raw_path_key`。五元组变化产生新 path；程序不能仅凭时间接近就声称它与旧 path 属于同一 QUIC 逻辑连接。

### 3.4 Window

`window_id` 唯一标识一次模型观察范围，由以下字段确定：

```text
session_id
traffic_scope
anchor_source
window_start_offset_ns
window_duration_ns 或 packet_prefix_length
augmentation_id
```

不同模型视图使用同一个 `window_id` 对齐。

## 4. 基础事件：UDP Datagram

[RFC 9000](https://www.rfc-editor.org/rfc/rfc9000.html) 允许一个 UDP Datagram coalesce 多个独立 QUIC Packet。因此：

```text
一个成功重组的 UDP Datagram = 一个基础模型事件
```

明确排除以下错误映射：

- 一个 PCAP Record 不必然等于一个可用 Datagram；记录可能截断或包含 offload 结果。
- 一个 IP Fragment 不是一个 Datagram；只有重组完成后才生成事件。
- 一个 UDP Datagram 不必然等于一个 QUIC Packet；coalescing 不拆成多个主事件。
- 一个 QUIC Packet 可能含多个 Frame；无密钥旁路不按 Frame 生成事件。

QUIC 长短头、coalescing 数量和可见 CID 可进入审计字段或消融视图，但不改变基础事件数量。

## 5. 文件索引与原始证据绑定

每个 capture 必须执行：

1. 计算 PCAP SHA-256。
2. 识别真实文件格式和压缩格式，不依赖扩展名。
3. 关联 `.metadata`、`operate.log`、`debug_info.txt` 和其他采集清单。
4. 校验 metadata 中的文件名、大小、时间和设备地址。
5. 记录 TShark、Wireshark、libpcap 或自定义解析器的版本。

同一 SHA-256 的 PCAP 只能保留一个 canonical capture，其他记录标记为物理副本。相同事件哈希但原始文件哈希不同的 capture 作为近重复候选，必须进入同一个数据划分组。

## 6. 捕获质量处理

### 6.1 必查项目

使用 `capinfos` 和逐包字段检查：

- 文件是否可完整读取；
- PCAP/PCAPNG 类型、LinkType、接口数和每接口 snaplen；
- 时间戳精度、最早/最晚时间、是否存在时间逆序；
- `captured_length < original_length` 的截断记录；
- IPv4/IPv6、VLAN、隧道和多接口分布；
- IP 分片率、重组成功率；
- checksum offload / partial checksum 迹象；
- GRO/GSO/TSO 或异常巨型 Datagram；
- 捕获程序报告的 dropped packets（如果 pcapng 或采集日志可得）。

Wireshark 官方文档说明 checksum offload 可能使本机捕获的校验和看似错误，因此校验和错误不能单独作为删除条件。应同时记录捕获位置、方向和 `udp.checksum.partial` 等证据。

### 6.2 IP 分片

处理策略：

```text
未分片 UDP
  → 直接生成 Datagram

成功重组
  → 生成一个 Datagram，并记录 fragment_count

重组失败或关键分片截断
  → 审计层保留，主模型层排除
```

禁止把后续分片当成独立 packet-level 事件。

### 6.3 GRO/GSO 巨型记录

按证据分三种策略，配置中必须显式选择：

- `quarantine`：默认策略。无法恢复真实 Datagram 边界时，将受影响 capture 或 path 移出主训练集。
- `segment_with_metadata`：只有捕获格式或辅助元数据明确给出分段大小时才允许确定性拆分。
- `retain_for_robustness`：保留巨型事件并打标，仅进入专门的捕获层级鲁棒性实验。

禁止根据 MTU 简单等分巨型记录并把结果当成真实 Datagram。

### 6.4 镜像重复

镜像重复候选可以综合：

```text
规范化路径、方向、长度、UDP Payload 哈希、极短时间差、VLAN、接口 ID
```

只有证据充分时才删除副本。QUIC 重传、连续等长视频包和 ACK 模式不是镜像重复，不能仅按方向与长度去重。

## 7. 第一遍流量提取

第一遍以 metadata 中的设备地址为锚点提取全部 UDP：

```text
udp && (ip.addr == client_ipv4 或 ipv6.addr == client_ipv6)
```

不使用以下单一硬过滤：

- `quic`
- `udp.port == 443`
- `googlevideo.com` IP
- 最大流量路径

原因包括 dissector 无法识别短头、QUIC 可运行在非 443 端口、UDP/443 可能承载其他协议、DNS 可能缓存或加密，以及同一播放涉及多条媒体和控制路径。

## 8. 长度语义

审计层保留：

```text
frame_length
captured_length
ip_total_length / ipv6_payload_length
udp_length
udp_payload_length
```

主模型默认：

```text
udp_payload_length = udp_length - 8
```

校验条件：

```text
udp_length >= 8
udp_payload_length >= 0
udp_length 与可用捕获字节一致，或有明确截断标记
```

不同长度定义只能作为显式消融适配器存在。任何模型产物必须在元数据中记录 `length_definition`，禁止将 `frame.len`、`udp.length` 和 UDP Payload 混称为 packet size。

## 9. 方向语义

基础层不使用正负包长，而分别保存：

```text
direction = c2s | s2c
udp_payload_length = 非负整数
```

方向由设备地址判断：

```text
src 属于客户端、dst 不属于客户端 → c2s
dst 属于客户端、src 不属于客户端 → s2c
其他情况 → unknown_direction
```

不得根据端口大小、包长或“流量大的一端”推断方向。模型适配器必须声明：

```text
sign_convention = c2s_positive | s2c_positive | separate_channel
```

这样可以兼容不同论文和代码约定，而不需要重建基础数据。

## 10. QUIC 候选证据

每条 path 计算可解释证据，而不是立即二元删除：

### 强证据

- 存在可解析的 QUIC Initial、Handshake、Retry 或 Version Negotiation；
- 同一双向 path 的握手上下文可使后续短头被识别为 QUIC；
- QUIC Version、长头结构和固定比特符合协议约束。

### 支持证据

- UDP/443 或采集配置声明的 QUIC 端口；
- 与已确认 QUIC path 时间连续且双向交互合理；
- DNS/SNI/采集日志指向 YouTube 或 `googlevideo.com`；
- 播放时间窗内出现符合媒体传输的下行突发。

### 反证或风险

- DNS、mDNS、SSDP、NTP 等已知非目标协议；
- 仅单向扫描或极短无响应流；
- 与目标设备无关；
- 发生在明确的播放阶段之外。

输出 `quic_candidate_score`、`selection_reasons` 和 `selection_confidence`。默认 `playback_quic_all` 视图保留所有高置信 QUIC path；阈值附近的 path 保留在审计层，并通过敏感性实验评估。

## 11. DNS 与媒体候选映射

DNS 状态表必须处理：

- 查询名规范化和大小写；
- CNAME 链；
- A 与 AAAA；
- TTL 生效区间；
- 同一 IP 同时或先后对应多个域名；
- DNS 不可见、缓存命中、DoH/DoQ 和 ECH 等降级情况。

当有效映射匹配 `*.googlevideo.com` 时，将 path 标记为 `media_candidate`，但不能标记为“纯视频流”。该域名可能同时承载视频、音频和其他媒体相关对象。

DNS 域名、服务端 IP 和地区只用于流量选择、标签验证与误差分析，不进入默认模型特征。

## 12. 路径编号和连接辅助关联

### 12.1 path_local_id

在一个 session 的基础事件流中，按 path 首次出现顺序分配：

```text
首个新 path → 0
下一个新 path → 1
...
```

编号一旦分配不因后续流量大小变化而改变。这一规则可在线执行，不读取未来数据。

### 12.2 CID 使用边界

握手长头中可见的 DCID/SCID 建立 `cid_observation` 审计表，用于：

- 解释握手上下文；
- 标记可能的 NAT rebinding 或迁移候选；
- 检查多个 CID 与 path 的关系。

服务器到客户端短头可能使用零长度或观察者未知长度的 DCID。因此 CID 不能替代五元组，也不能直接输入主模型。

### 12.3 Top-K 禁止未来泄漏

时间 Bin 或分路径模型需要 Top-K 时，只能按当前 `window_id` 范围内的字节量或事件数排名。禁止使用完整 session、窗口结束后或其他划分中的统计量决定当前窗口顺序。

## 13. 播放阶段定位

定位结果包含：

```text
playback_start_ns
playback_end_ns
anchor_source
anchor_confidence
estimated_error_ms
locator_evidence
```

优先级：

1. `operate.log` 中明确的开始观看、停止观看事件。
2. 对齐后的 `debug_info` 中目标 Video ID 和播放器状态。
3. DNS/媒体候选 path 与下行突发自动检测。
4. PCAP 开始时间加配置偏移的降级方案。

自动检测不能使用固定的单样本 `100 KB/s` 作为全局规则。阈值由有日志训练会话监督拟合；没有监督时使用相对会话基线的稳健突变检测，详见 [02-批量数据审计与参数标定](02-批量数据审计与参数标定.md)。

如果播放窗口内出现其他 Video ID、广告或目标 ID 不稳定，应分别标记：

```text
contains_foreign_video_id
contains_ad_candidate
target_id_consistency
```

主训练集默认排除明确污染会话，污染样本保留为鲁棒性测试集。

## 14. 模型安全基础事件表

`events_model_base.parquet` 一行对应一个有效 Datagram：

```text
session_id
event_index
timestamp_rel_ns
global_iat_ns
path_iat_ns
direction
udp_payload_length
path_local_id
traffic_scope_flags
quality_flags
```

其中：

```text
timestamp_rel_ns = timestamp_ns - playback_start_ns
```

如果使用降级锚点，仍按该锚点计算并通过 session 的 `anchor_source` 说明。基础表的 `global_iat_ns` 和 `path_iat_ns` 在完成重组、重复处理、`playback_quic_all` 筛选和稳定排序后计算，主要用于审计与复算检查。

模型适配器在应用 `traffic_scope`、窗口边界或事件采样后，必须重新计算该视图内部的 `view_global_iat_ns`、`view_path_iat_ns` 和 `time_in_window_ns`。不得让媒体候选视图或偏移窗口的首个事件引用窗外事件。每个视图首事件以及每条 path 的窗内首事件 IAT 设为 `null`，由适配器决定填 0、增加 first-event mask 或使用专用 token。

基础表禁止包含：

```text
video_id、IP、端口、MAC、CID、DNS 名称、文件名、绝对时间、地区、设备编号、采集顺序
```

`traffic_scope_flags` 只能用于选择 `playback_quic_all` 或 `media_candidate` 视图，默认不作为连续模型特征。

## 15. 稳定排序与时间异常

在确认多接口时钟可比较后按以下键稳定排序：

```text
timestamp_ns, interface_id, frame_number
```

如果接口时钟不可比较，应先按接口分别建表，并要求明确的时钟校准方案；不能简单合并。

出现负 IAT 时不得直接裁剪为 0。必须定位到：

- 时间戳精度不足；
- 多接口时钟偏差；
- 记录乱序；
- 解析或排序错误。

相同时间戳允许存在，通过稳定次序保留；零 IAT 是合法值。

## 16. 会话划分与泄漏边界

处理顺序固定为：

```text
原始文件与事件去重
→ 构建 session 和分组键
→ 分配 train / validation / test
→ 仅在 train 拟合参数
→ 在各集合内部生成窗口、增强、Burst 和模型视图
```

分组键至少考虑：

```text
capture/session 来源、采集批次、日期、设备、应用版本、网络、地区、重放来源
```

同一 session 的所有窗口、路径、Burst、字节视图和起点偏移增强必须在同一 split。开放集实验中，unknown 校准与 unknown 测试的原始 Video ID 必须完全不重叠。

## 17. 与 20260804 实测的关系

[20260804](../20260804/README.md) 的样本证明以下问题在真实数据中存在：

- 前若干包不代表视频内容开始；
- 服务器到客户端短头 DCID 可能不可见；
- 30 秒事件数可能超过先验 8192；
- 主媒体 path 可占绝大多数流量；
- 视频传输呈明显分段突发。

新版吸收这些观察，但不把单份样本的 16384、800B、固定 5 个大包或固定 100 KB/s 写成全局参数。所有分布参数由批量训练域拟合。

[返回 2026-08-05 文档总览](README.md)
