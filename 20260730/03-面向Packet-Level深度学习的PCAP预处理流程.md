# 面向 Packet-Level 深度学习的 PCAP 原始数据预处理流程

## 1. 目标与适用条件

本文定义从 YouTube 视频采集 PCAP 到 packet-level 深度学习基础事件表的预处理流程。当前数据满足以下条件：

- PCAP 在靠近客户端的交换机侧采集。
- 一份 PCAP 只播放一个目标视频。
- 每次采集均有 `.metadata`、`operate.log`、`debug_info.txt` 和 `adb_log.txt` 等日志。
- 广告已尽量避免，采集前清除缓存，同时避免其他标签页和后台网络流量。
- 目标传输协议为 QUIC，不使用会话密钥解密应用载荷。

预处理的最终目标是：

```text
原始 PCAP 与采集日志
→ 一行对应一个 UDP Datagram 的审计事件表
→ 去除身份泄漏字段的模型基础事件表
```

当前阶段只保留原始 packet-level 数值，不进行归一化、固定窗口、截断、Padding、Mask 或模型特征变换。

## 2. 基本数据单位

### 2.1 捕获记录不等于 QUIC Packet

一次 PCAP 捕获记录通常包含一个链路层帧，其中包含一个 IP Packet 和一个 UDP Datagram。一个 UDP Datagram 可以封装一个或多个 QUIC Packet，因此不能假设：

```text
一个 PCAP Record = 一个 QUIC Packet
```

本流程采用的基本事件单位是：

```text
一个经过 IP 分片重组后的 UDP Datagram
```

### 2.2 会话与样本标识

当前一份 PCAP 对应一次视频播放，因此：

```text
capture_id = 原始采集会话标识
session_id = capture_id
```

后续生成 10、30、60 秒观察窗口时，应另外创建 `sample_id`，不要修改原始 `session_id`。

## 3. 总体处理链路

```text
关联 PCAP、metadata 和日志
→ 校验采集文件一致性
→ 构造统一时间线和播放阶段
→ 检查交换机抓包质量
→ 提取客户端全部 UDP 流量
→ 重组 IP 分片
→ 识别目标 QUIC 候选路径
→ 根据 device_ip 确定方向
→ 生成匿名 path_id
→ 检测镜像重复和异常记录
→ 检测广告或外部内容
→ 按播放阶段切分
→ 计算相对时间和 IAT
→ 输出审计表与模型基础表
→ 生成质量报告和可追溯信息
```

## 4. 关联一次采集的全部文件

通过文件名前缀和 `capture_id` 关联：

```text
PCAP
.metadata
.operate.log
debug_info.txt
adb_log.txt
```

生成 `captures.parquet`，至少包含：

```text
capture_id
session_id
pcap_path
metadata_path
operate_log_path
debug_info_path
adb_log_path
target_video_id
device_ip
device_mac
capture_time
capture_end_time
phone_to_server_time_offset
application_version
device_mode
os_version
network_information
target_protocol
target_resolution
watch_speed
drag_start
drag_to
scripts_version
pcap_sha256
```

原始路径只用于审计，不得作为模型输入。

## 5. 验证采集文件一致性

每个采集会话执行以下检查：

1. `.metadata` 的 `file_name` 与实际 PCAP 文件名一致。
2. `.metadata` 的 `file_size` 与实际文件大小一致。
3. 所有相关文件的 UUID 与 `capture_id` 一致。
4. `target_protocol` 为 QUIC。
5. `debug_info.txt` 中至少一次出现目标 `Video ID`。
6. `operate.log` 包含启动抓包、清除缓存、播放、结束播放和停止抓包等阶段。
7. PCAP 捕获时间范围完整覆盖视频播放阶段。
8. PCAP 可以被 `capinfos` 和 TShark 正常读取。
9. 计算并保存原始 PCAP 的 SHA-256。

不满足要求的会话不直接删除，而是写入：

```text
excluded_sessions.parquet
```

并记录明确的 `exclusion_reason`。

## 6. 构造统一时间线

当前日志存在两类时钟：

```text
PCAP、operate.log   → 抓包服务器时间
debug_info、adb_log → 手机设备时间
```

使用 `phone_to_server_time_offset` 对齐手机日志与服务器时间，但必须先确认偏移量的准确公式，不能只根据字段名猜测。

