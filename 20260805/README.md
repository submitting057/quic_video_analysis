# 2026-08-05：可扩展 PCAP 预处理方案与实现规范

## 概述

本目录在 [20260804](../20260804/README.md) 单份真实 YouTube QUIC 抓包验证的基础上，给出面向批量数据集和多类 Packet-Level 深度网络的完整 PCAP 预处理规范。

本方案不绑定某个既有模型或代码仓库，而是将预处理拆成三层：

```text
原始 PCAP、metadata 与采集日志
  → 可追溯审计层
  → 模型安全的统一 Datagram 事件层
  → CNN / TCN / RNN / Transformer / Burst / 多视图适配层
```

所有会受数据分布影响的参数，包括固定序列长度、Top-K 路径数、归一化统计量、包长分档、Burst 间隔和自动播放定位阈值，均通过批量审计后仅在训练集上拟合。`20260804` 中单份 PCAP 的数值只作为未来 golden regression 的预期，不作为全数据集常量。

## 核心结论

- 一个基础事件定义为一个成功重组的 UDP Datagram，而不是 PCAP Record、IP Fragment 或单个 QUIC Packet。
- 原始事实、流量选择证据、标签和模型输入必须分层存储；模型表不得包含 IP、端口、CID、文件名、绝对时间等 shortcut 字段。
- 基础事件使用无符号 `udp_payload_length` 和明确的 `direction={c2s,s2c}`，正负号只在模型适配器中生成。
- 第一遍解析设备相关的全部 UDP；`quic`、UDP/443、DNS、握手和播放时间只作为候选证据，不能作为单一硬过滤条件。
- 五元组路径是主要可观察单位；CID 用于审计和辅助关联，不强行恢复不可验证的 QUIC 逻辑连接。
- 完整 ragged 事件序列永久保留；固定长度、采样、Padding、Mask、时间 Bin 和 Burst 均属于可重建的模型适配操作。
- 路径排序、窗口截取和参数拟合不得读取观察窗结束之后的信息。
- 默认研究主线使用包长、方向和时间等元数据；密文或包头字节仅作为关闭状态的受控实验视图。

## 阅读顺序

1. [PCAP 预处理总体架构与数据语义](01-PCAP预处理总体架构与数据语义.md) — 数据单位、质量处理、方向、路径、播放阶段和划分边界
2. [批量数据审计与参数标定](02-批量数据审计与参数标定.md) — 批量画像、质量门、训练集拟合和参数冻结
3. [多类深度网络输入适配方案](03-多类深度网络输入适配方案.md) — 事件序列、时间 Bin、分路径、Burst、Transformer 和多视图输入
4. [程序接口、配置与验收规范](04-程序接口配置与验收规范.md) — 数据契约、模块接口、配置枚举、错误处理和测试要求

## 方案边界

- 当前工作区没有原始 PCAP 和配套日志，因此本目录不声称完成了新的批量实测。
- `googlevideo.com` 只能标识 YouTube 媒体候选服务，不能在无密钥旁路场景中可靠区分视频、音频和同域控制数据。
- 本方案服务于自采、获得授权或合法公开的数据；原始抓包及可识别网络标识应按敏感数据管理。
- 本目录定义的是可直接实施的程序契约，但不依赖或背书工作区中任何现有 Packet-Level 模型实现。

## 主要参考依据

- [RFC 9000：QUIC Transport](https://www.rfc-editor.org/rfc/rfc9000.html)
- [RFC 9312：Manageability of the QUIC Transport Protocol](https://www.rfc-editor.org/rfc/rfc9312.html)
- [Wireshark capinfos Manual](https://www.wireshark.org/docs/man-pages/capinfos.html)
- [Wireshark UDP Display Filter Reference](https://www.wireshark.org/docs/dfref/u/udp.html)
- [Wireshark QUIC Display Filter Reference](https://www.wireshark.org/docs/dfref/q/quic.html)
- [QCSD: A QUIC Client-Side Website-Fingerprinting Defence Framework](https://www.usenix.org/system/files/sec22-smith.pdf)
- [Beauty and the Burst: Remote Identification of Encrypted Video Streams](https://www.usenix.org/system/files/conference/usenixsecurity17/sec17-schuster.pdf)
- [Deep Fingerprinting](https://arxiv.org/abs/1801.02265)
- [Var-CNN](https://arxiv.org/abs/1802.10215)
- [ET-BERT](https://arxiv.org/abs/2202.06335)
- [MIETT](https://ojs.aaai.org/index.php/AAAI/article/view/33748)
- [Rosetta](https://www.usenix.org/conference/usenixsecurity23/presentation/xie)
