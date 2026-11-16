# YouTube 经验迁移 Snapchat：pilot 解密审计后的要点与行动清单

日期：2026-11-17（三轮修订至本文版本，更正历史见 §七）　性质：**实测结论文档**
（基于本地实验室采集的 10 个 Snapchat Spotlight pilot 会话 + sslkey 离线解密；
文末附录列出分析脚本清单与可复现命令备查）。

前置阅读：platform_transferability.md（跨平台可迁移性理论）、SNAP_PILOT_20261116.md（首轮审计）——均位于源研究工作区，未随本文入库；本文按"结论→证据→还缺什么"自洽组织，可独立阅读。
本文按"结论 → 证据 → 还缺什么"组织，供下一轮采集和决策使用。

---

## 〇、读前须知：术语、数据、结论速览

### 0.1 术语表（首次出现即定义，后文直接用）

| 术语 | 含义 |
|---|---|
| **deeplink** | 采集脚本用 `https://www.snapchat.com/spotlight/W7_…` 链接直接打开目标视频的方式（不经人工刷 feed），是我们已知的强时间锚点 |
| **token（W7_…）** | 分享链接里的视频标识，采集侧当视频 ID（VID）用；内嵌 9 字符"种子 ID"（如 yfjnusddk） |
| **对象 ID** | CDN 上一个文件的 20 字符标识（如 `zbPR55HxLpLvEQS6yI4Zu`）；`.1025/.1395/.1398` 是同一对象的编号变体（不同档位/资产类型，具体语义未完全判定，.1395 已证实是视频档） |
| **stream（HTTP/3 流）** | 一条 QUIC 连接内并行的一条"请求-响应"通道，按 stream_id 区分；**加密下不可见**，解密后可见 |
| **黑盒侧 / 解密侧** | 黑盒 = 运行时识别视角，只看得到 IP/SNI/包大小与节奏；解密侧 = 拿 sslkey 离线看 URL 和响应头，只用于建标注、做校验、当 ground truth，不进模型 |
| **0-RTT / 1-RTT** | QUIC 会话恢复时的"早数据"包 / 常规应用数据包，用不同密钥加密 |
| **etag** | 响应头里的内容指纹（同一文件跨请求恒定），可当对象身份证用 |
| **no_media** | 会话内视频通道零下行的入库标记（拒收或单独归类） |
| **便宜独立门** | 不训练、一遍推理的检验：辅助通道成员与视频成员在冒充者上的错认一致率，<10% 判"独立"（随机基线约 0.26%） |
| **geo4 / 45ms 切段** | YouTube 上的现役最优系统（通道拆分 + 几何平均组合）／按 >45ms 停顿切段提段特征的老做法 |

### 0.2 本批数据是什么

10 个会话、8 个唯一视频（2 个视频各采 2 次），同一台 Redmi（Android 15，
Snapchat 14.31.0.56）、同一 WiFi（出口经香港，DNS=8.8.8.8 明文），采集动作为
"停应用 → 清 cache/code_cache → 起 tcpdump（**GRO 关**）→ deeplink 打开目标视频
→ 录 10 秒"。每会话附带 sslkey（frida 抓的 TLS 密钥）。

| 会话 | 采集时间 | 种子 ID | 媒体 GET | 解密后判定 |
|---|---|---|---|---|
| 00ba8b03 | 11-12 19:18 | yfjnusddk | ✅ 2.4MB（两个视频对象+封面） | **唯一完整播放（下称"好会话"）** |
| 0ba36679 | 11-12 00:27 | cxtudekwq | 🟡 两条 mp4 GET，1 条续传 23.8KB 后停滞 | 半程（详见 §四·观测 E） |
| 0ac6e20e | 11-12 16:32 | tzjfetvdm | ❌ 零 | 零媒体 |
| 0b58797c | 11-12 17:37 | vnquldktp | ❌ 零 | 零媒体（该视频第 1 次） |
| 0a6139d7 | 11-12 22:34 | grlkunfzj | ❌ 零 | 零媒体 |
| 0a1a4f0b | 11-13 00:09 | ftqqtacoo | ❌ 零 | 零媒体 |
| 0a5c894e | 11-13 01:09 | tzjfetvdm | ❌ 零 | 零媒体（该视频第 2 次） |
| 00c18f95 | 11-13 17:34 | pcbsihfgv | ❌ 零 | 零媒体，但 /context/spotlight 与 feed API 有调用 → 链接送达、播放未启动 |
| 00f025ce | 11-13 19:34 | dwsvjpdxmp | ❌ 零 | 同上 |
| 0a31b6bc | 11-13 19:36 | vnquldktp | ❌ 零 | 零媒体（该视频第 2 次） |

**先记住一个数字：10 个会话里 9 个没有下载任何视频字节。** 这意味着本批数据
能回答"通道结构长什么样、归因工具够不够用"（解密侧问题），回答不了"流量里
有没有身份信号、跨播放稳不稳定"（识别侧问题）——后者需要修好采集后重采。

### 0.3 结论速览