生成 `phases.parquet`：

```text
capture_id
capture_start_ns
search_start_ns
video_click_ns
playback_start_ns
playback_end_ns
capture_end_ns
alignment_offset_ms
alignment_formula
alignment_quality
```

一次采集划分为：

```text
initialization
app_cleanup
search
pre_play
playback
post_play
```

预处理结果保留所有阶段，并为每个 Datagram 增加 `phase`。主训练集默认只使用 `playback` 阶段，但不要在基础数据层永久删除 `pre_play`，以便后续分析预加载和起点定义。

## 7. 检查交换机侧抓包质量

交换机侧抓包通常避免客户端主机 GRO/GSO 导致的虚假超大包，但需要重点检查交换机镜像问题：

- SPAN/镜像是否同时包含 ingress 和 egress。
- 同一个包是否被多个镜像源重复复制。
- 镜像端口带宽是否不足并产生丢包。
- 是否包含 VLAN Tag。
- 是否混入同一 VLAN 的其他设备流量。
- 捕获文件是否包含多个不同时间源的接口。
- 是否存在 IP 分片、截断或异常时间戳。

生成 `capture_quality.parquet`：

```text
capture_id
packet_count
udp_datagram_count
truncated_count
fragmented_count
malformed_count
duplicate_candidate_count
negative_timestamp_count
mirror_drop_count
unexpected_large_datagram_count
unrelated_device_packet_count
quality_status
```

交换机镜像丢包会直接改变包序列和 IAT，因此必须作为正式质量指标，而不能仅作为备注。

## 8. 第一遍提取客户端全部 UDP 流量

第一遍不要只使用 `quic` Display Filter，否则可能遗漏未被当前解析器识别的后续短头 Datagram。

根据 `.metadata` 中的 `device_ip` 提取 UDP。例如：

```bash
tshark -n -2 -r input.pcap -Y "udp && ip.addr == DEVICE_IP" -T fields \
  -E header=y -E separator=/t -E quote=d -E occurrence=f \
  -e frame.number \
  -e frame.interface_id \
  -e frame.time_epoch \
  -e frame.len \
  -e frame.cap_len \
  -e eth.src \
  -e eth.dst \
  -e vlan.id \
  -e ip.version \
  -e ip.src \
  -e ip.dst \
  -e ip.len \
  -e ip.id \
  -e ip.flags.mf \
  -e ip.frag_offset \
  -e udp.srcport \
  -e udp.dstport \
  -e udp.length
```

执行时必须将 `DEVICE_IP` 替换为当前会话的真实设备 IP。PowerShell 可将反斜杠续行改为反引号，或者将命令写成一行。

如果网络启用 IPv6，还应提取：

```text
ipv6.src
ipv6.dst
ipv6.plen
ipv6.nxt
```

并建立设备 IPv6 地址与采集设备的可靠映射。

## 9. 统一长度定义

保留以下原始长度：

```text
frame_len
frame_cap_len
ip_len / ipv6_payload_length
udp_length
udp_payload_length
```

主候选长度定义为：

```text
udp_payload_length = udp.length - 8
```

基本校验：

```text
udp.length >= 8
udp_payload_length >= 0
```

当前阶段不删除其他长度字段，以便后续消融实验和异常定位，但模型基础表优先保留 `udp_payload_length`。

## 10. IP 分片重组

IP Fragment 不是独立 UDP Datagram。处理规则：

```text
未分片 UDP
→ 直接生成一个 Datagram 事件

成功重组的 IP 分片
→ 生成一个 Datagram 事件

重组失败
→ 标记并排除该 Datagram
```

后续分片不能作为单独 packet-level 事件进入模型。

记录：

```text
is_fragmented
fragment_count
reassembly_success
```

## 11. 识别目标 QUIC 候选路径

即使后台流量已经尽量避免，也不能直接假设设备的全部 UDP 流量都是目标视频。

建议按照以下证据筛选：

1. 流量属于 `.metadata.device_ip`。
2. 流量发生在采集和播放相关时间窗口内。
3. 传输层协议为 UDP。
4. 路径出现 QUIC Initial、QUIC Version，或被 dissector 识别为 QUIC。
5. 后续相同双向五元组的短头流量继续归入已确认路径。
6. 可选使用 SNI、服务端网段或 `googlevideo.com` 信息辅助验证。

