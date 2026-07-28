# PCAP 数据预处理、标准化与会话构建

PCAP 预处理不是简单转换成 CSV，而是建立可追溯、可复现、不会泄漏标签的数据集。

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

## 确定分析单位

| 分析单位 | 一条样本是什么 | 适合场景 |
| --- | --- | --- |
| 数据包级 | 一个捕获记录或 UDP Datagram | 包长和到达间隔分析 |
| 五元组流级 | 一组相同五元组的数据包 | 流量统计和异常检测 |
| 会话级 | 一次完整播放或交互 | 视频 ID 识别和性能分析 |
| QUIC 连接级 | 关联到同一逻辑 QUIC 连接的包 | 迁移和连接行为研究 |

对于 YouTube 视频识别，监督学习样本应优先定义为一次独立播放会话，而不是单个包或单个五元组。

## 文件质量检查

检查：

- PCAP 或 PCAPNG 类型。
- LinkType、`snaplen` 和时间戳精度。
- 是否截断、损坏、重复或混合多个接口。
- 捕获点、方向锚点和时间同步方式。
- 抓包点是否受 GRO/GSO 等网卡卸载影响；主机侧出现超过 MTU 的“巨型 UDP 包”可能是抓包伪影。

常用工具：

```bash
capinfos input.pcapng
```

如果出现 `incl_len < orig_len`，说明只保存了原始包的一部分。

## 过滤候选流量

可以使用 TShark 过滤候选 UDP/QUIC 流量，但不能只依靠 `udp.port == 443`：QUIC 不一定使用 443，443/UDP 也不一定都是目标视频流量。

```bash
tshark -r input.pcapng -Y 'quic || udp.port == 443' -w quic_candidates.pcapng
```

只过滤 `quic` 还可能遗漏未被当前 Wireshark 版本正确识别的 QUIC 包，因此应同时保存原始 PCAP 和过滤规则。

## 字段提取

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

## 时间标准化

保留原始时间戳，同时生成：

```text
relative_time[i] = timestamp[i] - session_start
IAT[i]           = timestamp[i] - timestamp[i-1]
```

不要把格式化日期或绝对 Unix 时间直接作为模型特征，以免模型记住采集批次。

## 方向标准化

方向约定可以任意，但必须全数据集一致。例如：

```text
服务器 → 客户端：+1
客户端 → 服务器：-1
```

有符号长度：

```text
signed_length = direction × udp_payload_length
```

客户端身份不能永远通过“小端口一侧”判断，应使用受控实验日志、已知端点或播放行为进行确认。

## IPv4/IPv6 和路径标准化

统一成：

```text
src_ip, dst_ip, ip_version, src_port, dst_port, protocol
```

生成模型输入时，不直接保存真实五元组，而是在每个会话内分配匿名 `path_id`，仅保留“哪些事件属于同一路径”的关系。

## 去重与异常处理

镜像端口、多接口抓包或文件合并可能产生副本，但不能简单按照包内容删除，因为真实重传、ACK 和控制包也可能相似。

建议为记录增加质量标记：

```text
is_truncated
is_malformed
duplicate_candidate
is_out_of_order_timestamp
```

只删除能够证明由抓包机制产生的重复副本。

## 会话构建

推荐保留三层数据：

```text
datagrams  逐 UDP Datagram 事件
paths      五元组或路径时期
sessions   一次独立播放实验
```

训练阶段最好由浏览器自动化或实验日志记录真实点击播放时间。部署阶段无法获得该时间时，需要额外的会话检测器或滑动窗口策略。

## 数据组织

```text
sessions:
  session_id, video_id, device, browser, codec,
  resolution_policy, network_profile, collection_date,
  observation_start, split

datagrams:
  session_id, event_index, relative_time,
  direction, udp_payload_length, path_id

quality:
  session_id, truncated_count, malformed_count,
  duplicate_candidate_count, capture_notes
```

原始 PCAP 应只读保存，结构化大表优先使用 Parquet。

## 数据划分

必须先按完整的 `playback_run_id` 或 `session_id` 划分训练、验证和测试，再从各集合内部生成窗口、Burst 和数据增强。

错误方式：

```text
先把一次播放切成 100 个窗口
→ 随机把 80 个放训练集、20 个放测试集
```

正确方式：

```text
先把完整播放会话分配到某个 split
→ 只在该 split 内生成所有窗口和增强样本
```

同一次播放产生的连接、窗口、Burst 和增强样本不能跨集合。

[返回本次讨论总览](README.md)
