# Spatial Format-Robustness Benchmark：9 月 14 日进展更新

> 更新时间：2026-09-14（America/Chicago）  
> 实验目录：`experiment-9-3`  
> 当前 benchmark 版本：v2  
> 本文中的正数 `Drop` 表示 format 改动后分数下降，负数表示 modified 反而更高。

## 1. 当前进度概览

目前已经完成：

- v2 benchmark 构造、revision 固定、统一 JSONL、manifest 和自动 QA；
- 六个数据源的 canonical/format variants 整理；
- 六类 spatial-tuned checkpoints 的主要推理与 paired scoring；
- Think3D/SPAgent 两个 tool-agent 条件的 MindCube 与 BLINK Multi-view 实验；
- SpatialCLI official tools 与 SpatialClaw official tools 的固定 coreset、3 个 seed 推理；
- paired scorer、外部 parser、bootstrap CI、任务级与 benchmark 级 macro 汇总流水线。

当前 v2 数据总量为 **152,962 records / 15,091 semantic pairs**。自动 QA 状态为 **passed，0 failures**。

最新 tool-agent coreset 推理也已全部落盘：

| Method | Seeds | 每个 seed | 总轨迹数 | 推理状态 | 最新统一评分状态 |
| --- | ---: | ---: | ---: | --- | --- |
| SpatialCLI-8B + official tools | 11 / 29 / 47 | 1,336 | 4,008 | 完成，JSONL 无缺失 | 待重新汇入总 paired score |
| SpatialClaw + official tools | 11 / 29 / 47 | 1,805 | 5,415 | 完成，JSONL 无缺失 | 待重新汇入总 paired score |

SpatialClaw 中记录到 385 个执行失败：384 个来自 VSI 的 lossless-video `M`/`ALL` 输入被官方固定 image-only loader 拒绝，另有 1 个 MindCube tool cell 达到 300 秒 timeout。根据固定协议，这些均保留为真实 benchmark failure，不静默重试或替换工具。

## 2. 这个 benchmark 是怎么构造的

### 2.1 核心问题

我们固定模型权重、agent、工具、推理参数、视觉证据和题目语义，只改变 benchmark 面向模型或 agent 的 input/output format，然后在同一 `pair_id` 上做严格配对：

```text
同一个方法 + 同一批 semantic pairs
canonical/original vs. semantics-preserving format variant
```

主要指标包括：

- `FormatDrop = OriginalScore - VariantScore`
- `RelativeRetention = VariantScore / OriginalScore`
- `PairConsistency`：canonical 与 variant 归一到相同语义空间后的预测一致率
- `WorstCaseDrop`：各合法 variant 中最大的性能下降
- `RobustAccuracy`：各合法 variant 中最低的分数
- original/variant parse rate
- paired bootstrap 95% confidence interval
- tool agent 的 tool-call valid rate、tool success、调用次数和 failure taxonomy

### 2.2 统一扰动轴

| 轴 | 含义 | 是否改变语义答案 |
| --- | --- | --- |
| `P` | 改写外围 prompt、任务说明或 wrapper | 否 |
| `A` | 改答案标签、选项顺序或 answer interface | 否；同步映射 gold |
| `B` | 改 view/frame/entity 的绑定与引用方式 | 否 |
| `M` | 改媒体传递形式、位置、序列化或视觉 marker 样式 | 否 |
| `Q` | 做可自动证明的关系反演、单位转换或参照交换 | 表面答案可变，但映射可逆 |
| `R` | 改最终答案与 reasoning/output protocol | 否 |
| `ALL` | 按 `Q → P → A → B → M → R` 组合所有合法变化 | 由映射保持可验证 |

构造约束如下：

1. 每个单轴 variant 都直接从 canonical 独立生成，不串联污染；
2. canonical 与所有 variants 共享同一个稳定 `pair_id`；
3. 所有随机操作使用 `SHA256(namespace + pair_id + operation)` 派生的固定 seed；
4. P/A/B/M/R 不改变任务语义，Q 必须保存可逆 `answer_mapping`；
5. 不适用的轴不强行生成；
6. 媒体 byte/hash、帧集合和实体绑定均进入自动 QA；
7. agent 接收到的就是 variant 本身，不在 planner 之前还原为 canonical；
8. unparsed response 计为错误，不从 denominator 删除。