| # | 问题 | 判定 | 关键证据 | 还缺什么 |
|---|---|---|---|---|
| 1 | 视频流量存在、可归因 | ✅ 已证 | 好会话 2.4MB 单连接；全部 GET URL/响应头（etag、content-range）解密可读 | — |
| 2 | 目标 vs 推荐视频可区分 | ✅ 机制成立 | token 回显 + 对象 ID + 时序三层手段（§2.3） | "token→URL 对号"这步因元数据走缓存未及演示，重采后自然解决 |
| 3 | 辅助通道有独立证据 | 🟡 首个正面证据 | /context/spotlight 响应按视频变化（含音乐元数据明文） | 重采后跑便宜独立门量化 |
| 4 | 黑盒能拆开一条连接里的多路对象 | 🟠 一半 | 封面=独立突发可分；两个视频交织不可分（§三） | 组成稳定性的统计（重采后） |
| 5 | 缓存清干净了吗 | ❌ 没清（实锤） | 断点精确对齐 KiB + 文件头从未上网（§四观测 A-D） | 判决实验定位藏匿目录 |
| 6 | 采集协议可靠吗 | ❌ 9/10 无媒体 | 零媒体 GET；9 会话两两余弦相似 61-99% | P0 全套修复 |

**总现状一句话**：解密侧的归因与验证工具链已经全部打通（这是本批最大的收获）；
识别侧的一切检验被采集协议卡住，先修 P0 再谈迁移成败。

---

## 一、背景：为什么"辅助通道"是 YouTube 方案的命门

YouTube 现役最优系统（通道拆分 geo4）的原理：把流量按"跟谁通信"拆成几摊
（视频服务器、封面图服务器、API），每摊单独建模，最后概率相乘（几何平均 =
AND 门，都说像才算像）。单个模型总会有过度自信的时候；让几个**看的材料
完全不同**的模型同时被骗，概率极小。项目反复验证的铁律：给模型换脑子
（架构/损失/训练法）没用，错误结构由信息源决定；只有换材料（看不同通道）
才能买来独立性。

迁到 Snapchat，辅助通道要过两关：

1. **存在关**：有跟视频不同的其他流量摊（封面、API、元数据…）；
2. **信息关（更严）**：辅助摊的字节要**依赖视频身份**——YouTube 的封面图每个
   视频都不一样，所以封面通道自己也带身份信号。如果辅助通道对任何视频都长
   一个样（纯平台常数），它提供不了独立证据，AND 门里那一路等于空转。

§二回答存在关，§0.3 第 3 行回答信息关的现状。

---

## 二、要点 1：Snapchat 能否分离"视频流量"与"辅助流量"

### 2.1 域名级分离：半成立

SNI/DNS 全部可见（QUIC v1 Initial 包可解密提出 SNI、TCP SNI 明文、DNS 明文），
域名图谱如下。注意"视频依赖性"一列回答的是存在关，不是信息关：

| 通道 | 域名/归属 | 实测内容 | 视频依赖性 |
|---|---|---|---|
| 媒体 | cf-st.sc-cdn.net (CloudFront) | **视频 + 封面 + 其他媒体对象混在同一个 QUIC 连接的不同 stream 上**；另有专用缩放端点 `/aps/bolt/<base64原图URL>_RS0,640_FMwebp/png` 出封面变体 | ✅ 但混布 |
| bolt 资产 | bolt-gcdn.sc-cdn.net (GCP) | 恒定部分 = `GET /sc-bolt-nm-prod/network_mapping.bin`，**全 10 会话响应体同为 25,345B、同 SHA-256**（CDN 调度配置，与视频无关）；偶发 `/z/<对象ID>.<编号>` 媒体变体（好会话 zbPR.1025，12KB，疑海报档） | ❌ 常数 + 🟡 偶发媒体对象 |
| API | *.api.snapchat.com（GCP/AWS 十余个子域） | deeplink 后 +0.13s 启动的一批请求；**/context/spotlight 响应按视频变化**（§2.3） | 🟡 首个正面证据，幅度小（1.6-3.8KB/次） |
| 遥测 | app-analytics-v2 / aws.duplex | 形态恒定的噪声 | ❌ |
| 系统噪声 | tracking.miui.com / loc.map.baidu.com | MIUI 系统遥测混入窗口 | ❌（可过滤） |

两个容易踩的坑，都已实测定性：

- **network_mapping.bin 陷阱**：每个会话都有它（好会话 bolt 流 43.6KB = 25.3KB
  network_mapping + 12KB 媒体变体；其余会话 ~31KB 几乎纯它）。把 bolt 流当
  视频指纹用，等于把平台调度常数混进视频通道；
- **cf-st 不是纯视频域**：同一域名既出 MB 级视频、又出 49KB 封面、又出 API 内
  嵌的静态图 URL——通道划分必须做到流级，域名级不够。

### 2.2 流级分离：封面可以，其余不行

好会话的 cf-st 连接（一条 QUIC 连接，4 条并行 stream），解密后 stream↔URL↔字节
已完全钉死（tshark 复核）：

