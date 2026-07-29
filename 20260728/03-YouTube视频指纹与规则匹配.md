# YouTube 视频指纹、Chunk 概念与规则匹配流程

在没有会话密钥的被动抓包中，通常无法直接读取 HTTP/3 URL 和 YouTube `video_id`。视频识别依赖加密流量仍暴露的侧信道：

```text
方向、UDP 载荷长度、到达时间、下载突发和多尺度节奏
```

这是一种统计推断，输出应带置信度，并允许拒识为 `unknown`，不能将预测当作确定事实。

## 指纹库

指纹库是已知视频与参考流量模式的映射：

```text
video_id
→ 多次独立参考播放
→ 每次播放的包/Burst 特征序列或学习 embedding
```

显式规则指纹示例：

```text
video_A:
  template_1 = [420 KB, 680 KB, 510 KB, 760 KB, ...]
  template_2 = [405 KB, 700 KB, 495 KB, 740 KB, ...]
```

同一视频需要多个模板或原型，因为其流量会受到以下因素影响：

- 清晰度和自适应码率轨迹。
- VP9、AV1、H.264 等 Codec。
- 浏览器、设备和客户端版本。
- 带宽、RTT、丢包、MTU 和拥塞。
- CDN、缓存、广告和预加载。
- YouTube 重新编码或内容更新。

指纹记录应包含模板来源和采集条件：

```text
video_id
template_id
source_session_id
representation / resolution
codec
client_type
collection_date
network_profile
feature_sequence
```

## 三种 Chunk

“Chunk”在该领域经常被混用，应区分三类。

### Media Segment

应用层真正的视频或音频分段：

```text
视频时间 0～4 秒   → Segment 0
视频时间 4～8 秒   → Segment 1
视频时间 8～12 秒  → Segment 2
```

同一时间段可能有多个清晰度和 Codec 版本。无密钥被动 QUIC 抓包通常不能准确观察其边界。

### Observed Burst

从 PCAP 中根据连续下载和空闲间隔推测的网络突发：

```text
连续下行大包
→ 短暂空闲
→ 再次连续下行大包
```

每个 Burst 可以描述为：

```text
start_time
end_time
downstream_bytes
upstream_bytes
packet_count
preceding_idle_time
```

但必须记住：

```text
一个 Observed Burst 不一定等于一个 Media Segment
```

原因包括：

- 一个媒体分段可能被拆成多个传输突发。
- 音频和视频可能分别下载。
- 多个分段可能并发或连续下载。
- HTTP/3 会复用多个 QUIC Stream。
- 丢包恢复和拥塞控制会改变突发形状。

### Model Window

人为切分的模型计算窗口，例如：

```text
每 5 秒一个窗口
每 256 个 Datagram 一个窗口
每 20 个 Burst 一个窗口
```

它没有应用层语义。建议分别命名为 `media_segment_id`、`burst_id` 和 `window_id`。

## 规则匹配的离线建库

### 1. 采集有标签数据

对每个视频进行多次独立播放，并覆盖不同条件：

```text
video_id × 日期 × 设备 × 浏览器 × 网络条件 × 清晰度策略 × 重复次数
```

每个视频只采集一条 PCAP 会让系统很容易记住偶然网络条件。

### 2. 提取标准事件序列

每个 UDP Datagram 建议保留：

```text
session_id
event_index
relative_time
direction
udp_payload_length
inter_arrival_time
anonymous_path_id
```

不要将 IP、端口、CID 原值或 CDN 地址作为视频身份特征。

### 3. 检测 Observed Burst

一种简单规则：

```text
相邻下行事件间隔 < idle_threshold
→ 属于同一 Burst

相邻下行事件间隔 ≥ idle_threshold
→ 开始新 Burst
```

`idle_threshold` 必须只使用训练集标定，并记录到实验配置中。

### 4. 提取 Burst 特征

```text
downstream_bytes
upstream_bytes
duration
packet_count
max_packet_size
previous_gap
mean_iat
large_packet_ratio
small_packet_ratio
```

### 5. 生成视频模板

一次播放会话可以表示为：

```text
F = [
  (burst_bytes_1, gap_1, duration_1),
  (burst_bytes_2, gap_2, duration_2),
  ...
]
```

每个视频应保留多个模板，不应过早求成单一平均序列。

## 在线匹配

```text
待识别播放流量
→ 使用相同预处理构造 Burst 序列
→ 使用总字节数、前几个 Burst 等进行候选召回
→ 使用 DTW / 编辑距离 / 序列对齐精确匹配
→ 最佳得分不足或与第二名差距太小时输出 unknown
```

DTW 或序列对齐比逐项欧氏距离更合适，因为观测序列可能出现拆分、合并、缺失和额外 Burst：

```text
参考模板：[A, B, C, D]
实际观测：[A, B1, B2, C, D]
```

输出至少包含：

```text
predicted_video_id
matching_score
second_best_score
confidence
observed_duration
```

拒识规则是实际系统的必要组成部分：

```text
最佳得分不够好
或最佳与第二名差距太小
→ unknown
```

## 规则方法的数据组织

```text
sessions:
  session_id, video_id, device, browser, codec,
  network_profile, collection_date, split

datagrams:
  session_id, event_index, relative_time,
  direction, udp_payload_length, anonymous_path_id

bursts:
  session_id, burst_id, start_time, duration,
  downstream_bytes, upstream_bytes, packet_count, preceding_gap

fingerprints:
  video_id, template_id, source_session_id,
  collection_conditions, burst_feature_sequence
```

原始 PCAP 只读保存，结构化大表优先使用 Parquet，变长模板可以使用 Parquet List、JSON 或 NumPy 数组。

[返回本次讨论总览](README.md)
