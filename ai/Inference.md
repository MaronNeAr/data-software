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

- 每一个 Transformer 层的 Attention 模块，首先把输入矩阵 X 分别投影为 Q、K、V 三个矩阵。这是 Prefill 中 FLOP 消耗最大的操作。

  <img src="https://p.ipic.vip/4hfzsd.png" alt="image-20260526145952837" style="zoom:50%;" />

- Prefill 时 S 可以是几千，三次 GEMM 形状为 [S×d]×[d×d]，是真正的矩阵乘矩阵（GEMM），GPU 算力利用率高。Decode 时 S=1，退化为向量乘矩阵（GEMV），GPU 大量空转。

###### 第三步：Flash Attention——重新定义 Attention 计算

- 原始 Attention 需要把完整的 QK^T 矩阵（大小 S×S）写到 HBM，再读回来做 Softmax，内存带宽成瓶颈。Flash Attention 通过分块计算（tiling）完全在 SRAM 内完成，彻底规避 HBM 往返。

<img src="https://p.ipic.vip/as8gop.png" alt="image-20260526150053785" style="zoom:50%;" />

###### 第四步：KV Cache 写入——Prefill 唯一的"遗产"

- Prefill 阶段的核心产出就是 KV Cache。每个 Transformer 层，每个 token 的 K 向量和 V 向量被持久化到 GPU HBM，供后续所有 Decode step 反复读取。

  <img src="/Users/marlon1475/Library/Application Support/typora-user-images/image-20260526153430613.png" alt="image-20260526153430613" style="zoom:50%;" />

- KV Cache 是显存的最大消耗者。FP8 量化 KV Cache 可以将其减半，但需要额外的反量化开销。前缀缓存（Prefix Caching）让相同 system prompt 的请求直接复用已有 KV Block，避免重复 Prefill。

###### 第五步：FFN、LayerNorm 与残差连接

- Attention 子层之后，每个 Transformer 层还有 FFN（前馈网络）、两个 LayerNorm 和两条残差路径。这些操作在 Prefill 中同样是并行处理所有 S 个 token。

  <img src="/Users/marlon1475/Library/Application Support/typora-user-images/image-20260526154921798.png" alt="image-20260526154921798" style="zoom:50%;" />

- GQA（Grouped Query Attention）通过让多个 Query head 共享一组 K/V head，把 KV Cache 减小 8× 以上，同时基本不损失精度。Llama-3 / Mistral 等都采用了 GQA。

###### 第六步：FLOP、显存与时延的全局量化

- 把前面所有步骤汇总，看 Prefill 阶段的算力消耗、显存占用和时延的完整数学图景。这决定了 TTFT 能做到多低。

  <img src="/Users/marlon1475/Library/Application Support/typora-user-images/image-20260526154959397.png" alt="image-20260526154959397" style="zoom:50%;" />

- TTFT 优化核心路径：
  - ① FP8/INT8 量化（降低 FLOP 和显存带宽）
  - ② 张量并行 TP（多卡分摊计算）
  - ③ 前缀缓存（命中时跳过整个 Prefill）
  - ④ Flash Attention v3（提升 MFU 到 75%+）
  - ⑤ 算子融合（减少 kernel launch overhead）

#### Decode阶段



#### Flash Attention