| stream | GET（对象.变体） | 响应 | 判定 |
|---|---|---|---|
| 0 | `/d/lFq9TIfQVc2tW1tiq4GK5.1398` | `content-range: bytes 289776-2067135/2067136`，etag `b7a72e72…b9a3`，1.78MB | 视频对象甲（2.07MB 文件的尾部续传段） |
| 4+8 | `/d/zbPR55HxLpLvEQS6yI4Zu.1395` ×2 | bytes 0-84991 + 84992-598463，共 598KB | 视频对象乙（**从 byte 0 完整拉取**） |
| 12 | `/aps/bolt/aHR0cHM6…SEIxekJvOTF3aURwbTE0ZG9uWjhl…_RS0,640_FMwebp` | content-type image/webp，body 魔数 RIFF/WEBP，49KB | **封面**（URL 的 base64 解开就是原图地址 + 640px webp 缩放参数） |

（bolt 连接上另有 `/z/zbPR….1025`，12KB，先于两个视频 70ms 拉取——同一对象乙
的另一变体，poster 档特征。）

**解密后全事件时间线**（本文唯一权威时间轴，§5.1 的链与本表同源；
`pcap 起`=tshark frame.time_relative，`deeplink 起`≈pcap 起减 0.7s；
DeltaSync/Valis 等纯后台杂讯归并显示）：

```
══ 好会话 00ba8b03（唯一完整播放）══
── ① 启动 API 潮（deeplink 后 ~0.3s 起，App 常规启动流量，与视频内容无关）──
+0.99  +0.29  POST app-analytics/bz2（遥测）
+1.00  +0.30  POST aws-proxy-gcp …/SyncConversations（消息同步）
+1.01  +0.31  POST us-central1 …/sync_groups + batch_stories×2（feed 元数据）
+1.05  +0.35  POST us-central1 …/batch_story_lookup（故事查询）
+1.16  +0.46  POST gateway/Connect、GetRanking、IsPrimary、SyncFriendData
+1.40  +0.70  POST RegisterDevice、DeltaSync×7、listManagedPublicProfiles
+1.47  +0.77  POST CircumstancesService/targetingQuery
── ② CDN 资产 + 广告（deeplink 后 ~0.8s）──
+1.52  +0.82  GET  bolt network_mapping.bin        （25.3KB 平台常数）
+1.53  +0.83  GET  bolt /z/zbPR….1025              （12KB 海报档，先于视频 70ms）
+1.56  +0.86  POST us-central1 adserver INTERSTITIAL_SPOTLIGHT（插屏广告）
── ③ 媒体主体（deeplink 后 ~0.9s 起，1.2s 内全部完成）──
+1.60  +0.90  GET  cf-st /d/lFq9….1398   → stream 0，1.78MB（206 续传段）
+1.60  +0.90  GET  cf-st /d/zbPR….1395   → stream 4，85KB（bytes 0-84991）
+1.80  +1.10  GET  cf-st /d/zbPR….1395   → stream 8，513KB（84992-598463）
── ④ 上下文记账 ×2（媒体完成之后）──
+1.91  +1.21  POST /context/spotlight #1 → 回显目标 token（3763B，含音乐元数据）
+2.59  +1.89  POST /context/spotlight #2 → 回显推荐位 token（1595B）
+2.96  +2.26  GET  cf-st /aps/bolt/…HB1z…FMwebp → 49KB 封面（归属见 §2.4）
── ⑤ 空闲窗杂讯（媒体完成 7s 后，与视频无关）──
+9.5~+12.9    DeltaSync×N、Valis、Fidelius、analytics、scauth/validate×3
+3.55  +2.85  （参考）采集脚本 "Start watching" 标志——远晚于媒体完成

══ 半程会话 0ba36679（起播后中止）══
+1.08~+1.55  启动 API 潮 + network_mapping（形态同上）
+1.60        POST /context/spotlight #1 → 回显 14 位数字 ID（63175374911488）
+2.59        POST /context/spotlight #2 → 回显另一 14 位数字 ID（63179065233408）
+2.68        GET cf-st /d/TGcx….mp4      → stream 2，响应 0 字节
+2.68        GET cf-st /d/Ajha.1301.mp4  → stream 4，30.8MB 文件断点续传 23.8KB 后停滞
+2.78        GET /3ebbd180-…（cf-st 一条 TCP 流上的小对象）
+9.5~+11.6   空闲窗杂讯

══ 零媒体会话 00c18f95（deeplink 送达、媒体层零动作）══
+0.26~+0.30  启动 API 潮（含 batch_stories + mixed_feed/stories）
+0.72        POST adserver INTERSTITIAL_SPOTLIGHT
+0.82        POST /context/spotlight #1 → 回显本会话目标 token（pcbsihfgv）
+2.68        POST /context/spotlight #2 → 回显推荐位 token（zrrfmfukx，不在本批清单）
+8.7~+12.0   prod/ad/init、network_mapping(+8.98)、scauth —— 媒体层零动作
```

三形态对照（同一采集协议下的三种命运）：

