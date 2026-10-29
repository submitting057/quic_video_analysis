**开集识别（OSR）经典研究与 YouTube VID 识别技术路线调研**

检索与核验：2026-10-29 至 2026-10-30。目标是建立清晰的发展脉络，筛选可迁移的技术路线，并保存作者代码入口。重点覆盖 2013—2022 年经典工作；这是一份定向文献调研，不是截至 2026 年的最新方法排行榜。

用户任务：输入加密视频流量，正确识别训练中已有的 YouTube 视频 ID（VID），把未见 VID 拒识为 unknown。研究不限于网络领域；复现环境与部署实现暂不展开。

**核心建议：以强闭集分类器建立参照，优先研究特征空间与接收边界，再考察伪未知和重建。** 具体顺序是：同一模型的 MSP/MLS/Energy → CAC/ARPL 的特征塑形 → OpenMax/EVM 的统计边界 → 有条件引入 PROSER、背景样本或重建。这个顺序是针对项目的研究建议，不代表论文之间存在普遍性能排序。

**1. 先明确任务与技术边界**

设训练 VID 集合为 K。测试样本属于 K 时，应输出正确 VID；不属于 K 时，应输出 unknown。基础 OSR 不要求把未知 VID 互相区分，也不要求立即学会新 VID。这与进一步给未知类命名、聚类或增量学习的任务有区别。[OSR 定义与语义未知评测](https://www.robots.ox.ac.uk/~vgg/research/osr/)

对本项目，一个实用分解是：流量编码器产生表示，分类头预测已知 VID，拒识规则决定是否接收这个预测。方法的主要区别在于：改变拒识分数、改变表示几何、拟合接收区域，还是引入代理未知训练。

尤其需要区分两件事：

- **类别未知**：测试 VID 不在 K 中，目标是拒识。
- **已知类别的条件变化**：VID 仍在 K 中，但采集条件发生变化，目标仍是接收且正确分类。

“已知视频在新网络条件下的流量”不能仅因为分布变了就被视为新 VID。这个任务分解借鉴语义变化与其他分布变化的区分；是否存在码率、设备、播放位置等具体因素，应以项目数据为准。[SSB 研究](https://arxiv.org/abs/2110.06207)

**2. 发展脉络：从风险约束到表示学习，再回到强基线**

| 时期 | 核心问题 | 代表工作 | 对后续研究的作用 |
|---|---|---|---|
| 2013—2014 | 分类边界为什么会接收离训练样本很远的未知输入？ | 1-vs-Set、W-SVM/CAP | 形式化开放空间风险，研究接收区域与概率衰减 |
| 2016—2018 | 如何让深度分类器输出“都不属于”？ | OpenMax、DOC、EVM | 尾部校准、逐类接收阈值、局部类别包含模型 |
| 2018—2021 | 仅用已知类学习的表示，如何更有利于拒识？ | OSRCI、CROSR、RPL/ARPL、CAC、PROSER | 形成伪未知、重建、特征塑形等并行分支 |
| 2018 起的背景样本分支 | 若能获得部分非目标类别，怎样利用？ | Entropic Open-Set / Objectosphere | 用背景数据约束响应与特征，需要明确额外数据条件 |
| 2022 | 复杂开集机制的收益是否来自更好的训练和评测？ | MLS 强基线、SSB | 要求与充分训练的闭集分类器比较，并关注难未知类别 |

这是机制发展的归纳，不是后发表的方法必然优于先发表的方法。依据为下面逐项列出的原始论文。

**3. 经典论文与作者代码矩阵**

“作者代码公开”与“明确开源许可”分别记录；代码可访问不等于已运行复现。技术摘要来自论文，代码状态来自作者页面或仓库。

| 研究与发表信息 | 机制与阅读重点 | 作者代码及核验结果 |
|---|---|---|
| Scheirer et al., **1-vs-Set**，TPAMI 2013；[Toward Open Set Recognition](https://doi.org/10.1109/TPAMI.2012.256) | 形式化 open space risk，研究如何约束已知类接受区域。用于理解为什么闭集分类边界不足以处理未知。 | [tboult/libSVM-onevset](https://github.com/tboult/libSVM-onevset)，作者主页链接，BSD-3-Clause；传统特征与 SVM 实现。 |
| Scheirer et al., **W-SVM / CAP**，TPAMI 2014；[Probability Models for Open Set Recognition](https://doi.org/10.1109/TPAMI.2014.2321392) | 概率随远离已知数据而衰减，结合极值理论（EVT）与 SVM。重点是有约束的接收区域。 | [ljain2/libsvm-openset](https://github.com/ljain2/libsvm-openset)，含 W-SVM、PI-SVM；有历史依赖许可文本，标为作者公开代码。 |
| Bendale & Boult，**OpenMax**，CVPR 2016；[Towards Open Set Deep Networks](https://openaccess.thecvf.com/content_cvpr_2016/html/Bendale_Towards_Open_Set_CVPR_2016_paper.html) | 对正确分类训练样本到类均值激活向量的距离尾部拟合，再将部分响应重新分配给 unknown。深度 OSR 的经典起点。 | [abhijitbendale/OSDN](https://github.com/abhijitbendale/OSDN)，官方；顶层 BSD-3-Clause；原始实现依赖旧 Caffe/libMR。 |
| Shu, Xu & Liu，**DOC**，EMNLP 2017；[Deep Open Classification of Text Documents](https://aclanthology.org/D17-1314/) | 用逐类 sigmoid 替代共享 softmax，再以高斯拟合设置类别阈值。提供来自 NLP 的跨模态借鉴：每个已知类分别决定是否接收。 | [leishu02/EMNLP2017_DOC](https://github.com/leishu02/EMNLP2017_DOC)，MIT；作者说明公开示例未使用论文的预训练词向量，不应把默认脚本视为完整原结果。 |
| Rudd et al., **EVM**，TPAMI 2018，预印本 2015；[The Extreme Value Machine](https://arxiv.org/abs/1506.06112) | 从样本与异类样本的 margin distance 建模局部类别包含概率，可由多个局部区域表示一个类。与 OpenMax 的类均值尾部校准不同。 | [EMRResearch/ExtremeValueMachine](https://github.com/EMRResearch/ExtremeValueMachine)，由[作者项目页](https://www.wjscheirer.com/projects/openset-recognition/)链接；README 说明相对原文有小改动，未见独立标准 LICENSE，记为作者公开实现。 |
| Neal et al., **OSRCI**，ECCV 2018；[Open Set Learning with Counterfactual Images](https://openaccess.thecvf.com/content_ECCV_2018/html/Lawrence_Neal_Open_Set_Learning_ECCV_2018_paper.html) | 生成接近已知样本、又不属于已知类别的反事实图像，以代理 unknown 辅助训练。代表显式生成伪未知路线。 | [lwneal/counterfactual-open-set](https://github.com/lwneal/counterfactual-open-set)，作者实现；本次未发现明确 LICENSE，记为代码公开。 |
| Dhamija, Günther & Boult，**Objectosphere / Entropic Open-Set**，NeurIPS 2018；[Reducing Network Agnostophobia](https://papers.neurips.cc/paper/8129-reducing-network-agnostophobia.pdf) | 利用背景样本训练，使其输出更不确定，并约束特征幅度；同时提出 OSCR 评测。需要区分训练背景样本与从未接触的测试未知类。 | [Vastlab/Reducing-Network-Agnostophobia](https://github.com/Vastlab/Reducing-Network-Agnostophobia)，作者仓库；明确提供论文 MNIST 实验，不能据此声称覆盖全部实验。 |
| Yoshihashi et al., **CROSR**，CVPR 2019；[Classification-Reconstruction Learning for Open-Set Recognition](https://d-itlab.c.titech.ac.jp/~reikawa/research/crosr/index.html) | 联合分类与重建，把重建所需的潜表示用于未知检测。重点是保留分类损失可能丢弃的信息，不宜简化成只对重建误差设阈值。 | [作者代码 ZIP](https://d-itlab.c.titech.ac.jp/~reikawa/research/crosr/crosr-release.zip)；下载入口已核实，内部环境/许可未完成核验。[saketd403/CROSR](https://github.com/saketd403/CROSR) 是第三方实现。 |
| Chen et al., **RPL**，ECCV 2020；[Learning Open Set Network with Discriminative Reciprocal Points](https://www.ecva.net/papers/eccv_2020/papers_ECCV/papers/123480511.pdf) | reciprocal points 表示对应已知类的类外空间，通过多类交互约束特征空间。它们不是普通类别中心。 | 后续作者 [ARPL 仓库](https://github.com/gary23ai/ARPL)包含 RPLoss；应说明使用后续代码中的 RPL 实现。 |
| Chen et al., **ARPL**，TPAMI，2021 在线发表；[Adversarial Reciprocal Points Learning for Open Set Recognition](https://arxiv.org/abs/2103.00953) | 在 reciprocal points 路线上进一步约束开放空间；可加入 confusing samples。ARPL 与 ARPL+CS 应分开研究。 | [gary23ai/ARPL](https://github.com/gary23ai/ARPL)，官方 MIT；原 iCGY96 地址重定向至此。作者仓库同时出现 TPAMI 2021/2022 标记，本文按 2021 在线版本记年。 |
| Miller et al., **CAC**，WACV 2021；[Class Anchor Clustering: A Loss for Distance-Based Open Set Recognition](https://openaccess.thecvf.com/content/WACV2021/papers/Miller_Class_Anchor_Clustering_A_Loss_for_Distance-Based_Open_Set_Recognition_WACV_2021_paper.pdf) | 在 logit 空间把已知类样本聚集到固定类锚点附近，直接塑造适合距离拒识的结构。 | [dimitymiller/cac-openset](https://github.com/dimitymiller/cac-openset)，官方 BSD-3-Clause；README 警告旧权重与当前 trial 划分不匹配。 |
| Zhou, Ye & Zhan，**PROSER**，CVPR 2021；[Learning Placeholders for Open-Set Recognition](https://openaccess.thecvf.com/content/CVPR2021/papers/Zhou_Learning_Placeholders_for_Open-Set_Recognition_CVPR_2021_paper.pdf) | 同时学习数据 placeholder 和分类器 placeholder；以 manifold mixup 产生代理未知，为未知保留决策空间。 | [LAMDA-CL/CVPR21-Proser](https://github.com/LAMDA-CL/CVPR21-Proser)，作者代码；README 具体提供 CIFAR-10 未知检测实验，未见明确 LICENSE。 |
| Vaze et al., **MLS / SSB**，ICLR 2022；[Open-Set Recognition: a Good Closed-Set Classifier is All You Need?](https://arxiv.org/abs/2110.06207) | 在其评测条件下，充分训练的闭集分类器配合最大 logit 已具竞争力；SSB 更重视语义未知的难度。结论不是“OSR 已解决”。 | [sgvaze/osr_closed_set_all_you_need](https://github.com/sgvaze/osr_closed_set_all_you_need)，官方 MIT；提供划分、训练与评估代码，以及 TinyImageNet 权重入口。 |

补读 **C2AE**：Oza & Patel，CVPR 2019，[Class Conditioned Auto-Encoder for Open-Set Recognition](https://openaccess.thecvf.com/content_CVPR_2019/html/Oza_C2AE_Class_Conditioned_Auto-Encoder_for_Open-Set_Recognition_CVPR_2019_paper.html)。它通过类别条件重建和误差建模拒识，适合与 CROSR 对照理解。原文给出的作者仓库 `otkupjnoz/c2ae` 本次网页与 API 访问均返回 404，故不列为当前可用官方代码入口；这不意味着作者从未公开代码。

关于旧依赖：OpenMax/W-SVM 仓库保留的历史 libMR 文本与新版不同；独立 [Vastlab/libMR](https://github.com/Vastlab/libMR) 已说明采用 BSD-3-Clause 替代旧许可。以后复现应记录实际依赖版本。

**4. 六条技术路线分别解决什么问题**

| 路线 | 实际改变什么 | 能学到什么 | 核心局限与待检验问题 |
|---|---|---|---|
| 强闭集模型＋拒识分数 | 在同一分类器上使用 MSP、MLS、Energy 等分数 | 分离“表示质量”和“评分函数”的贡献 | 已知样本分类正确不保证未知拒识；结论依赖训练条件 |
| 统计校准／接收区域 | OpenMax 拟合类均值距离尾部；EVM 建立局部包含模型 | 用已知数据的统计结构决定接受范围 | 表示和校准分布改变后，已知样本也可能落入拒识区 |
| 特征几何与边界学习 | CAC 的类锚点、RPL/ARPL 的类外空间约束 | 在训练时使已知与潜在未知更易分离 | 单中心是否合适、边界是否过紧，需要数据检验 |
| 伪未知训练 | OSRCI 生成样本，PROSER 构造特征与分类器 placeholder | 学习代理边界，而非仅观察已知类内部 | 代理未知不等于未来真实未知；生成方法引入自身偏差 |
| 分类＋重建 | CROSR 保留重建潜变量；C2AE 利用条件重建误差 | 补充纯判别训练忽略的信息 | 重建的信息是否与类别身份有关，不能事先保证 |
| 辅助背景样本 | Objectosphere 等显式利用非目标训练数据 | 从有限背景样本中学习低响应/拒识行为 | 额外数据改变监督条件，仍须测试未见背景类别 |

各行的机制依据见上一节对应论文；局限是用于项目选型的分析，不是对这些论文在 VID 上性能的实验证明。

作为评分对照，设分类器原始输出为 z：MSP 为 `max softmax(z)`；MLS 为 `max z`。Energy 原始形式为 `E(x) = -T log Σ exp(z_k/T)`，若统一使用“分数越高越倾向已知”，可使用 `-E(x)`，并明确温度 T。后处理 Energy 与使用额外异常数据微调的 Energy 不是相同训练设定。[MSP 原作者代码](https://github.com/hendrycks/error-detection)、[Energy 原论文与代码](https://github.com/weitliu/energy_ood)

**5. 针对 VID 项目的技术取舍**

以下内容全部是把通用 OSR 机制迁移到项目的研究假设。未访问项目数据，也未验证哪种方法最优。

**优先路线 A：可靠的 VID 表示＋简单评分对照。** 先在相同编码器与训练配置下比较 MSP、MLS、后处理 Energy，观察已知 VID 准确率与未知拒识的关系。如果已知 VID 换采集条件就发生严重混淆，应先改进表示或训练数据覆盖。对一个缺少身份信息的表示，仅改变拒识分数可能不足以恢复 VID 区分能力。强闭集基线论文支持这种对照思路，但不能直接外推其图像结果到流量。[MLS/SSB](https://www.robots.ox.ac.uk/~vgg/research/osr/)

**优先路线 B：CAC 与 ARPL，比较两种特征塑形思路。** 重点观察类内是否紧凑、类间是否分离、未知落在何处，以及跨条件已知样本是否被挤出接受范围。若同一 VID 在不同播放位置或采集条件下形成多个稳定簇，可以进一步检验“单中心与多个局部区域”的差异。这里指同一类别的多簇分布，不是音频/视频等多模态输入。多个原型只是待检验扩展，不能当成已经证明的改进。[CAC](https://arxiv.org/abs/2004.02434)、[ARPL](https://arxiv.org/abs/2103.00953)

**优先路线 C：OpenMax / EVM，比较统计边界。** 在相同冻结表示上研究这类方法，有助于判断收益来自编码器还是接受区域建模。OpenMax 适合研究类条件尾部校准；EVM 值得用于检验一个 VID 是否需要多个局部接收区域。需同步检查未知误接收与跨条件已知误拒，EVT 本身不提供对采集条件变化的不变性。[OpenMax](https://openaccess.thecvf.com/content_cvpr_2016/html/Bendale_Towards_Open_Set_CVPR_2016_paper.html)、[EVM](https://arxiv.org/abs/1506.06112)

**第二阶段：PROSER、背景 VID 与重建。** 特征层伪未知可能比生成完整流量更便于研究，但两个 VID 的特征混合不保证是真实未知，也可能落入某个已知 VID 的合理变化范围。若有额外背景 VID，可单列 Objectosphere 一类方案，保证背景 VID 与最终测试未知 VID 不重合。重建分支值得作为对照：它可能保留有用身份细节，也可能更容易拟合传输条件；需通过消融辨别。[PROSER](https://arxiv.org/abs/2103.15086)、[Objectosphere](https://arxiv.org/abs/1811.04110)、[CROSR](https://d-itlab.c.titech.ac.jp/~reikawa/research/crosr/index.html)

不应直接用“视频主题相似”定义流量中的难未知。两个内容相似的视频是否具有相近加密流量表示，需要观察证据。难度分组可以基于预先规定的流量特征或独立验证分析，不能根据最终测试结果挑选有利样本。

**6. 能支撑项目结论的评估方案**

建议先建立如下四格。它是本项目的实验设计建议，借鉴 SSB 对类别未知的强调和 OpenOOD 对分布偏移、独立验证的处理。[SSB](https://arxiv.org/abs/2110.06207)、[OpenOOD v1.5](https://arxiv.org/html/2306.09301v5)

| 测试输入 | 与训练相近的采集条件 | 预先定义的变化条件 |
|---|---|---|
| 已知 VID | 接收并正确分类，衡量基础识别能力 | 仍应接收并正确分类，衡量身份表示与阈值稳健性 |
| 未知 VID | 拒识，检验类别未知，而非采集来源差异 | 拒识，检验类别未知与条件变化并存的情况 |

划分同时考虑 **VID 身份与独立播放/采集会话**：已知 VID 可以跨训练和测试出现，但会话必须独立；最终未知 VID 不进入训练或模型选择。若同一会话切成多个重叠窗口，应整组划分，避免相邻片段跨集合泄漏。已知与未知测试样本尽量匹配采集条件，以免“识别采集批次”替代“识别未知 VID”。

推荐共同报告：

| 指标 | 回答的问题 | 解释注意 |
|---|---|---|
| 已知 VID 闭集 accuracy／宏平均 recall | 不考虑拒识时，VID 能否分对？ | 类别不均衡时不要只报总体 accuracy |
| AUROC | 已知与未知分数能否排序分离？ | 不计已知 VID 是否分类正确，不能单独作为 OSR 质量 |
| OSCR 曲线及其面积 | 在不同未知误接收率下，多少已知样本被正确识别并接收？ | 与仅进行已知/未知二分类的指标互补 |
| 固定阈值下的已知正确接收率、未知误接收率 | 实际工作点是否满足项目需求？ | 同时报告阈值来源与每个测试条件的表现 |

OSCR 的 CCR 分子为“已知样本中分类正确且被接收的数量”，分母为全部已知测试样本；对应横轴 FPR 为“未知样本被接收的比例”。因此一个只会拒识、不能正确区分 VID 的系统不会得到理想的 OSCR。[OSCR 原始论文](https://papers.neurips.cc/paper/8129-reducing-network-agnostophobia.pdf)

对于常被简称 FPR95 的指标，建议本项目明确写 **FPR@95TPR-ID：已知样本接收率为 95% 时的未知误接收率**。一些工具采用相反方向，即检出 95% 未知时已知样本被误拒的比例。OpenOOD v1.5 的相关实现采用未知为正类；两种数值不能直接比较。[OpenOOD 指标实现](https://github.com/Jingkang50/OpenOOD/blob/main/openood/evaluators/metrics.py)

还要区分测试 ROC 上的统计量与可用阈值：部署阈值由独立验证集确定，随后冻结，在不同采集条件上报告实际已知接收率与未知误接收率。不要在每个测试条件重新调阈值到 95%，否则会掩盖阈值随条件变化失效的问题。

为防止“复杂方法只因训练更好而获胜”，比较时固定或报告编码器、预训练数据、输入长度、增强、训练预算、模型选择规则、类别划分及随机性。ARPL 与 ARPL+CS 分开；背景数据方法与仅已知数据方法分开。按多个 VID 划分和随机种子报告结果及波动，不以单次最优结果定路线。

**7. 阅读顺序与可研究问题**

建议的主阅读顺序：

1. **MLS/SSB（2022）**：先建立强基线与公平评估意识。
2. **Toward Open Set Recognition（2013）**：理解开放空间风险。
3. **OpenMax（2016）＋EVM（2018）**：理解两种统计接收边界。
4. **CAC（2021）＋RPL/ARPL（2020/2021）**：比较类内结构与类外空间约束。
5. **PROSER（2021）**：理解怎样构造代理未知以及其假设。

第二轮按项目需要补充 DOC 的类别阈值、Objectosphere 的背景监督、OSRCI 的生成路线，以及 CROSR/C2AE 的重建路线。W-SVM/CAP 用于补齐理论沿革。

以下问题适合进一步实证研究，但本次调研不构成新颖性证明：

- **条件变化下的接受区域**：同一 VID 的跨条件变化与未知 VID 接近已知区域，能否同时处理？
- **单中心与多个局部区域**：同一 VID 的真实分布是否需要多个局部模型？增加模型容量是否只是过拟合会话？
- **代理未知的有效性**：特征混合/背景 VID 带来的收益，是否能泛化到完全未见的 VID？
- **阈值的迁移稳定性**：在独立验证条件上选定的阈值，换采集条件后能否维持已知接收与未知拒识的平衡？

这些问题比直接把某个图像 OSR 模块接到流量模型后只报一个 AUROC，更有利于判断技术是否真正支持项目。

**8. 后续复现入口与已核实的限制**

| 用途 | 首选入口 | 已确认的限制 |
|---|---|---|
| 强基线、经典划分、SSB | [MLS/SSB 作者仓库](https://github.com/sgvaze/osr_closed_set_all_you_need) | 包含作者本地路径；训练 shell 脚本中 `dataset` 与 `DATASET` 大小写不一致，直接执行前需适配。仅做静态检查，未运行。 |
| 专门 OSR 损失及变体 | [ARPL 作者仓库](https://github.com/gary23ai/ARPL) | 区分 RPL、ARPL、ARPL+CS；不同仓库的默认训练配置不等价。 |
| 类锚点与距离拒识 | [CAC 作者仓库](https://github.com/dimitymiller/cac-openset) | 核对 trial 与权重，仓库已明确警告旧权重问题。 |
| 数据／分类器 placeholder | [PROSER 作者仓库](https://github.com/LAMDA-CL/CVPR21-Proser) | 公开示例的实验覆盖范围有限；未发现明确许可证。 |
| OOD 后处理与评估组件 | [OpenOOD](https://github.com/Jingkang50/OpenOOD) | MIT；可借用组件，但默认 OOD 数据协议不等于 VID OSR 协议，也不等于经典 CIFAR-10 类别留出协议。 |

为方便未来锁定版本，本次记录四个仓库的 HEAD：

| 仓库 | 提交 SHA |
|---|---|
| sgvaze/osr_closed_set_all_you_need | `3258f058d4fd75cb88e23d0bd638d24b629e60d3` |
| gary23ai/ARPL | `3ede8b38e1cfb9d70e106cc19d563453110c36ab` |
| dimitymiller/cac-openset | `bd9c7f5d0518c2e6b346933e468c344135619b31` |
| LAMDA-CL/CVPR21-Proser | `f5b53b90509e6460a783a0baa72bda52364810fd` |

上述只是代码入口和静态核验记录，不承诺当前环境可直接运行或原论文指标可直接复现。

**9. 检索方法、证据与局限**

检索使用论文完整标题、方法名与 `open set recognition / official code / GitHub` 等组合，并从论文和作者主页追溯代码。论文优先核对 CVF、ECVA、ACL Anthology、NeurIPS、作者项目主页和作者预印本；仓库优先核对 README、文件结构、许可与可用地址。核心纳入条件是：定义了重要问题或代表不同机制的同行评议研究，并能核实论文及代码来源；C2AE 作为代码失效的文献补充单列。没有依据 GitHub star 数做学术排序，也没有拼接不同协议的性能数字。

研究证据以同行评议的原始算法研究为主；README/许可只支持代码状态、接口和使用条件，不作为算法效果证据。没有将计算机科学实验套入临床研究证据等级。

局限包括：这不是穷尽性系统综述；没有覆盖所有经典分支或 2023—2026 年新方法；部分站点访问受限时使用作者预印本和项目页交叉核验；未执行模型训练，未逐个验证权重和全部数据下载；CROSR 官方 ZIP 内部内容尚未核实。对 VID 的适用性均为研究推断，需要独立数据实验验证。

本报告使用 AI 辅助检索、来源核验与综合整理。正式论文引用应回到各原始论文；本报告适合作为研究导航和实验设计依据。
