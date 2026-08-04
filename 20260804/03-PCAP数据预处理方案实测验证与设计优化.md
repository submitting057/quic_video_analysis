# PCAP 数据预处理方案：实测验证与设计优化

本文基于一份真实 YouTube QUIC 加密视频 PCAP (77,212 包, 335 秒) 的全流程分析，对 [20260728/04-端到端深度学习输入设计](../20260728/04-端到端深度学习输入设计.md) 和 [20260730/03-面向Packet-Level深度学习的PCAP预处理流程](../20260730/03-面向Packet-Level深度学习的PCAP预处理流程.md) 中的设计进行实测验证，提出优化建议，并给出完整的预处理方案。

## 1. 实测数据验证结果

### 1.1 文件质量检查

| 检查项 | 结果 |
| --- | --- |
| IP 分片 | 0 个 |
| 抓包截断 (incl_len < orig_len) | 0 个 |
| GRO/GSO 巨型帧 (> MTU=1514) | 0 个 |
| 非 QUIC 的 UDP:443 流量 | 0 个 |

结论：本次抓包环境下 PCAP Record ≈ UDP Datagram，无需 IP 重组或截断修复。但预处理流程仍须保留这些检查步骤，因为其他采集环境可能出现异常。

### 1.2 Packet Coalescing

tshark 的 `quic.packet_type` 字段无法直接输出逗号分隔的多类型（Wireshark 对 coalescing 的显示存在解析限制）。但在 Pkt 30 处确认存在 Handshake + 1-RTT 共存的记录。`quic.header_form` 字段可以输出 `1,0` 这样的多值来标识 coalescing。

结论：tshark 的 `quic.packet_type` 对 coalescing 检测不可靠，应改用 `quic.header_form` 字段。

### 1.3 DCID 可见性（重大发现）

| 方向 | 1-RTT 包总数 | DCID 可见数 | 可见率 |
| --- | --- | --- | --- |
| 服务器→客户端 | 73,760 | 0 | **0%** |
| 客户端→服务器 | 2,636 | 2,636 | **100%** |

QUIC 短头（1-RTT）包中，服务器→客户端方向的 DCID 完全不可见。这是因为 QUIC 短头中 DCID 长度由服务器在握手时决定，若长度为 0 则短头中不携带 DCID。Wireshark 无法仅从短头字节判断 DCID，必须依赖握手阶段建立的上下文。

**影响**：文档 [20260728/02](../20260728/02-PCAP数据预处理与会话构建.md) 中的字段提取使用了 `quic.dcid` 和 `quic.scid`，但对占流量绝大多数的服务器→客户端 1-RTT 包，这些字段为空。连接追踪必须依赖五元组，并从握手阶段建立 DCID ↔ 五元组的映射表。

### 1.4 包大小分档验证

服务器→客户端（下行）：

| 分档 | 阈值 | 数量 | 占比 |
| --- | --- | --- | --- |
| small | ≤ 160B | 404 | 0.5% |
| medium | 161-1000B | 150 | **0.2%** |
| large | > 1000B | 73,291 | 99.2% |

客户端→服务器（上行）：

| 分档 | 阈值 | 数量 | 占比 |
| --- | --- | --- | --- |
| small | ≤ 160B | 2,169 | 79.6% |
| medium | 161-1000B | 108 | 4.0% |
| large | > 1000B | 447 | 16.4% |

**问题**：文档 [20260728/04](../20260728/04-端到端深度学习输入设计.md) 方案 B 的三档分档 (small ≤ 160, medium = 161-1000, large > 1000) 中，中档包极少（下行仅 0.2%），8 维特征中 medium_count 相关维度几乎全零。

**建议**：将 medium 上界从 1000B 调整为 800B，使分档边界更贴合 QUIC 帧的实际大小分布；或对视频 CDN 连接使用二档 (control ≤ 200, data > 200)。

### 1.5 序列长度上限

文档方案 A 建议 8192 个事件上限。实测每个 30 秒窗口的 UDP Datagram 数量：

```text
Window  0 (  0- 30s):  7,716 datagrams
Window  1 ( 30- 60s):  8,021 datagrams
Window  2 ( 60- 90s): 10,066 datagrams
Window  3 ( 90-120s):  6,297 datagrams
...
```

**问题**：8192 不足以覆盖所有 30 秒窗口，Window 2 有 10,066 个 datagram 会截断约 23%。