不要只保留流量最大的一个路径。一次播放可能同时涉及视频、音频、控制连接、新旧 CDN 路径或连接迁移。

每条候选路径记录：

```text
raw_path_key
first_seen_ns
last_seen_ns
quic_detected
quic_version
selection_reason
selection_confidence
```

## 12. 确定上下行方向

使用 `.metadata.device_ip`，不要根据端口大小判断方向：

```text
src_ip == device_ip
→ 客户端到服务器
→ direction = -1

dst_ip == device_ip
→ 服务器到客户端
→ direction = +1
```

采用：

```text
下行 = +1
上行 = -1
```

并生成：

```text
signed_udp_payload_length = direction × udp_payload_length
```

方向未知的记录不进入正式模型表，但应保留在审计和质量报告中。

## 13. 生成匿名路径编号

审计表保留原始五元组，但模型不能看到真实 IP 和端口。

将正向和反向规范化为同一个双向路径，再按首次出现顺序编号：

```text
第一条路径 → path_id = 0
第二条路径 → path_id = 1
第三条路径 → path_id = 2
```

必须区分：

```text
path_id    → 可观察的五元组路径
QUIC CID   → QUIC 使用的连接标识之一
session_id → 一次完整播放会话
```

当前阶段不要强行将多个 `path_id` 合并成一个 QUIC 逻辑连接。CID 可用于辅助分析，但不能作为唯一关联依据。

## 14. 检测镜像重复和异常记录

交换机镜像重复候选可以综合以下字段判断：

```text
方向
规范化五元组
UDP 长度
UDP payload 哈希
极短时间差
VLAN
捕获接口
```

处理原则：

- 能证明是 SPAN 重复时删除一个副本。
- 无法证明时保留并标记 `duplicate_candidate`。
- 不按包长和方向简单去重。
- 不把连续相同长度的大包误判为镜像副本。
- 保留原始 `frame.number`，确保所有删除行为可追溯。

## 15. 检测广告和外部内容

使用 `debug_info.txt` 交叉验证：

```text
debug_info.Video ID == metadata.target_video_id
```

如果播放窗口内出现其他 Video ID：

```text
contains_foreign_video_id = true
contains_ad_candidate = true
```

首版干净训练集建议：

- 明确出现广告或其他 Video ID：从主训练集排除。
- 短暂不确定状态：保留审计表，但不进入主训练集。
- 将广告会话单独保存为后续鲁棒性测试集。

`Video format`、`Audio format`、`sCPN`、`Readahead`、`Bandwidth` 和 `Dropped frames` 用于标签验证、分层评估和异常分析，不作为视频 ID 模型输入。

## 16. 生成完整 Datagram 审计表

一行严格对应一个成功重组的 UDP Datagram：

```text
capture_id
session_id
target_video_id
frame_number
interface_id
timestamp_ns
phase
direction
src_ip
dst_ip
src_port
dst_port
ip_version
vlan_id
frame_len
frame_cap_len
udp_length
udp_payload_length
signed_udp_payload_length
path_id
is_quic_candidate
quic_version
is_fragmented
is_truncated
is_duplicate_candidate
contains_ad_candidate
quality_flags
```

该表用于审计、排错和重现实验，不直接作为模型输入。

## 17. 排序并计算 Packet-Level 时间

在确认捕获接口时间一致后，按以下字段稳定排序：

```text
timestamp_ns
frame_number
```

完成候选流量筛选和播放会话切分后，再计算：

```text
t_rel_ns = timestamp_ns - playback_start_ns

global_iat_ns[i] =
    timestamp_ns[i] - timestamp_ns[i-1]

path_iat_ns[i] =
    timestamp_ns[i] - previous_timestamp_on_same_path
```

不要直接使用 TShark 的 `frame.time_delta` 作为最终 IAT，因为它可能以非目标设备、非 UDP 或播放前后的帧为参照。

如果出现负 IAT，不应直接裁剪为 0，而应检查多接口时钟、镜像重复、时间戳异常和排序逻辑。

## 18. 生成模型基础事件表

模型安全表只保留：

```text
session_id
event_index
t_rel_ns
global_iat_ns
path_iat_ns
direction
udp_payload_length
path_id
```

标签表单独保存：

```text
session_id
capture_id
video_id
sCPN
device_mode
application_version
codec
resolution
network_profile
collection_date
quality_group
```

