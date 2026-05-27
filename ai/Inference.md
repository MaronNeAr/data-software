## 第一部分 Inference Infrastructure

#### 整体架构概览

<img src="https://p.ipic.vip/j534x2.png" alt="image-20260526133338251" style="zoom:50%;" />

#### 推理与训练的根本差异

| 维度     | 训练                     | 推理                            |
| -------- | ------------------------ | ------------------------------- |
| 优化目标 | 最大化 MFU（算力利用率） | 最小化 TTFT + TBT，控制成本     |
| 批次大小 | 越大越好（数千）         | 动态变化，需要在线调度          |
| 显存使用 | 激活值+梯度+优化器状态   | 模型权重 + KV Cache（主要瓶颈） |
| 并发     | 固定进程数               | 数千并发请求                    |
| 故障处理 | checkpoint 恢复即可      | 请求失败直接影响用户体验        |

- 推理的计算特点是**内存带宽受限**，不是算力受限。每生成一个 token，需要把所有模型权重从 HBM 读进来做一次矩阵向量乘法，GPU 的 TFLOPS 大量闲置，瓶颈在 HBM 带宽（~3 TB/s）。这个认知决定了推理优化的整个方向。

#### 推理的两个阶段：Prefill 与 Decode

<img src="https://p.ipic.vip/ujb49r.png" alt="image-20260526133356353" style="zoom:50%;" />

- Prefill 阶段是算力密集型，批量处理输入 prompt，GPU 利用率高；Decode 阶段是内存带宽密集型，每次只读权重、生成一个 token，GPU 大量空转。这个不对称性催生了 **Prefill/Decode 分离（PD 分离）**架构—把两个阶段放到不同的机器上跑，各自优化。

#### KV Cache 管理：推理显存的核心战场

<img src="https://p.ipic.vip/zf8wsy.png" alt="image-20260526133417983" style="zoom:50%;" />

- **PagedAttention**（vLLM 提出）是推理 Infra 的最重要创新之一。传统方案为每个请求预分配连续显存，导致严重的内存碎片化和浪费。PagedAttention 借鉴操作系统的虚拟内存分页思想，把 KV Cache 切成固定大小的 block，用 Block Table 做逻辑→物理映射，显存利用率从约 20-40% 提升到 90%+。

- 在此基础上，**前缀缓存（Prefix Caching）** 进一步优化：相同 system prompt 的请求共享 KV Block，命中时直接跳过 Prefill，TTFT 可以降低 50-90%。

#### 连续批处理：让 GPU 持续满载

<img src="https://p.ipic.vip/oenvjx.png" alt="image-20260526133435863" style="zoom:50%;" />

- 传统静态批处理（Static Batching）需要等批次中最长的请求完成，短请求完成后 GPU 空转。**连续批处理（Continuous Batching）**在每个 decode step 粒度动态插入新请求，一旦有请求完成就立刻填入新的，GPU 几乎不空转。这是吞吐量提升的最重要工程手段，现在是所有主流推理引擎（vLLM、TRT-LLM、SGLang）的默认策略。

#### 模型优化技术

- **量化（Quantization）**是最直接的优化，将 FP16/BF16 权重压缩到更低精度：

  - `INT8`（W8A8）：权重和激活都量化到 8 位，几乎无精度损失，推理速度提升约 1.5-2×

  - `FP8`（H100 原生支持）：兼顾精度与速度，目前最主流的量化方案

  - `INT4`（GPTQ / AWQ）：激进量化，模型体积降至 FP16 的 1/4，速度提升 3-4×，有一定精度损失

  - `KV Cache 量化`：KV Cache 也可以量化到 INT8/FP8，显存占用降低 50%

- **投机解码（Speculative Decoding）**是另一个重要技巧：用一个小的草稿模型（draft model）先快速生成几个 token，再用大模型并行验证，批量接受或拒绝。由于验证是并行的，如果草稿模型猜对率高，相当于每次 decode step 生成了多个 token，TTFT 不变但 TBT 显著降低，吞吐提升 2-3×。

- **算子融合与图优化**：将 LayerNorm + QKV 投影、Attention + Softmax、FFN 的多个 CUDA kernel 融合成一个，减少内存往返和 kernel launch 开销。Flash Attention 是这一方向的标志性工作，把 Attention 的 HBM 访问从 O(n²) 降到 O(n)，对长序列效果尤为显著。