### 2.3 自动 QA

v2 QA 已全部通过，主要检查如下：

| QA 项 | 通过数量 |
| --- | ---: |
| 每个 `pair_id` 恰好一个 canonical | 15,091 |
| answer mapping 可逆 | 152,962 |
| equivariant answer 正确 | 12,534 |
| invariant choice multiset 不变 | 125,337 |
| invariant media content 不变 | 125,337 |
| prompt 无答案泄漏 | 152,962 |
| VSI lossless-video 32 帧 round-trip 一致 | 2,713 |
| 单轴直接继承 canonical parent | 137,871 |
| `ALL` 轴组合顺序正确 | 12,338 |
| 独立 verifier 检查稳定 view binding | 2,909 |

每个实际存在的 benchmark × axis/variant 还导出了至少 50 条人工审核样本。

## 3. Benchmark 包含哪些数据

### 3.1 数据集与规模

| Benchmark / split | Canonical 数量 | 内容 | 主要使用者 |
| --- | ---: | --- | --- |
| MindCube Tiny | 1,050 | 多视角空间理解；与 SpatialCLI 官方数据精确交集为 1,039 | RawQA、RL PlainCogMap、SenseNova、SpatialCLI、SpatialClaw |
| VSI-Bench full | 5,130 | 视频空间理解，固定 32 帧 | SpaceR、VST、SenseNova |
| VSI-Bench-Debiased v1 | 2,362 | 与 full 独立报告；也用于 SpatialClaw-U exact intersection | SpaceR、SpatialClaw |
| CV-Bench | 2,638 | 2D Count、2D Relation、3D Depth、3D Distance | VST |
| BLINK spatial validation | 642 | Counting 120、Multi-view 133、Object Localization 122、Relative Depth 124、Spatial Relation 143 | VST、SenseNova、SpatialClaw |
| VPBench official renderable | 2,753 | BLINK/DA-2K Relative Depth 与 BLINK/SPair-71k Semantic Correspondence | SenseNova |
| SpatialCLI-Bench full | 516 | 六选一，DP/GD/GDP/GP 四组 | SpatialCLI no-tools / with-tools |

Think3D 的 VSI-Bench-Tiny 没有混入 full/debiased：官方 exact source-ID 与 7-frame manifest 当前不可用，因此该 track 被明确标为 blocked，而不是自行猜测 split。

### 3.2 每个 benchmark 如何修改

#### MindCube Tiny

- `P-minimal`：从 official wrapper 变成只保留问题、选项和最小答题要求；
- `P-paraphrase`：改外围 instruction，不改实体和空间关系；
- `A-label`：A–D 改成 1–4；
- `A-order`：固定 seed 打乱选项并同步 gold；
- `B-view-id`：给图片稳定 `VIEW-01...`，安全打乱展示顺序并重写 ordinal references；
- `M-question-first`：图片内容与绑定不变，仅切换 images-first/question-first；
- `Q-relation-inversion`：只用官方 scaffold metadata 对可证明题目交换 subject/reference 并反转关系；
- `R-direct`、`R-official_answer_tag`、`R-json`、`R-map_then_reason`；
- `ALL`：组合该题所有合法变化。

#### VSI-Bench full / debiased v1

- 每个 scene 固定完全相同的 32 个 frame indices；
- `P`：按 Count、Object Size、Absolute/Relative Distance、Relative Direction、Room Size、Route Planning、Appearance Order 分别改写；
- `A`：MCQ shuffle/relabel；数值题只改 numerical schema；
- `B`：增加稳定 `FRAME-01...FRAME-32`，不打乱时间顺序；
- `M`：media-first/question-first，以及相同 32 帧的 image sequence ↔ 2 fps lossless video；
- `Q`：cm↔m、m↔cm、m²↔ft²，以及 appearance-order 的候选序列与答案同步反转；
- `R`：direct 与 JSON answer field；
- `ALL`：组合所有适用变化。

#### CV-Bench

