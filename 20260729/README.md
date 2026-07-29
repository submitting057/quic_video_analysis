# 2026-07-29：PCAP Record、UDP Datagram 与 QUIC Packet

本目录记录 2026 年 7 月 29 日新增的讨论，重点解释 PCAP Record、UDP Datagram、QUIC Packet 和“QUIC PCAP”的定义、区别与嵌套关系，以及这些概念对 packet-level 数据预处理的影响。

## 主题文档

1. [PCAP Record、UDP Datagram、QUIC Packet 与协议分层](01-PCAP文件格式与QUIC协议分层.md)

## 核心结论

- PCAP Record 是抓包文件中的存储记录，不是网络协议报文单位。
- UDP Datagram 是一个 UDP Header 加 UDP Payload。
- QUIC Packet 位于 UDP Payload 内，一个 UDP Datagram 可以承载一个或多个 QUIC Packet。
- “QUIC PCAP”通常只是包含 QUIC 流量的普通 PCAP/PCAPNG 文件，并非独立文件格式。
- packet-level 训练应明确事件层级；无密钥旁路分析推荐以完整 UDP Datagram 为基础事件。
- IP 分片、抓包截断和网卡卸载会破坏 PCAP Record 与 UDP Datagram 的近似一一对应关系。

> 合规说明：视频流量指纹可能暴露用户的观看兴趣和其他敏感偏好。研究和实验应仅使用自采、获得明确授权或合法公开的数据。