#### 推理系统的部署架构

<img src="https://p.ipic.vip/yqusls.png" alt="image-20260526133456220" style="zoom:50%;" />

- **PD 分离（Prefill-Decode 分离）**是当前大规模推理的前沿架构，由 Splitwise 等论文提出，已被多家大厂采用。Prefill 和 Decode 计算特性完全不同，混部会互相干扰——Prefill 的大算力需求会抢占 Decode 的显存，Decode 的低 GPU 利用率又拖累 Prefill 的资源效率。分离后，各自选用更匹配的硬件（Prefill 用算力强的 H100，Decode 用显存大的 A100），调度策略独立优化。

- **张量并行（Tensor Parallelism）**在推理中也广泛使用，将模型权重按列/行切分到多卡，降低单卡显存需求，也可以通过多卡并行降低延迟。推理的 TP 度一般是 2、4、8（节点内），因为跨节点通信延迟会拖慢 TBT。

#### 关键指标与 SLO

- `TTFT`（Time to First Token）：从发出请求到收到第一个 token 的延迟，主要由 Prefill 决定。用户感知最直接。
- `TBT`（Time Between Tokens）也叫 `ITL`（Inter-Token Latency）：每两个 token 之间的间隔，决定了流式输出的"打字速度"，由 Decode 决定。
- `吞吐量`（Throughput）：系统每秒总共能生成多少 token，是成本的反面。
- `$/token`：推理的终极商业指标，= 硬件成本 / 吞吐量。

#### 主流推理引擎对比

| 引擎         | 核心特点                                          | 适用场景             |
| ------------ | ------------------------------------------------- | -------------------- |
| vLLM         | PagedAttention、连续批处理的开创者，生态最全      | 通用部署，研究首选   |
| TensorRT-LLM | NVIDIA 出品，极致 kernel 优化，FP8 原生，性能最强 | 生产环境 H100 集群   |
| SGLang       | 激进前缀缓存、RadixAttention，复杂 agent 场景极佳 | 多轮对话、RAG、agent |
| MLC-LLM      | 编译器驱动，跨硬件（GPU/CPU/移动端）              | 边缘/移动端部署      |
| Ollama       | 极简本地部署，CPU+GPU 混合                        | 个人开发者本地使用   |

#### 推理 Infra 的前沿方向

- **长上下文推理**：100K+ token 的 KV Cache 单卡放不下，需要跨卡甚至跨机器的分布式 KV Cache，以及 Attention 的分块计算（Ring Attention、Sequence Parallelism）。

- **MoE 推理**：专家混合模型（如 Mixtral、DeepSeek）每次只激活少量专家，理论算力低，但专家路由引入了 All-to-All 通信，且不同专家负载不均衡，是专门的调度难题。

- **多模态推理**：图像/视频的 Vision Encoder 输出大量 token，与语言模型的推理调度需要协同，异构计算（GPU + 专用编解码器）管理复杂。

- **Edge 推理**：在手机、PC 等算力受限设备上运行小模型，需要 INT4 量化、CPU 推理、内存 offload 等一系列技术的组合。

## 第一部分 推理引擎

#### Prefill阶段

###### 概览：Prefill 在整个推理中的位置

- Prefill 是推理的第一个阶段。用户发来一条 prompt，引擎需要"消化"这整段输入，为后续每一步 Decode 奠定基础。它决定了 TTFT（Time to First Token）—— 用户等待第一个字出现的时间。

  <img src="https://p.ipic.vip/7gge8d.png" alt="image-20260526143818247" style="zoom:50%;" />

- Prefill 是 算力密集型（compute-bound）操作——GPU 的 Tensor Core 满载运行矩阵乘法。与 Decode 的内存带宽瓶颈截然相反。

###### 第一步：Tokenization → Embedding 查表

- 输入文字首先经过分词器（BPE / SentencePiece）切成 token ID 序列，再通过 Embedding 矩阵映射成稠密向量。这是 Prefill 的入口。

  <img src="https://p.ipic.vip/yw0nom.png" alt="image-20260526145134079" style="zoom:50%;" />

###### 第二步：QKV 投影——最重的矩阵乘法

- Q（Query）：查询词，“我现在是谁？”
- K（Key）：索引词，“我是谁，我有什么特征，谁可以来找我？”
- V（Value）：内容/资产，“我代表的真正含义和具体价值是什么？”
- 每一个 Transformer 层的 Attention 模块，首先把输入矩阵 X 分别投影为 Q、K、V 三个矩阵。这是 Prefill 中 FLOP 消耗最大的操作。