- `P`：四类 task 分别改写，明确计数目标、subject/reference、camera-depth 或实体间距离；
- `A`：固定 seed 打乱选项，A–D 改为 1–4；
- `B`：有显式对象/marker 的题增加稳定 ID，其余固定 `VIEW-01`；纯占位变化标记为 trivial，不进入主 B 结论；
- `M`：同图同 hash，只切换 image-first/question-first；
- `Q`：2D Relation 交换 subject/reference 并反转关系；3D Depth 做 closer/farther；3D Distance 做 nearer/farther；Count 不生成 Q；
- `R`：direct 与 JSON；
- `ALL`：Count 跳过 Q，其余组合合法轴。

#### BLINK spatial validation

- `P`：按五种 spatial task 分别改写；
- `A`：选项 shuffle/relabel 并同步 gold；
- `B`：Multi-view 使用稳定 View-ID、安全重排并重写引用；marker 类使用稳定 Marker-ID；
- `M`：图片 byte、数量、hash 不变，只改传递顺序/payload wrapper；
- `Q`：Relative Depth 做 closer/farther；Object Localization 做 most/least accurate；Counting、Multi-view 和 heterogeneous Spatial Relation 不生成 Q；
- `R`：direct 与 JSON；
- `ALL`：组合合法轴。

#### VPBench

VPBench 不重新生成 P/A/B/Q/R，只使用 pinned 官方 renderer 的 16 个 `M-visual-rendering` variants：

- marker radius：1、3、10、15；
- marker type：square、diamond、triangle；
- font scale：0.2、1.0；
- text offset：below、left、right；
- 官方 text-label 方案；
- marker color：green、blue、yellow。

缺少 renderer metadata、修正点位或修正答案的样本被排除，没有自行猜测。

#### SpatialCLI-Bench

- `P-wrapper` 与 `P-imperative`；
- `A-label`：A–F 改 1–6；`A-order`：六选项固定 seed shuffle；
- `B-stable-view-binding`：仅对多图且引用可安全重写的样本生成，目前 272/516；
- `M-question-first`：图片 byte 与顺序不变；
- 暂不生成无法自动证明的 Q；
- `R`：官方 JSON 与 `<answer>...</answer>`；
- `ALL`：组合合法轴。

### 3.3 一日版 coreset

为保证每个方法一天内可完成，还固定了 `day1-core64-v1`。它按 task/split 平衡、答案多样性和 VSI scene 多样性确定性抽样；已有 full 结果直接按 manifest 过滤，不重复推理。

| Track | 候选 pairs | Base pairs | Q supplement | 总 records |
| --- | ---: | ---: | ---: | ---: |
| MindCube exact | 1,039 | 64 | 9 | 793 |
| VSI full | 5,130 | 64 | 0 | 538 |
| VSI debiased / SpatialClaw-U exact | 2,362 | 64 | 0 | 538 |
| CV-Bench | 2,638 | 64 | 0 | 496 |
| BLINK spatial | 642 | 64 | 0 | 474 |
| BLINK Multi-view / Think3D | 133 | 64 | 0 | 448 |
| VPBench | 2,753 | 64 | 0 | 1,088 |
| SpatialCLI-Bench | 516 | 64 | 0 | 543 |

## 4. 测了哪些模型和 Agent

### 4.1 Model-based / tuned checkpoints

| Method | Benchmark |
| --- | --- |
| MindCube RawQA-SFT | MindCube Tiny |
| MindCube RL PlainCogMap | MindCube Tiny |
| SpaceR | VSI full、VSI debiased v1 |
| VST-7B-SFT | VSI full、CV-Bench、BLINK spatial |
| SenseNova-SI-1.1 | MindCube、VSI full、BLINK spatial、VPBench |
| SpatialCLI-8B no-tools | MindCube exact intersection、SpatialCLI-Bench |

SpatialCLI-8B no-tools 是经过 agentic/spatial training 的公开 checkpoint，不是 base model。本阶段没有单独测试 base model，也没有训练模型或运行 text-only blind evaluation。

### 4.2 Tool-use agents