**建议**：将上限提升至 16384，或根据训练集高分位数（如 P99）确定 L 值。

### 1.6 五元组路径数量

```text
30 条唯一五元组路径 (UDP:443)
Top 1: 209.85.229.136:443 → 192.168.251.102:43207  71,430 包 (97%)
Top 2: 74.125.171.230:443 → 192.168.251.102:41070   1,001 包
Top 3: 142.250.197.118:443 → 192.168.251.102:57783     919 包
```

文档方案 B 的 K=4 top 路径策略合理，4 条路径已覆盖 > 99.9% 流量。

### 1.7 QUIC 版本

所有连接均使用 QUIC Version 1 (`0x00000001`)，即 IETF QUIC / HTTP/3，非 gQUIC。

### 1.8 0-RTT 包

在本次抓包中未检测到 0-RTT 包。但在抓包末尾（Pkt 77193，t=334s）出现了新 QUIC 连接的 Initial 包，属于后台统计上报，不是视频数据。

## 2. 视频段自动定位（新增步骤）

文档 [20260728/04](../20260728/04-端到端深度学习输入设计.md) 和 [20260730/03](../20260730/03-面向Packet-Level深度学习的PCAP预处理流程.md) 仅提到"训练时用浏览器日志定位 t0"，但缺少从 PCAP 中自动识别视频数据起止的规则。本节补充这一关键步骤。

### 2.1 从 operate.log 获取精确时间

operate.log 记录了完整的自动化操作流程，可以精确获取视频播放起点：

```text
phone_to_server_time_offset = -786 ms

关键时间点（相对 PCAP 首包）：
  t =  0.0s  PCAP 开始
  t =  5.4s  搜索视频 (operate.log: "Start search target video")
  t = 11.1s  开始观看 (operate.log: "Start watching the video of ENt4uip1MJ8")
  t = 11.1s  视频数据开始到达
  t =326.0s  最后一个视频段
  t =335.6s  PCAP 结束
```

### 2.2 无日志时的自动检测

当无 operate.log 可用时，按以下规则自动定位：

1. 从 DNS 响应构建 `video_cdn_ips` 集合（匹配 `*.googlevideo.com`）。
2. 找到首个连续 ≥ 5 个 ≥ 1200B 的 UDP Datagram 来自 `video_cdn_ips` 的时间点，作为视频数据开始。
3. 找到 `video_cdn_ips` 最后一次发送 ≥ 1200B 包的时间点，若此后 5 秒无新大包，判定视频数据结束。

### 2.3 视频段突发分析

使用 idle_threshold = 0.5s 对主视频 CDN 的 1-RTT 大包进行突发分割，实测结果：

```text
VideoSeg#00: t=  11.06s  0.94 MB  ← 初始缓冲(码率探测)
VideoSeg#01: t=  19.20s  2.80 MB  ← 正式开始
VideoSeg#02: t=  29.51s  2.56 MB
VideoSeg#03: t=  39.87s  1.82 MB
VideoSeg#04: t=  44.36s  2.33 MB
...
VideoSeg#34: t=326.06s  2.51 MB  ← 最后一段
```

稳定态下每个突发段约 1.5-3.0 MB，传输速率约 5 MB/s，与 av01 1080p@24fps 的码率一致。

## 3. 连接追踪（修正 DCID 不可见问题）

### 3.1 问题

文档 [20260728/02](../20260728/02-PCAP数据预处理与会话构建.md) 的字段提取包含 `quic.dcid` 和 `quic.scid`，意图用 CID 追踪 QUIC 连接。但实测表明：

- 长头包（Initial/Handshake）：SCID/DCID 均可见。
- 1-RTT 短头包，客户端→服务器：DCID 可见（100%）。
- 1-RTT 短头包，服务器→客户端：DCID 不可见（0%）。

占流量 97% 的下行 1-RTT 包无法通过 CID 追踪连接。

### 3.2 修正方案

使用五元组作为主要连接追踪标识，并从握手阶段建立 DCID → 五元组的映射表以支持 CID 迁移场景：