<img src="https://p.ipic.vip/4hfzsd.png" alt="image-20260526145952837" style="zoom:50%;" />

- Prefill 时 S 可以是几千，三次 GEMM 形状为 [S×d]×[d×d]，是真正的矩阵乘矩阵（GEMM），GPU 算力利用率高。Decode 时 S=1，退化为向量乘矩阵（GEMV），GPU 大量空转。

###### 第三步：Flash Attention——重新定义 Attention 计算

- 原始 Attention 需要把完整的 QK^T 矩阵（大小 S×S）写到 HBM，再读回来做 Softmax，内存带宽成瓶颈。
- 因为 Softmax 算子是一个“全局非线性算子”。为了算出某一个元素的 Softmax 概率，它必须知道整一行的所有元素加起来的总和。
- Flash Attention 通过分块计算（tiling）完全在 SRAM 内完成，彻底规避 HBM 往返。

<img src="https://p.ipic.vip/3hoj04.png" alt="image-20260526150053785" style="zoom:50%;" />

###### 第四步：KV Cache 写入——Prefill 唯一的"遗产"

- Prefill 阶段的核心产出就是 KV Cache。每个 Transformer 层，每个 token 的 K 向量和 V 向量被持久化到 GPU HBM，供后续所有 Decode step 反复读取。

  <img src="https://p.ipic.vip/unbtp1.png" alt="image-20260526153430613" style="zoom:50%;" />

- Prefix Caching：它利用基数树（Radix Tree）和虚拟页表将共享的提示词前缀锁定在显存中，让后续并发请求直接复用已算好的 KV Cache，从而彻底免单昂贵的 Prefill 计算、秒开首字。

- KV Cache 是显存的最大消耗者。FP8 量化 KV Cache 可以将其减半，但需要额外的反量化开销。前缀缓存（Prefix Caching）让相同 system prompt 的请求直接复用已有 KV Block，避免重复 Prefill。

###### 第五步：FFN、LayerNorm 与残差连接

- **Attention（像分布式通信网络）：** 负责在水平方向上“连线”。它告诉系统当前这个词和句子里的其他词有什么关系。它的计算是跨词（Cross-token）的。
- **FFN（像本地超级处理器/只读知识库）：** 负责在垂直方向上“纵深挖掘”。在 Attention 把周围词的信息召回并融合进来之后，FFN 独自拿着这个融合后的向量，疯狂进行高维度的非线性变换。它的计算是完全独立（Token-wise）的，词与词之间互不干扰。
- **MoE专家模式**：MoE 的做法就是保留 Attention 不动，把后面这个笨重的 FFN 拆成 16 个或者 64 个微型的 FFN（专家）。
- **LayerNorm**：它是一种归一化机制，不改变数据的几何形状，只负责在每一层计算前后，把每一个词向量里的所有维度强行拉回到“均值为 0、方差为 1”的正态分布轨道上，防止数据在几十层连续的矩阵乘法中活活“爆炸”或“枯竭”
- Attention 子层之后，每个 Transformer 层还有 FFN（前馈网络）、两个 LayerNorm 和两条残差路径。这些操作在 Prefill 中同样是并行处理所有 S 个 token。

<img src="https://p.ipic.vip/jngf5q.png" alt="image-20260526154921798" style="zoom:50%;" />

- GQA（Grouped Query Attention）通过让多个 Query head 共享一组 K/V head，把 KV Cache 减小 8× 以上，同时基本不损失精度。Llama-3 / Mistral 等都采用了 GQA。

###### 第六步：FLOP、显存与时延的全局量化

- 把前面所有步骤汇总，看 Prefill 阶段的算力消耗、显存占用和时延的完整数学图景。这决定了 TTFT 能做到多低。

  <img src="https://p.ipic.vip/f7s730.png" alt="image-20260526154959397" style="zoom:50%;" />

- TTFT 优化核心路径：
  - ① FP8/INT8 量化（降低 FLOP 和显存带宽）
  - ② 张量并行 TP（多卡分摊计算）
  - ③ 前缀缓存（命中时跳过整个 Prefill）
  - ④ Flash Attention v3（提升 MFU 到 75%+）
  - ⑤ 算子融合（减少 kernel launch overhead）