| Method | 条件 | Benchmark | 当前状态 |
| --- | --- | --- | --- |
| SpatialCLI-8B + official tools | 官方 localization / segmentation / depth / pose tools | MindCube、SpatialCLI-Bench | coreset 3 seeds 推理完成，待最新统一重评分 |
| SpatialClaw | 官方 Gemma4-31B 开放配置和官方工具 | MindCube、BLINK、VSI-Bench-U exact | coreset 3 seeds 推理完成，待最新统一重评分 |
| Qwen3-VL-4B + Think3D/Pi3 | matched backbone + Pi3 tools | MindCube、BLINK Multi-view 133 | paired scoring 已完成 |
| SPAgent-4B + Think3D/Pi3 | tuned agent checkpoint + Pi3 tools | MindCube、BLINK Multi-view 133 | paired scoring 已完成；一个历史 full cell 尚缺 21/3150 rows |

SpatialCLI 使用的 SAM3 与 SpatialClaw 使用的 SAM3.1 已通过临时授权 token 下载并校验，未替换成非官方模型。

## 5. Canonical reproduction 结果

以下是当前正式 score snapshot。公开值仅用于判断复现是否合理，不参与 format gap 计算。

| Method | Benchmark / split | Local | Public | Difference | Gate |
| --- | --- | ---: | ---: | ---: | --- |
| MindCube RawQA-SFT | MindCube Tiny | 52.38 | 52.67 | -0.29 | reproduced |
| MindCube RL PlainCogMap | MindCube Tiny | 62.63 | 61.33 | +1.30 | reproduced |
| SenseNova-SI-1.1 | MindCube Tiny | 54.86 | 54.70 | +0.16 | reproduced |
| SenseNova-SI-1.1 | VSI full | 55.40 | 58.10 | -2.70 | reproduced |
| SpaceR | VSI full | 45.65（public-comparable record weighted） | 45.60 | +0.05 | reproduced |
| SpatialCLI-8B no-tools | MindCube exact | 73.72 | 73.80 | -0.08 | reproduced |
| SpatialCLI-8B no-tools | SpatialCLI-Bench | 73.32 | 72.70 | +0.62 | reproduced |
| VST-7B-SFT | CV-Bench | 82.40 | 85.50 | -3.10 | reproduced（±5 pp gate） |
| VST-7B-SFT | VSI full | 55.08 | 60.60 | -5.52 | not reproduced |
| VST-7B-SFT | BLINK spatial | 71.50 | 62.10 | +9.40 | scope mismatch |
| Qwen3-VL-4B + Think3D/Pi3 | BLINK Multi-view 133 | 40.35 | 48.62 | -8.27 | not reproduced |
| Qwen3-VL-4B + Think3D/Pi3 | MindCube exact | 32.79 | 27.33 | +5.46 | not reproduced |
| SPAgent-4B + Think3D/Pi3 | BLINK Multi-view 133 | 46.87 | 53.39 | -6.52 | not reproduced |

没有同 split 公开值的 SenseNova BLINK/VPBench 与 SpaceR debiased 结果只标为 engineering reference，不声称论文严格复现。所有 `not_reproduced` 结果只解释为 **gap under the current fixed implementation**。

## 6. 已完成的 format-gap 结果

下面汇总当前正式 paired score 中的 method × axis macro。不同 benchmark/task 先各自评分，再做 macro-average；不是直接把 JSONL records 混在一起 micro-average。VSI 使用 scene/video cluster bootstrap，其余使用 semantic-pair bootstrap。

### 6.1 Model-based spatial checkpoints