```python
# Phase 1: 从握手包建立映射
cid_to_path = {}
for d in datagrams:
    if d.quic_header_form == 1:  # 长头 = Initial/Handshake
        if d.src_ip == client_ip and d.quic_dcid:
            path = (d.dst_ip, d.dst_port, d.src_ip, d.src_port)
            cid_to_path[d.quic_dcid] = path
        if d.src_ip != client_ip and d.quic_scid:
            path = (d.src_ip, d.src_port, d.dst_ip, d.dst_port)
            cid_to_path[d.quic_scid] = path

# Phase 2: 为每个 datagram 分配匿名 path_id
path_counter = {}
next_id = 0
for d in datagrams:
    if d.src_ip == client_ip:
        path_key = (d.dst_ip, d.dst_port, d.src_ip, d.src_port)
    else:
        path_key = (d.src_ip, d.src_port, d.dst_ip, d.dst_port)
    if path_key not in path_counter:
        path_counter[path_key] = next_id
        next_id += 1
    d.path_id = path_counter[path_key]
```

### 3.3 path_id 编号策略优化

文档方案 A 按首次出现顺序编号。实测中主路径占 97% 流量，如果主路径不是 path_id=0，则 path_id 特征的信息效率降低。

**建议**：按流量降序编号，path_id=0 为主视频路径，使模型更容易学习到主要流量模式。

## 4. 优化点汇总

| # | 文档设计 | 实测发现 | 优化建议 |
| --- | --- | --- | --- |
| 1 | 8192 事件上限 | 30s 窗口可达 10,066 个 datagram | 提升至 **16384** 或按训练集 P99 确定 |
| 2 | 包大小三档 ≤160/161-1000/>1000 | 中档包极少（下行仅 0.2%） | medium 上界调整为 **800B**；或对视频 CDN 连接使用二档 |
| 3 | quic.dcid 用于连接追踪 | 服务器→客户端 1-RTT 中 DCID **0% 可见** | **必须用五元组 + 握手阶段 DCID 映射** |
| 4 | tshark 提取 quic.packet_type | 输出为空或不可靠 | 用 **quic.header_form** 替代 (0=1-RTT, 1=长头) |
| 5 | 文档未涉及视频段自动定位 | 可从 DNS + 吞吐量可靠定位 | **新增视频段定位步骤** |
| 6 | 文档未涉及非 QUIC 流量处理 | 实测无假阳性 | 保留 `quic \|\| udp.port==443` 双条件过滤 |
| 7 | 方案 D Burst 检测的 idle_threshold | 视频段间隔 5-10 秒，段内间隔 < 0.5s | **idle_threshold = 1.0s** 可有效分割段 |
| 8 | 匿名 path_id 按首次出现编号 | 主路径占 97% 流量 | 按流量降序编号，**path_id=0 为主视频路径** |
| 9 | 方案 B 的 K=4 top 路径 | 4 条路径已覆盖 > 99.9% 流量 | K=4 合理，需验证其他网络条件 |
| 10 | 文档未提及 debug_info.txt 的 codec/resolution | debug_info 包含 av01 1920x1080 | 应将 codec/resolution 作为 **session 元数据**，用于条件分层实验 |

## 5. 完整预处理方案

### 5.1 流水线总览

```text
原始 PCAP
  → Step 1: 文件质量检查
  → Step 2: 流量过滤
  → Step 3: 字段提取（修正 DCID 问题）
  → Step 4: 视频段定位（新增）
  → Step 5: 连接追踪（修正）
  → Step 6: 事件序列生成
  → Step 7: 会话构建
  → Step 8: 数据划分
```

### 5.2 Step 1 — 文件质量检查

使用 `capinfos` 提取元数据，检查 snaplen、LinkType、是否存在截断和分片。

### 5.3 Step 2 — 流量过滤

```bash
tshark -r input.pcap -Y 'quic || udp.port == 443' -w quic_candidates.pcap
tshark -r input.pcap -Y 'dns' -w dns_records.pcap
```

不能只依靠 `udp.port == 443`：QUIC 不一定使用 443，443/UDP 也不一定都是目标视频流量。

### 5.4 Step 3 — 字段提取

```bash
tshark -r quic_candidates.pcap -T fields -E header=y -E separator=, \
  -e frame.number \
  -e frame.time_epoch \
  -e frame.len -e frame.cap_len \
  -e ip.src -e ip.dst \
  -e udp.srcport -e udp.dstport \
  -e udp.length \
  -e ip.flags.mf -e ip.frag_offset \
  -e quic.header_form \
  -e quic.dcid -e quic.scid \
  -e quic.version
```

关键修正：