| 维度 | 好会话 | 半程 | 零媒体 |
|---|---|---|---|
| /context/spotlight | 2 条，token 回显 | 2 条，**数字 ID 回显** | 2 条，token 回显 |
| 媒体 GET | 2 视频 + 1 封面，2.4MB | 2 条 mp4，0 + 23.8KB 后停 | **无** |
| context 与媒体的先后 | 媒体先行 0.1-0.3s | context 紧贴媒体**前** 0.09s | 有 context 无媒体 |

时间线直接给出的四个结构性结论：

1. **双 context/spotlight 是标准行为**：三种会话都是两条——第一条围着目标视频、
   第二条围着推荐位（好会话与 00c18f95 的第二个 token 都不在本批视频清单内，
   推荐位预载是 Snapchat 的常规动作，不是偶发）；
2. **context 回显有两种形态**：A=回显 W7_ token（好会话、00c18f95）；
   B=回显 14 位数字 ID（0ba366，两条响应除 ID 外几乎相同）。形态由请求参数
   决定，与播放成败无关；
3. **context 与媒体 GET 的先后不固定**：好会话里媒体先行（URL 由本地缓存元数据
   驱动，§2.4 零命中证据）；0ba366 里第二条 context 紧贴媒体 GET 前 0.09s。
   它时而像触发器、时而像事后记账——角色随缓存态/会话路径而变，**但 token
   回显的归因价值两种形态下都不变**；
4. **零媒体会话死因收窄**：00c18f95 的目标 token context 已成功解析（+0.82s）
   → deeplink 确实送达、App 已处理目标视频上下文，死在媒体层（整文件缓存或
   起播被拦）——不是"链接没打开"。

黑盒视角（45ms 停顿切突发）对照这份真值：

- **封面 vs 其他：本会话可分**——封面是独立突发（晚于媒体突发约 1.3s、49KB、
  突发内纯度 100%）。但这是 n=1 观察：封面到达位置（先后、是否独立成突发、
  走不走同连接）可能随下载调度和缓存态变化，启发式是否普遍成立要重采验证；
- **视频甲 vs 视频乙：不可分**（至少本样本如此）——两者交织在同一个 2.37MB
  突发里，0.4 秒内流间切换 840 次，包大小 96.9% 恒为 1452B，无任何包级签名；
- **n=1 警告**：另一会话（0ba366）的媒体是单对象纯突发——"混不混"本身取决于
  下载调度和缓存态，组成本身不稳定。

### 2.3 token 级归因：目标与推荐视频的区分

**核心发现：`/context/spotlight` 的响应逐字回显 W7_ token。** 先澄清它的定位
（曾有反复，见 §七更正 3；§2.2 时间线的结论 3 是完整版）：它**不是稳定的
下载触发器**——好会话里媒体 GET 先于两条 context（视频 URL 来自本地缓存元数据），
0ba366 里它又紧贴媒体 GET 之前。角色随缓存态/会话路径而变，但**token 回显
这个归因锚点在两种形态下都成立**，是解密层最强的"这条流量属于谁"证据：

- 好会话两条 context/spotlight 响应分别回显目标 token（…eWZmanVzZGRr… =
  yfjnusddk）与**另一个 token**（…YWNneWZvYnFp… = acgyfobqi，**不在本批 8 个
  视频清单内**）——同一 10s 窗口 App 处理了第二个视频的上下文，**推荐位预载
  实锤**；
- 目标那条响应（3763B）内嵌**音乐元数据明文**（"LOVE." / Kendrick Lamar, Zacari
  / lensid / 音频预览 cf-st URL）；另一条 1595B、无音乐块——**响应内容按视频
  显著不同**；
- 由此，目标/推荐的区分有三层手段（按可信度降序）：
  1. **金标准**：context/spotlight 响应里的 token 回显定"哪个上下文属于谁"，
     再从 feed API 响应（batch_stories/mixed_feed）把各视频的媒体 URL 对号——
     全离线、现有 sslkey 即可；
  2. **银标准**：媒体 GET 的对象 ID + per-stream 字节归属（上表：1.78MB/598KB/
     49KB 谁是谁一目了然）；
  3. **铜标准**：时间+大小启发式（黑盒运行时唯一可用层：deeplink 后首个 MB 级
     突发=视频，其后 ~1s 的小突发=封面）；
- **对信息关的意义**：这是 API 通道字节随视频变化的第一个正面证据（此前 9 个
  无播放会话里 API 逐字节恒定，一度全负面）。幅度 KB 级；"每播放是否重发"
  待重采统计。

### 2.4 边界案例：S12 封面属于目标还是推荐？（当前无解，倾向后者）

S12 封面的原图对象是 `HB1zBo91wiDpm14donZ8e`（缩略图 URL 的 base64 内嵌），
它既不等于 lFq9 也不等于 zbPR——归属只能靠"哪个 API 响应把它和哪个 token/视频
URL 写在同一条故事条目里"判定。实测：把好会话**全部解密字节落盘**
（`snap_dump_bodies.py`，6 条流 2.49MB），grep 三个对象 ID 和两个种子 ID——
**全部零命中**。结论：