禁止进入模型输入：

```text
video_id
文件名
绝对时间
IP 地址
端口
MAC 地址
CID 原始值
地区
设备编号
采集顺序
```

这些字段可保留在标签和审计表中，用于跨条件划分和误差分析。

## 19. 当前阶段不执行的操作

基础事件层保留原始整数值：

```text
udp_payload_length → 字节
t_rel_ns           → 纳秒
global_iat_ns      → 纳秒
path_iat_ns        → 纳秒
direction          → -1 / +1
path_id            → 会话内整数
```

暂时不要执行：

- 包长除以 MTU 或 1500。
- 对 IAT 执行 `log1p`。
- Z-score 或 Min-Max 归一化。
- 固定前 4096/8192 个 Datagram。
- Padding 和 Mask。
- 永久删除完整会话，只保留 10/30/60 秒窗口。
- Burst 聚合。
- 删除小包或只保留下行包。

上述操作应在模型 Dataset/DataLoader 阶段完成，并且归一化统计量只能来自训练集。

## 20. 推荐输出目录

```text
processed/
├── manifests/
│   ├── captures.parquet
│   ├── sessions.parquet
│   ├── phases.parquet
│   └── labels.parquet
├── events/
│   ├── datagrams_audit.parquet
│   └── datagrams_model_base.parquet
├── quality/
│   ├── capture_quality.parquet
│   ├── excluded_sessions.parquet
│   └── exclusion_reasons.json
└── provenance/
    ├── tshark_version.txt
    ├── parser_config.json
    ├── field_schema.json
    └── preprocessing_version.json
```

结构化事件表优先使用 Parquet，以保留整数纳秒、字段类型、压缩和分区能力。

## 21. 质量验收标准

正式训练数据至少满足：

- PCAP 可完整读取，SHA-256 已记录。
- metadata、日志和 PCAP 的 `capture_id` 关联一致。
- `video_id`、客户端 IP 和播放阶段明确。
- 一行严格对应一个有效 UDP Datagram。
- IP Fragment 没有被作为独立模型事件。
- UDP 长度有效，方向明确。
- 最终会话序列不存在未解释的负 IAT。
- 截断率、镜像重复率、分片率和捕获丢包均有报告。
- 播放窗口中不存在未经处理的其他 Video ID。
- 模型表不包含 IP、端口、CID、绝对时间、文件名和设备编号。
- 同一次播放的所有事件只能进入同一个数据划分。
- 使用相同 PCAP、日志、解析版本和配置重复运行时，事件数量和内容哈希一致。

## 22. 推荐实现模块

预处理程序建议拆分为：

```text
index_sessions
  解析 metadata 和日志，关联文件并生成 captures/sessions/phases

extract_datagrams
  调用 TShark 或解析库，生成原始 UDP Datagram 表

select_quic_paths
  根据设备、时间和 QUIC 证据筛选候选路径

clean_datagrams
  处理分片、方向、重复和质量标记

build_model_base
  计算相对时间、IAT、path_id，生成模型安全表

validate_dataset
  执行一致性、泄漏、质量和可复现性检查
```

每个模块输出确定的数据文件和质量报告，避免把全部逻辑写在一个不可审计的脚本中。

## 23. 尚需确认的技术细节

在实现程序前还需锁定：

1. 交换机抓包采用 SPAN/端口镜像，还是交换机自身抓包功能；是否同时镜像 ingress 和 egress。
2. `phone_to_server_time_offset` 的准确换算公式。
3. 网络是否启用 IPv6，以及如何获得设备 IPv6 地址。
4. 广告候选会话是否按默认策略从主训练集排除并单独作为鲁棒性测试集。

## 参考资料

- [RFC 9000：QUIC Transport](https://www.rfc-editor.org/rfc/rfc9000.html)
- [TShark Manual Page](https://www.wireshark.org/docs/man-pages/tshark.html)
- [Wireshark Frame Display Filter Reference](https://www.wireshark.org/docs/dfref/f/frame.html)
- [Wireshark UDP Display Filter Reference](https://www.wireshark.org/docs/dfref/u/udp.html)
- [Wireshark Capture Offloading](https://wiki.wireshark.org/CaptureSetup/Offloading)

[返回 2026-07-30 文档总览](README.md)