- `udp_payload_length = udp.length - 8`，不能混用 `frame.len`、`ip.len` 和 `udp.length`。
- `quic.header_form` 替代 `quic.packet_type`。
- DCID 在 1-RTT 服务端→客户端包中不可见，连接追踪需用五元组。

### 5.5 Step 4 — 视频段定位

训练时优先使用 operate.log 精确定位播放起点 t0。部署时使用 DNS 域名映射 + 吞吐量突发检测自动定位。详见 [02-视频数据传输的起止判断与截取规则](02-视频数据传输的起止判断与截取规则.md)。

### 5.6 Step 5 — 连接追踪

以五元组为主要标识，从握手阶段建立 DCID ↔ 五元组映射。按流量降序分配匿名 path_id。详见第 3 节。

### 5.7 Step 6 — 事件序列生成

方案 A 事件序列（修正后）：

```text
每个事件：
[
  signed_udp_payload_length,  # direction * udp_payload_length
  log_global_iat,             # 与会话前一 datagram 的时间间隔
  log_path_iat,               # 与同路径前一 datagram 的时间间隔
  relative_time               # 相对观察起点
]
+ path_id (匿名路径编号)
+ mask (有效/填充)

归一化：
  length_feature = signed_udp_payload_length / 1500
  global_iat     = zscore_train(log1p(clip(global_iat_ms, 0, 10000)))
  path_iat       = zscore_train(log1p(clip(path_iat_ms, 0, 10000)))
  relative_time  = t_rel / observation_duration
```

方案 B 多尺度时间 Bin（修正后）：

```text
大小分档修正：small ≤ 160B, medium = 161-800B, large > 800B
三尺度：20ms / 100ms / 500ms
Top K=4 路径 + other = 5 路径
每个 Bin 8 维：[下行字节, 上行字节, 下行小/中/大包数, 上行小/中/大包数]

X20:  [B, 5, 1500, 8]
X100: [B, 5,  300, 8]  (可由 X20 确定性池化)
X500: [B, 5,   60, 8]  (可由 X20 确定性池化)
```

方案 C 双视图融合和方案 D Burst 分层输入保持文档原始设计，无需修改。

### 5.8 Step 7 — 会话构建

```text
sessions:
  session_id, video_id, device, codec, resolution,
  network, region, observation_start, observation_duration, split

datagrams:
  session_id, event_index, relative_time,
  direction, udp_payload_length, path_id

quality:
  session_id, truncated_count, malformed_count,
  duplicate_candidate_count, capture_notes
```

新增：从 debug_info.txt 提取 `codec`（如 av01）和 `resolution`（如 1920x1080），作为 session 元数据用于条件分层实验。

### 5.9 Step 8 — 数据划分

必须先按完整 `session_id` 划分训练/验证/测试集，再在集合内部生成窗口和增强。同一 session 的连接、窗口、Burst 和增强样本不能跨集合。

## 6. 推荐代码组织

```text
preprocess/
├── quality_check.py          # Step 1: capinfos 封装
├── filter.py                 # Step 2: tshark 流量过滤
├── extract.py                # Step 3: 字段提取 (修正 DCID 问题)
├── video_locator.py          # Step 4: 视频段自动定位 (新增)
├── connection_tracker.py     # Step 5: 五元组+DCID 连接追踪 (修正)
├── event_generator.py        # Step 6: 事件序列+时间 Bin 生成
├── session_builder.py        # Step 7: 会话构建
├── splitter.py               # Step 8: 数据划分
├── pipeline.py               # 串联所有步骤
configs/
└── default.yaml              # 阈值、窗口大小、分档边界等配置
tests/
└── test_pipeline.py          # 用 ENt4uip1MJ8 作为 golden test
```

## 7. 不需修改的文档设计

以下设计经实测验证是合理的，无需调整：

- 基本事件定义为 UDP Datagram（非 PCAP Record，非 QUIC Packet）。
- 方向约定（服务器→客户端为正）和有符号长度。
- 归一化策略（length/1500，zscore on log1p(IAT)，不按会话缩放包长）。
- 匿名 path_id 不输入真实五元组、IP、端口或 CID。
- 数据划分按完整 session，不按窗口随机拆分。
- 不应作为主输入的字段列表（IP、MAC、真实五元组、CID 原始字节等）。
- 推荐实现顺序（最小事件序列 → 完整事件序列 → 多尺度 Bin → 双视图融合）。