1. **媒体 URL↔故事的映射元数据没走网络**（App 用本地缓存的 feed 元数据直接
   发 GET——"缓存没清"问题的又一面）。所以单凭本 pcap，S12 归属无解；
2. **时序给出强倾向**：S12 封面 GET（+2.96s）距推荐 token 的上下文记账
   （+2.59s）仅 0.37s，距目标的是 1.05s。该读法下整链自洽——目标=zbPR
   （.1025 海报先行 70ms → .1395 视频，deeplink 播放的教科书序列）、推荐=lFq9
   （1.78MB 全量预取）+ HB1z 封面（up-next 卡片渲染）。**倾向 S12=推荐视频的
   封面**，但 n=1 推断，不能当结论用。注意 0ba366 也有同样的"第二条 context →
   0.09s 后立刻取对象"邻接模式（那里取的是两条 mp4）——邻接性证明"取的动作
   跟着第二条 context 走"，但**不能证明取的是什么**，最终仍需映射元数据；
3. **拿铁证的两条路**：①重采清缓存后映射必然走网，故事条目里 token+视频 URL+
   封面 URL 同现，一次 grep 判定（被 P0 修复自动解锁，零额外成本）；②立即判决：
   直接从 CDN 拉 lFq9/zbPR/HB1z 三对象，封面与两视频首帧做感知比对（URL 无签名
   参数、可能仍有效；属主动外网请求，执行前需单独确认）。

### 2.5 对"辅助通道判定"的直接影响

1. **封面图通道存在且有身份信号潜力**（每个视频封面不同、专用缩放端点+API
   内嵌变体 URL 双路可见），但与视频同连接同域名——辅助证据的"独立性"只能
   来自流级拆分，不是域名级。比 YouTube 的 i.ytimg.com 分域结构天然弱一档；
2. **API 通道是唯一的域名级可分候选**，时间特征明确（deeplink 后第一个 API
   突发就是围绕这个视频的调用）。信息关已有首个正面证据（§2.3），净判定等
   重采后跑便宜独立门（杀线 <10%）；
3. **bolt 域不是纯背景域**：恒定的 network_mapping（平台常数，绝不能当视频
   指纹）之外偶发每视频对象（/z/ 变体）。重采后统计"/z/ 对象出现率与归属"；
   眼下最稳的用途是当采集健康度探针——哪天连 network_mapping 都没了，说明
   采集环境变了。

---

## 三、要点 2：多路复用下，黑盒（不解密）怎么分离

§2.2 已给出结论：封面可分、视频彼此不可分。这里只讲怎么办，按成本递增：

1. **时间+大小+顺序启发式（首选）**：deeplink 是强时间锚——"deeplink 后第一个
   MB 级 CDN 突发=视频主体；其后 ~1s 的小突发（几十 KB）=封面"。此规则在唯一
   样本上成立，重采后先验证命中率，别急着上模型；
2. **解密真值训分类器（可行但要样本）**：sslkey 采集侧本来就有，可离线批量
   生成带标签数据（每条突发属于哪个对象、URL 是什么），训一个黑盒特征
   （时间/大小/间隙形态）的突发分类器。需要几百条带标签突发，前置是采集修复；
3. **干脆不分离（条件方案）**：如果"每视频的下载组成"是确定的（视频尾部 +
   固定第二对象 + 封面），那混合突发整体就是该视频的指纹，拆开反而丢信息。
   **但这个条件成立的前提恰恰是缓存清干净**——缓存态会让 range 组成漂移
   （§四）。所以顺序是：先修缓存 → 看组成是否确定 → 再决定要不要分离。

配套现状（原"early secret 缺口"已证伪，见 §七更正 1）：

- URL 级真值用 tshark + 现有 keylog 就能拿到，**不依赖任何采集侧修复**；
- 分离规则的真值来源即 §2.3 的三层手段（金/银给训练与验证用，铜是黑盒运行时
  唯一可用层）。

---

## 四、要点 3：缓存没清干净——四条观测 + 判决实验

**主张**：采集脚本清了 `cache/` 和 `code_cache/` 两个子目录，但视频文件字节和
"已下载到第几 KiB"的断点账本存在别处、活了下来。表现为：目标视频在设备上已有
（部分）副本，本次会话只补拉缺口。

### 观测 A：range 起点不从 0 开始，且精确对齐 KiB 边界

用 tshark 直接看媒体流的响应头（命令在附录，已验证可复现）：

```
好会话 stream 0（lFq9 对象）：
  content-range: bytes 289776-2067135/2067136    ← 289776 = 283 × 1024，整
  etag: "b7a72e721ee8253378342e2811b9b9a3"
0ba366 会话 stream 4（Ajha 对象）：
  content-type: video/mp4
  content-range: bytes 156672-324983/30870237     ← 156672 = 153 × 1024，整
好会话另一对象的两段分界：84992 = 83 × 1024，整
```