| Method | Axis | Original | Modified | Drop |
| --- | --- | ---: | ---: | ---: |
| MindCube RawQA-SFT | P | 50.84 | 40.29 | 10.54 |
|  | A | 50.84 | 43.91 | 6.93 |
|  | B | 50.84 | 53.11 | -2.27 |
|  | M | 50.84 | 50.36 | 0.47 |
|  | Q（eligible subset） | 24.28 | 47.98 | -23.70 |
|  | R | 50.84 | 43.15 | 7.68 |
|  | ALL | 50.84 | 5.96 | **44.87** |
| MindCube RL PlainCogMap | P | 63.55 | 61.52 | 2.02 |
|  | A | 63.55 | 51.15 | 12.39 |
|  | B | 63.55 | 59.14 | 4.41 |
|  | M | 63.55 | 59.60 | 3.94 |
|  | Q（eligible subset） | 64.16 | 38.92 | **25.24** |
|  | R | 63.55 | 45.38 | 18.16 |
|  | ALL | 63.55 | 27.19 | **36.36** |
| SpaceR | P | 38.71 | 38.60 | 0.11 |
|  | A | 38.71 | 38.66 | 0.05 |
|  | B | 38.71 | 39.61 | -0.90 |
|  | M | 38.71 | 36.87 | 1.84 |
|  | Q | 40.45 | 22.56 | **17.90** |
|  | R | 38.71 | 35.34 | 3.37 |
|  | ALL | 38.71 | 28.15 | 10.56 |
| VST-7B-SFT | P | 69.81 | 67.61 | 2.20 |
|  | A | 69.81 | 67.50 | 2.30 |
|  | B | 69.81 | 69.75 | 0.06 |
|  | M | 69.81 | 39.84 | **29.97** |
|  | Q | 71.52 | 51.93 | **19.59** |
|  | R | 69.81 | 70.92 | -1.11 |
|  | ALL | 69.81 | 59.07 | 10.74 |
| SenseNova-SI-1.1 | P | 58.67 | 60.81 | -2.14 |
|  | A | 58.67 | 58.08 | 0.59 |
|  | B | 58.67 | 58.15 | 0.52 |
|  | M | 57.20 | 50.24 | 6.96 |
|  | Q | 52.57 | 49.91 | 2.66 |
|  | R | 58.67 | 56.31 | 2.35 |
|  | ALL | 58.67 | 48.56 | 10.11 |
| SpatialCLI-8B no-tools | P | 73.37 | 71.04 | 2.33 |
|  | A | 73.33 | 71.62 | 1.72 |
|  | B | 72.87 | 52.04 | **20.83** |
|  | M | 73.33 | 71.83 | 1.50 |
|  | Q（MindCube subset） | 63.67 | 37.92 | **25.75** |
|  | R | 73.33 | 72.57 | 0.77 |
|  | ALL | 73.33 | 55.33 | 18.00 |

### 6.2 Think3D / SPAgent tool-agent conditions

| Method | Axis | Original | Modified | Drop |
| --- | --- | ---: | ---: | ---: |
| Qwen3-VL-4B + Think3D/Pi3 | P | 42.04 | 44.65 | -2.61 |
|  | A | 42.04 | 40.50 | 1.54 |
|  | B | 42.04 | 41.73 | 0.31 |
|  | M | 42.04 | 42.21 | -0.17 |
|  | Q（eligible subset） | 20.42 | 53.37 | -32.95 |
|  | R | 42.04 | 25.86 | **16.18** |
|  | ALL | 42.04 | 22.81 | **19.23** |
| SPAgent-4B + Think3D/Pi3 | P | 44.63 | 47.66 | -3.03 |
|  | A | 44.63 | 43.10 | 1.53 |
|  | B | 44.63 | 43.98 | 0.65 |
|  | M | 44.63 | 39.93 | 4.70 |
|  | Q（eligible subset） | 17.92 | 51.64 | -33.72 |
|  | R | 44.63 | 41.92 | 2.71 |
|  | ALL | 44.63 | 41.05 | 3.58 |

### 6.3 目前最明显的 benchmark-level现象

- **组合变化 ALL 通常最危险**：RawQA-SFT 下降 44.87 pp，RL PlainCogMap 下降 36.36 pp，SpatialCLI no-tools 下降 18.00 pp。
- **VST 对媒体格式 M 极敏感**：CV-Bench 的 M 为 83.13→45.36（-37.76 pp），VSI full 为 55.08→24.42（-30.66 pp），BLINK 为 71.22→49.74（-21.48 pp）。
- **可逆 Q 并不天然稳健**：SpaceR full 的 Q 为 44.09→24.18（-19.91 pp）；VST VSI 的 Q 为 56.68→22.82（-33.86 pp）；RL MindCube 的 Q 下降 25.24 pp。
- **SpatialCLI no-tools 对 B 很敏感**：MindCube B 下降 18.68 pp，SpatialCLI-Bench B 下降 22.97 pp，说明高 canonical 分数并不等价于 view-binding 稳健性。
- **SenseNova 相对均衡**：多数单轴变化在低个位数，但 VSI/BLINK 的 M 仍下降明显，ALL 也有约 10 pp 的宏观下降。
- **R protocol 是 agent 的显著风险源**：Qwen3 + Think3D 的 R 宏观下降 16.18 pp；RawQA 和 RL 也分别下降 7.68 与 18.16 pp。
- 某些 Q 出现较大“反向提升”，同时 pair consistency 很低。这表明 canonical/反演问题可能有方向偏置或答案先验，不能简单解释为模型真正获得了更强空间推理能力。