#### Decode阶段

###### Decode 阶段的核心机制

- **每一步的输入只有一个新 token**（上一步刚生成的），但 Attention 需要看到所有历史 token 的 Key/Value——这些历史信息正是 KV Cache 存储的内容。
- 它是一个**严格的自回归循环**，每次只生成一个 token，反复执行直到遇到终止符。

###### 第一步  单Token输入

- **GEMM**：通用矩阵乘法
- **GEMV**：通用矩阵-向量乘法

- Prefill 结束后，第一个生成的 token（例如 id=3251）成为 Decode 第一步的唯一输入。与 Prefill 的 [S, d] 输入不同，Decode 每步输入形状为 [1, d]——只有一行。这个差异决定了后续所有计算的性质：从矩阵×矩阵（GEMM）退化为矩阵×向量（GEMV）。

<img src="https://p.ipic.vip/7k88va.png" alt="image-20260527135838419" style="zoom:50%;" />

###### 第二步 QKV投影

- 新 token 向量 x_t [1,d] 与 QKV 权重矩阵相乘，得到当前步的 q、k、v 向量。注意：这里是向量×矩阵，而非矩阵×矩阵。GPU 的 Tensor Core 专为大矩阵乘法设计，对 GEMV 的利用率极低——这正是 Decode 阶段 GPU 利用率仅 10-30% 的核心原因。

- GEMV 的算术强度（FLOPs / 字节）极低：每次运算需要把整个权重矩阵从 HBM 读入，但只做一次向量点积。H100 的算力约是内存带宽的 200 倍，GEMV 只能喂饱带宽，算力大量闲置。增大 batch size 是提升算术强度、让 GEMV 向 GEMM 退化的唯一途径。

  <img src="https://p.ipic.vip/8tifuo.png" alt="image-20260527141437422" style="zoom:50%;" />

###### 第三步 KV Cache 追加

- 为了预测第 $t$ 个 Token，它的 Attention 算子在数学上必须拿到从第 $1$ 个到第 $t-1$ 个 Token 所有的 $K$ 和 $V$ 信息。
- 将当前步新计算的 k_t、v_t 追加写入 KV Cache 对应位置。写入后，KV Cache 中历史长度从 t 变为 t+1。这个操作本身很轻量（只写一行），但随着序列增长，KV Cache 占据的显存线性增加，最终可能成为并发上限的决定因素。
- KV Cache 的显存占用随序列长度线性增长：每生成一个 token，每层多存一对 K/V 向量。对于 70B 模型，每 1000 token 的 KV Cache 约 8GB（FP16）。PagedAttention 将 KV Cache 组织成固定大小的 block，当请求生成完毕时立刻释放 block 供新请求使用，大幅提高显存利用率。

<img src="https://p.ipic.vip/4cc4eg.png" alt="image-20260527144319667" style="zoom:50%;" />

###### 第四步 Masked self-attention（KV Cache 读取）

- **$Q_t$向量**：当前的 $Q_t$ 向量，就是大模型在 Decode（吐字）阶段，为了预测“下一个字”，由当前步刚刚输入的那“唯一一个词”在当前层通过矩阵投影，衍生出来的、用来打捞全部历史记忆的“雷达探针”。

- **核心过程**：拿着当前步那孤零零的 $Q_t$ 向量，去显存里全量读取历史攒下来的 KV Cache 矩阵，通过高频搬运和精细的掩码控制，算完当前的 Attention 结果。
- 这是 Decode 与 Prefill 差异最大的地方。当前 token 的 q_t [1,d] 需要与 KV Cache 中所有 t+1 个历史 K 向量做点积，得到注意力分数，再与历史 V 向量加权求和。注意：计算量随序列增长是线性的（O(t·d)），但每步都要把整个 KV Cache 从 HBM 读出——这是 decode 最重的内存读取操作。
- 随着 t 增大，KV Cache 读取量线性增长：每生成一个 token，需要多读一行 K 和一行 V。长序列（t=8192）时，KV Cache 读取是单步最大的内存开销，远大于权重读取。Multi-Query Attention（MQA）和 GQA 通过减少 K/V head 数量，直接缩小 KV Cache 尺寸，降低这里的带宽压力。

<img src="https://p.ipic.vip/bgfiv9.png" alt="image-20260527150553509" style="zoom:50%;" />

###### 第五步 FNN前馈网络