三个边界全部精确落在 KiB 上 = 分块下载器的进度账本特征。播放器 seek 的话
位置由关键帧决定，不会恰好整 KiB；三个不同对象/会话同时整 KiB 更不可能是巧合。

### 观测 B：文件头部字节在整个抓包里从未上网

解密后可列出该连接**全部** stream 的响应头，逐条核对：

- 好会话：没有任何 stream 的 content-range 覆盖视频文件 [0, 289775)；
- 0ba366：没有任何 stream 覆盖 [0, 156671)；
- 两会话 cf-st 域仅剩的一条 TCP 流只有 6.3KB（装不下 283KB），其余通道
  （bolt/api/analytics）按域名和大小排除。
  （用 `scripts/analyze_snap_pilot.py <root>` 的 per-host 表可独立复核总量。）

### 观测 C：抓包窗口不可能错过头部字节的传输

operate.log 时间线（每个会话都有）：`停应用 → 清缓存 → tcpdump 启动 →
deeplink（仅晚 ~0.2s）`。tcpdump 先于 deeplink、应用冷启动——头部那 283KB 若
走了网络必然落在 pcap 里。没有，就说明它没走网络。

### 观测 D：对照组排除"播放器总是跳过文件头"

同一连接里 598KB 的对象乙是**从 byte 0 完整拉取**的（stream 4: 0-84991；
stream 8: 84992-598463）。如果播放器习惯性跳文件头（moov），它也该跳这个
文件。它没跳。

### 观测 E：零 GET 会话的两种死因 + 广告排除

- 0ba366 的两条 mp4 GET 与 stream 钉死：`AjhacsRUbUzd69i7wxamJ.1301.mp4` →
  stream 4，即观测 A 那条 30.8MB 文件的断点续传（实收 23.8KB 后停滞）；
  `TGcxiN0K1H7lVUwNpMyLe.mp4` → stream 2，**响应 0 字节**。"续传的大文件"和
  "零字节的 GET"是两个不同对象，谁是目标待重采后用 etag 校验器判定（§5.4）；
- **9 个零媒体会话解密后零媒体 GET**（连推荐位的预取 GET 都没有，只有
  network_mapping + API/广告）——与两个媒体会话"部分缓存、补拉缺口"的形态
  不同：要么**整文件已在缓存**（一字节没上网），要么**媒体层从未启动**。
  本批无法区分两者，但都指向同一修复（彻底清缓存 + 起播验证门）。00c18f95/
  00f025ce 有 /context/spotlight 与 feed 调用但零媒体 GET，说明 deeplink 送达、
  播放未启动——"整文件缓存"假说略占上风（上下文都过了、只差媒体层）；
- 插屏广告请求（adserver INTERSTITIAL_SPOTLIGHT）在**好会话里也发出**
  （frame 417）——广告请求与否不构成播放成败的分水岭，"插屏挡住起播"的
  怀疑排除。

### 判决实验（下一轮采集时做，成本极低）

1. 采集前 `adb shell pm clear com.snapchat.android` 再走 deeplink → 若首个
   content-range 变成 `bytes 0-…` 即坐实。**代价**：pm clear 清登录态，deeplink
   可能撞登录墙（这本身也要观测）；撞墙则改方案 2；
2. 方案 2（外科式）：播放一个视频前后各跑一次
   `adb shell su -c "du -a /data/data/com.snapchat.android | sort -n | tail -50"`
   和 `ls -la /sdcard/Android/data/com.snapchat.android/`，找出增长的目录，
   加进采集脚本的清理清单；
3. 无论哪种，入库前加**媒体存在门**：sc-cdn 上无 ≥500KB 下行且解密无媒体
   GET 的会话标 `no_media` 拒收。本批 9 个这样的会话两两余弦相似 61-99%、与
   token 无关——直接入池训练会学成"设备常数分类器"，闭集指标好看但零视频信息。

---

## 五、补充要点（前三条没覆盖、但同样卡迁移的）

### 5.1 采集标志点与时钟（原"观测窗锚点错位"的具体化）

§2.2 时间轴已显示：媒体在 deeplink 后 ~0.9s 就开始、~2.0s 完成，而采集脚本的
"Start watching" 标志在 +2.85s（deeplink 起）才出现——旧 10 秒播放窗以错误锚点
起算。采集脚本应按下表记录标志点（全部 epoch 写入 manifest）：

| 标志 | 含义 | 来源 |
|---|---|---|
| T0 | tcpdump 启动 | 采集端时钟（现有） |
| T1 | deeplink 发出 | 采集端时钟（现有日志有，需入 manifest） |
| T2 | 目标 Activity 获得焦点 | 现有焦点检查的时刻 |
| T3 | Start watching | 现有 |
| T4a/T4b | 媒体通道首个 MB 级突发起/止 | 离线从 pcap 判定后回写 manifest |

- **观测窗 = [T1, T1+5s]**；若 T4b 存在则收紧为 [T1, T4b+0.5s]。以 deeplink
  为锚，不再用 T3 起算；
