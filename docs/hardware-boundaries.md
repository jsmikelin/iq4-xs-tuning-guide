# 硬件瓶颈调查（串行/PCIe/新方案）

> 为什么 40-42 t/s 是 8GB + 消费级 CPU 的天花板？全查实证据。

## 瓶颈本质

**CPU-GPU 层间串行**：ncmoe 27 把 27 层 MoE 专家放 CPU 计算，13 层 + attention 放 GPU。CPU 算专家时 GPU 空闲，GPU 算 attention 时 CPU 空闲——**无跨设备流水线并行**。

实测证据：
- 推理时 CPU 86% 忙碌 vs GPU util 仅 34-44%
- ncmoe 27→25 少算 2/40 层但速度不变 → CPU 计算量不是瓶颈，**串行同步/固定开销才是**

## PCIe 侧（已最优，无提升空间）

| 检查项 | 状态 |
|---|---|
| Resizable BAR | ✅ 已启用 (BAR1 = 8192MiB 全量映射) |
| PCIe gen4 x8 | ✅ 推理时满速 16GB/s，实际需求仅 2.3GB/s |
| NVLink/多卡 | ❌ 笔记本无 |

## 串行侧（唯一解药 = 异步 CPU-GPU overlap）

| 方案 | 结论 |
|---|---|
| **ktransformers** (kvcache-ai) | ❌ 硬门槛：24GB VRAM + AVX512/AMX + Linux + 800GB RAM，面向 400B+ 超大 MoE (教程 Qwen3.5-400B 需 4×4090) |
| **APEX** (arxiv 2506.03296) | ❌ 研究论文，无生产实现 (T4 上 +84-96% 的理论) |
| **llama.cpp 主线** | ❌ b10375→b10451 无异步调度提交 |
| **i9-14900HX AVX512/AMX** | ❌ Raptor Lake 消费级禁用 (仅 Sapphire Rapids 服务器有) |

## 连续批处理（唯一实证有效的方法论）🏆

**APEX 论文思想 → llama.cpp 连续批处理天然实现部分 overlap**：

| 场景 | 单请求 | 2 请求并发组合吞吐 | 提升 |
|:--|:--|:--|:--|
| 双短 decode | 42.2-42.8 t/s | 45.7-53.8 t/s (均值 50.0) | **+18%** |
| 6K 长 prefill + 短请求 | 串行 9.1s+ | long 4.6-8.3s 完成 | 混合加速 |

**机理**：2 序列 token 进同一 batch → CPU 专家矩阵 batch 维度复用（一次权重加载服务两序列）+ GPU attention 批量化 → CPU-GPU 空闲窗口互填。

**结论**：多请求并发（如同时跑对话 + 批处理任务）总吞吐不降反升 +18%，零配置成本（`-np 2` 即可）。

## 结论

**40-42 t/s = 8GB + 消费级 CPU 的物理天花板**。要突破：
1. 换 12GB+ 显卡（4060 Ti 16GB → 60-80 t/s，全专家上 GPU）
2. 等更高效的模型架构发布
3. 散热优化（热态 65°C vs 冷态 40°C 差 ~2 t/s）