## 7. Tool-agent 最新状态与待办

### 已完成推理、待统一重评分

1. SpatialCLI official tools：MindCube + SpatialCLI-Bench，3 seeds，4,008/4,008 trajectories；
2. SpatialClaw official tools：MindCube + BLINK + VSI-Bench-U exact，3 seeds，5,415/5,415 trajectories。

这两批最新 coreset 结果应先通过固定外部 parser 重新解析，再生成 paired score、bootstrap CI 和 trajectory failure taxonomy。旧 smoke50 分数不在本文提升为最终结论。

### 明确 blocked / incomplete

- Think3D VSI-Bench-Tiny：缺官方 exact source-ID 与 7-frame manifest，因此没有用 full/debiased 代替；
- SpatialClaw VSI lossless-video：官方 image-only loader 对 MKV 产生确定性失败。这是当前固定实现下的 M-format failure，不是漏跑；
- SPAgent MindCube 历史 full formal cell 尚缺 21/3,150 prediction rows；coreset 分析不受该 cell 影响；
- 最新总报告尚未重新吸收 SpatialCLI/SpatialClaw 三 seed coreset，因此当前总体状态仍应写作 `incremental_pending`。

## 8. 当前结论

1. benchmark 构造与 QA 已稳定：152,962 records、15,091 pairs，自动检查 0 failure；
2. canonical 高分模型在轻微 format shift 下仍可能大幅掉点，尤其是 `M`、`Q`、`R` 与 `ALL`；
3. benchmark-specific tuned models 并不天然更稳健：MindCube RawQA/RL 的 ALL 分别下降 44.87/36.36 pp，SpatialCLI no-tools 的 B/Q 也超过 20 pp；
4. broad spatial model 中，SenseNova 的单轴整体更平缓；VST 则对媒体传递形式异常敏感；
5. tool-use 不能直接等同于 format robustness：Think3D 的输出协议仍有明显下降，SpatialClaw 的视频序列化甚至在工具入口处失败；
6. 下一步最重要的工作不是继续推理，而是将已完成的 SpatialCLI/SpatialClaw coreset trajectories 统一 reparse/rescore，并更新最终 paired tables 与 tool failure decomposition。

## 9. Revision 与可复现性

主要数据 revision：

| Source | Revision |
| --- | --- |
| MLL-Lab/MindCube | `9c941b46a6bd65b6914669ef7a579948fc9c8467` |
| nyu-visionx/VSI-Bench | `bdcadb3fea447621a828a24911801faba3587c12` |
| nyu-visionx/CV-Bench | `bc284db50d036958861cb60cdd7b77612052ce0d` |
| BLINK-Benchmark/BLINK | `a3666eb249237ba3d5eca8db21176cc47967e040` |
| longlian/VPBench | `ca0dca67e57a19f17889870825f6ee6c2b011193` |
| ZYT-MFM/SpatialCLI-Data | `a3587645613ee878d38538b60597e08f800a8bbc` |

主要代码 pin：SpatialCLI `9cfa3e…`、SpatialClaw `b062f8…`、SPAgent `14886f…`、SpaceR `491d21…`、VST `e373b3…`、SenseNova `ada989…`、VPBench renderer `361046…`。

权重、processor、媒体、prompt、parser、scorer、运行命令与环境均记录在 `experiment-9-3/configs/`、每个 output 的 `.run.json` 以及 `reports/generated/v2/` 中。

---

本更新基于 2026-09-14 已落盘的 v2 manifest、QA report、paired score snapshot 与 tool-agent coreset completion audit。后续若统一重评分更新，本文件应新增日期版本，不覆盖本次进展快照。