- **时钟对齐是硬要求（时钟陷阱）**：手机 tcpdump 把本地时间当 epoch 写入
  （pcap 时间 = 真实时间 +8h，残差 ±1s），而 manifest 里
  phone_to_server_time_offset 填的是 0。任何 pcap↔日志联合分析都要先扣
  28800 秒，否则全错。采集脚本应在每轮抓包前后各执行一次
  `adb shell date +%s.%N` 与采集端对时，实测偏移写入 manifest；
- 入库字段：watch_window_start/end、media_burst_start/end、no_media、etag、
  file_total_bytes（供 5.4 校验器用）。

### 5.2 缓存态 = 第六干扰轴（理论文档未预判）

range 组成随缓存态漂移，直接冲击"同视频跨播放指纹一致"这一识别前提。修好
清理只是第一步，重采后还要量化"同视频跨播放 range 组成的一致率"；若 pm clear
后仍有残余漂移，指纹表示必须对起点不敏感（YouTube 管线的随机裁头增强思想
可迁移）。

### 5.3 keylog 与解密工具现状

采集到的 sslkey（五个标签：CLIENT/SERVER_HANDSHAKE_TRAFFIC_SECRET、
CLIENT/SERVER_TRAFFIC_SECRET_0、EXPORTER_SECRET）经 tshark 4.6.9 全量复核
**够用**：TCP/TLS 与 QUIC/HTTP3 的请求 URL、响应头（etag/content-range）、
DATA 全部可解，bolt 连接同样可解。曾有"请求在 0-RTT 解不开、bolt 无密钥"的
判断，复核证明是自研解密器的 bug 而非 keylog 缺口（详见 §七更正 1/2）。

仍然要做的两件卫生项：

- frida 钩子补抓 CLIENT_EARLY_TRAFFIC_SECRET：当前未见 0-RTT 承载请求，但
  App 版本更新后可能变化，提前保险；
- 把"tshark 也解不开的连接占比"当采集质量指标统计告警（这才是真缺密的信号）；
- 自研解密器（snap_decrypt_flow.py）有两处 bug 待修（keylog 按随机值匹配、
  客户端方向包号重建）；修好前 URL 真值一律以 tshark 为准。

### 5.4 跨会话一致性校验器（详细设计）

原理：sc-cdn 媒体对象响应头自带两个内容指纹——**etag** 和 **content-range
里的文件总大小 N**（`bytes x-y/N`）。同一 token 指向同一媒体文件：无论哪次
会话、什么缓存态、从哪个断点续传，etag 和 N 必须相同。校验器就是对
(token → {etag, N, object_id}) 表做断言：

| 规则 | 检查内容 | 触发时的含义 |
|---|---|---|
| R1 | 同 token 所有会话的 etag 集合大小 == 1（且 N 唯一） | ≥2 个 etag = 归因错误 / 平台换文件重编码 / token 复用 → 人工审 |
| R2 | 不同 token 命中同一 etag | 别的视频的流量被算到这个 token 头上（预取混入/时间窗切错）→ 告警 |
| R3 | 会话中该 token 无任何 etag 记录 | no_media 会话 → 拒收或标记 |
| R4 | 同 token 跨会话的 range 起点集合 | 量化缓存态漂移程度（§三要不要做分离的直接输入） |

实现：采集后离线批量解密（tshark 批处理 + `snap_h3_reassemble.py` 重组），每条
媒体响应落一行 (token, 请求URL, object_id, stream_id, etag, N, range, 实收字节)，
四条规则跑表上，结果回写 manifest。能抓的错误：deeplink 打开失败被误标、预取
流量被误归因、缓存命中被当播放、平台换文件、脚本 token 映射错。局限：仅覆盖
tshark 可解的连接。

### 5.5 单连接多流 vs TikTok 多连接

Snapchat 的多路复用发生在**一条 QUIC 连接内部**（stream 交织），不是 TikTok
那种多连接并行 range——理论文档里 TikTok 才需要的"通道×连接"表示层扩展暂不
需要；但"通道×对象"的流级分解变成新需求，性质类似。

---

## 六、行动清单（优先级序）

**P0 采集协议修复（不修则一切信号检验无效）**
1. 缓存：执行 §四 判决实验，确定清理清单（或 pm clear + 登录态处理）；
2. 窗口与标志点：按 §5.1 的 T0-T4 采集，观测窗 = deeplink 锚定（T1+5s 或
   T4b+0.5s）；每轮实测时钟偏移入 manifest；
3. 入库门：媒体存在断言 + §5.4 校验器（R1-R4，含 object_id 联表）+ **token
   回显校验**（context/spotlight 响应若为形态 A，必须含本会话 token，否则
   deeplink 未达——比 no_media 更早的失败信号；注意 §2.2 结论 2 的形态 B
   回显数字 ID、无 token 可验，这类会话靠媒体存在门 + etag 门兜底）；
4. keylog：按 §5.3 两个卫生项执行；自研解密器修 bug（不紧急）；
5. 重采 pilot：≥8 视频 × ≥10 播放（沿用原设计）。

