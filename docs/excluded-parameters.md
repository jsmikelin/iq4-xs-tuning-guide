# 已验证不适用参数（40+ 项排除）

> 全部 A/B 实测，避免后人重复踩坑。测试环境：RTX 4060 Laptop 8GB + i9-14900HX + llama.cpp b10375。

## 投机解码（全部净负）

| 技术 | 数据 | 根因 |
|---|---|---|
| MTP 投机解码 (draft-mtp) | -48% | MoE+8GB 无净收益，draft 验证需跑 CPU 专家 |
| Draft Model (draft-simple) | -10% | 干扰 ngram 匹配，MoE 验证开销>收益 |
| ngram-cache | -2.9% | MoE 验证开销同根因 |
| ngram-map-k | -1.6% | 同上 |
| ngram-mod (16/20/4) | -3% | 同上 |
| 共 6 种 ngram 变体 + 2 种 draft | 全负 | draft 验证开销 > 接受 token 收益 |

## KV/显存相关

| 技术 | 数据 | 根因 |
|---|---|---|
| -nkvo (KV offload 到 CPU) | -45% | PCIe 延迟致命：KV 跨 PCIe 访问 16GB/s vs HBM 256GB/s 差 16 倍 |
| TurboQuant fork (turbo4 KV) | 无法加载 | 不支持 Qwen3.6 SSM 层 (missing tensor blk.40.ssm_conv1d.weight) |
| --swa-full | warning | Qwen3.6 hybrid SSM 不支持 |
| --cache-reuse | 无效果 | SSM 层状态不可移位 |

## 线程/调度

| 技术 | 数据 | 结论 |
|---|---|---|
| --prio-batch 2 | -0.4% 噪声 | 线程已继承 --prio 2，无需单独提权 |
| --poll 100 (满速轮询) | -2.2% | 默认 50 已最优 |
| --cpu-strict 1 (+batch) | -3.2% | 严格 CPU 放置限制调度器自由 |
| --cpu-mask 0xFFFF (P-core 绑定) | 无差异 | Windows 调度器已默认放 P-core |
| -tb 16/32 (threads-batch) | 无差异 | 保持默认 |
| --mlock | 无差异 | --load-mode none 下无意义 |
| -np 1 vs -np 4 | +0.8% 噪声 | 维持 -np 2 |

## 硬件控制

| 技术 | 数据 | 结论 |
|---|---|---|
| GPU 锁频 (nvidia-smi -lgc) | -3.0%~-4.5% | 自动 boost 已最优，功耗墙下锁频反害 |
| Fn+F5 增强档 (G-Helper turbo) | +0.7% 噪声边缘 | CPU 已 86% 满血，PL2 余量吃不满 |
| GGML_CUDA_GRAPH_OPT=1 | **-63%** 🔴 | graph 强制重放与 ncmoe CPU 专家串行冲突 |

## 量化

| 技术 | 数据 | 结论 |
|---|---|---|
| Q4_K_M (unsloth UD) | tg 13.5 vs 36.0 t/s | UD 格式与 CUDA 内核路径不兼容，PPL 差 75% |
| Q3_K_XL | 全面劣于 IQ4_XS | CPU offload 场景 dequant 开销吃掉体积优势 |
| APEX 量化 (I-Mini/I-Compact) | 乱码 | GDN 算子不兼容/精度太低 |
| 显式 -b batch | -11% | 默认 batch 行为更优 |

## 参数微调

| 技术 | 数据 | 结论 |
|---|---|---|
| --no-op-offload | prefill -66%, gen -10% 🔴 | host 算子送 GPU 是正确默认 |
| --checkpoint-min-step 64 | -3.6% | 默认 8192 已最优 |
| --slot-prompt-similarity 0.30 | -1.9%~-3.5% | 默认 0.100 已最优 |
| -np 1 + -kvu | -4.2% | 失去并发 |
| --no-prefill-assistant | -4.2%~-6% | assistant 预填充是正向默认 |

## Fork/框架

| 技术 | 结论 |
|---|---|
| ik_llama.cpp v4538 | -21%（fork 太旧，2024-08 同步，不支持 qwen35moe） |
| ktransformers | 24GB VRAM + AVX512/AMX + 800GB RAM 硬门槛，面向 400B+ 超大 MoE |
| llama.moe | 7★ 小项目，未验证 qwen35moe |
| llama.cpp b10451/b10448 | 与 b10375 无差异（76 提交仅 1 个 CUDA 微优化无增益） |
| CUDA 13.3 构建 | 无提升（优化针对 Blackwell sm_120，Ada sm_89 在 CUDA 12.4 已覆盖） |

## 引擎版本

| 版本 | 结论 |
|---|---|
| b10375 (当前) | ✅ 主线最优 |
| b10405→b10451 | 无 MoE/CUDA 相关优化 |
| Qwen3.8 系列 | 2.4T-A95B 太大 / 27B dense 8GB 跑不动，无 35B-A3B |