- Attention 输出经过 Add&Norm 后进入 FFN（SwiGLU/GeLU）。与 QKV 投影一样，这里也是 GEMV：[1,d] 与 [d, 4d] 的权重矩阵相乘。FFN 权重是模型最大的参数块（约占总参数 2/3），每步 decode 都要把这部分权重完整地从 HBM 读入，这是 Decode 阶段单步最大的内存读取量。

- FFN GEMV 的内存读取量：3 × d × 4d × 2 bytes（FP16）。对 7B 模型（d=4096, 4d=11008），单层 FFN 约需读取 ~270MB 权重，32 层合计 ~8.6GB。这是 decode 速度的硬下限：HBM 带宽 3.35TB/s ÷ 8.6GB ≈ 390步/秒（单请求理论上限），实际因其他开销更低。

  <img src="https://p.ipic.vip/asrxkl.png" alt="image-20260527164517463" style="zoom:50%;" />

###### 第六步 采样策略

<img src="https://p.ipic.vip/q3a5zc.png" alt="image-20260527164720534" style="zoom:50%;" />

###### 第七步 终止条件检测

<img src="https://p.ipic.vip/a4c55y.png" alt="image-20260527164823761" style="zoom:50%;" />

###### 第八步 Decode性能全景与优化

<img src="https://p.ipic.vip/txhxlu.png" alt="image-20260527164917153" style="zoom:50%;" />

###### Decode 的本质：内存带宽瓶颈

- **它是内存带宽受限（memory-bandwidth bound），而非算力受限**。每生成一个 token，GPU 必须把全部模型权重从 HBM 读入一遍，而只做极少量的乘法——这就像一个超级工厂，原料运输是瓶颈，机器却大量闲置。

  <img src="https://p.ipic.vip/6v6rg7.png" alt="image-20260527165125213" style="zoom:50%;" />

- 可以切换不同模型大小、精度和 GPU 来观察吞吐变化。在"瓶颈点"（knee）之前，增大 batch size 是提升吞吐最直接的方式；之后受算力封顶趋于平缓。

###### 投机解码：突破自回归串行约束

- 投机解码（Speculative Decoding）是目前降低 Decode TBT 最有效的方法，它通过引入一个小的草稿模型来绕过自回归的串行约束。

- **核心思想**：小模型生成 k 个候选不需要很准，只要接受率足够高，大模型一次验证就等效于多步输出。接受率 α 越高（草稿质量越好），加速比越大。常用草稿来源包括独立的小模型、同一模型的浅层（Medusa 多头）、或 n-gram 缓存（ngram speculation）。

  <img src="https://p.ipic.vip/qlchql.png" alt="image-20260527165242397" style="zoom:50%;" />

###### Decode 的完整循环视图

<img src="https://p.ipic.vip/cn1s2i.png" alt="image-20260527171934961" style="zoom:50%;" />

###### Decode 优化技术总结

- **增大有效 batch size（摊薄带宽）**：连续批处理（Continuous Batching）+ PagedAttention 是最基础的组合拳。前者保证 GPU 随时满载不空转，后者通过消除显存碎片让尽可能多的请求同时驻留在显存，两者共同最大化 active batch size，从而让 GEMV 尽量接近 GEMM 的效率。
- **减少每步需读取的权重字节数（降低带宽压力）**：量化是这里的核心手段。INT8 将每个参数从 2 字节压到 1 字节，权重读取量减半，TBT 理论上减半。INT4（GPTQ/AWQ）进一步压到 0.5 字节，但需要仔细的校准来控制精度损失。FP8 是 H100 原生支持的最优方案：精度损失极小，速度接近 INT8。
- **让一次大模型前向产出更多 token（突破串行上限）**：投机解码（Speculative Decoding）通过草稿模型预测 k 个候选，大模型并行验证，实现 2-4× 的有效 TBT 改善。Medusa 是变体方案，在同一模型末尾加多个并行头，无需单独草稿模型。
- **减少 KV Cache 体积（让更多请求同时驻留）**：GQA（Grouped Query Attention，LLaMA-3 / Qwen2 标配）把 K/V head 数量从 H 降到 H/G，KV Cache 体积降低 G 倍。KV Cache 量化（INT8/FP8）进一步减半。前缀缓存（Prefix Caching）让共享 system prompt 的请求复用同一份 KV Cache，从根本上减少 KV 存储需求。