**P1 归因与分离验证（P0 数据到位后）**
6. 批量解密生成 URL 级真值（tshark + snap_h3_reassemble.py，已验证可行）；
7. 验证"deeplink 后首 MB 突发=视频、其后小突发=封面"启发式命中率与混合突发
   组成稳定性；同时统计：①推荐位第二对象的出现率与大小分布（好会话 598KB /
   0ba366 零字节两种形态的占比）；②context/spotlight 每播放重发率（信息关的
   量化输入）；
8. 跑便宜独立门：API 通道 / 封面通道 vs 视频通道的冒充者错认一致率（<10%）。

**P2 表示层（P1 结论出来再定）**
9. 若下载组成确定：混合突发整体当指纹（包序列线主吃；45ms 段特征在 0.4s
   突发里只有 2 段，段指纹成员基本失业）；
10. 若组成漂移：起点鲁棒表示（裁头增强）或 time_bins 链路归一化剖面。

---

## 七、更正记录（11-17，按时间序）

初版文档基于自研解密器（snap_decrypt_flow.py）+ 初步 tshark 分析，同日经
tshark 全量复核修正三处：

1. **"请求头全在 0-RTT、early secret 缺失"→ 证伪**。所有携带请求 URL 的帧
   `quic.long.packet_type` 为空 = 1-RTT 短头包，用 keylog 现有的
   CLIENT_TRAFFIC_SECRET_0 即可解；cf-st/bolt 连接的全部 GET 均已解出。真相是
   自研解密器客户端方向的包号重建有 bug，把"自家解不出"误判为"钥匙不在"；
2. **"bolt 连接零密钥、内容无法判明"→ 证伪**。00c18f95（原断言依据的会话）
   frame 360 即解出 `GET …/network_mapping.bin`；10 会话 bolt GET 全部可解，
   恒定部分定性为平台常数（25,345B 同 SHA-256）。自研解密器的 keylog 匹配
   逻辑（按 ClientHello random 对行）有缺陷；
3. **"/context/spotlight = deeplink 解析器"→ 修正定位（后又按三形态再校准）**。
   初判依据是好会话：媒体 GET 先于两条 context（视频 URL 来自本地缓存元数据），
   故"是播放后的记账接口"。补充半程/零媒体会话后校准为：它与媒体 GET 的先后
   不固定（0ba366 里紧贴媒体前 0.09s），角色随缓存态/会话路径而变；且回显有
   两种形态（token / 14 位数字 ID）。稳定的结论只剩一条——**token 回显的归因
   价值**（§2.2 结论 1-3、§2.3）。

---

## 附：工具、产物与已验证命令

| 工具 | 用途 |
|---|---|
| scripts/analyze_snap_pilot.py | pcap 解析、QUIC v1 Initial SNI 解密、DNS 映射、按主机聚合（--dump 导出） |
| scripts/snap_burst_diag.py | 秒级时间轴、突发形态、包大小分布 |
| scripts/snap_pktsize_audit.py | 包大小直方图（>200KB 流审计、IPv6 排查） |
| **tshark（Wireshark ≥ 4.x 自带）** | **解密基准工具**：TCP/TLS 与 QUIC/HTTP3 通吃，URL/响应头/etag 全出（URL 真值以此为准） |
| scripts/snap_h3_reassemble.py | tshark 输出二次处理：HTTP/2+HTTP/3 按 (UDP/TCP 流, stream_id) 两遍法重组响应体，token/objectID grep |
| scripts/snap_dump_bodies.py | 把全部解密 DATA 字节按 L4 流落盘 + 模式搜索（§2.4 零命中证据的来源） |
| scripts/snap_decrypt_flow.py | 自研全链路解密（**两处 bug 待修**，修前仅作交叉核对） |
| scripts/snap_stream_timeline.py | 锚点对齐（含 +8h 修正）、流交织度、黑盒突发纯度、100ms 时间轴 |

中间产物（flows.json、逐帧时间戳、解密字节落盘等）与全部脚本位于源研究工作区，未随本文入库。

已验证的 tshark 命令（Git Bash，`$PCAP`/`$KEY` 分别为 pcap 与 sslkey 路径）：

```bash
# 1) 全部 HTTP 请求 URL（含时间，HTTP/2+HTTP/3 都覆盖）
tshark -r "$PCAP" -o "tls.keylog_file:$KEY" \
  -Y "http3.request.full_uri" -T fields \
  -e frame.number -e frame.time_relative -e http3.request.full_uri

# 2) 某条媒体流的响应头（etag/content-range；stream_id 与 udp.stream 按需改）
tshark -r "$PCAP" -o "tls.keylog_file:$KEY" \
  -Y "http3 && quic.stream.stream_id == 0 && udp.stream == 21" -T fields \
  -e frame.number -e _ws.col.Info        # 先找到 "HEADERS: 206" 的帧号 N
tshark -r "$PCAP" -o "tls.keylog_file:$KEY" \
  -Y "frame.number == N" -V | grep -iE "content-range|etag|content-type"
```

依赖：tshark 即 Wireshark 命令行（≥4.x）；自研解密器另依赖 aioquic 1.8.0（pylsqpack）。
