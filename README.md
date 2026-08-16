# IQ4_XS 8GB VRAM 推理优化指南

> **Qwen3.6-35B-A3B-UD-IQ4_XS 在 8GB 显卡上的完整调优方法论**
> 40+ 参数实测数据 · 13 项已部署配置 · 全部 A/B 验证

## 🎯 适用配置

| 组件 | 规格 |
|---|---|
| GPU | RTX 4060 Laptop 8GB (Ada, sm_89) |
| CPU | i9-14900HX 24C/32T (无 AVX512) |
| RAM | 32GB DDR5 |
| 模型 | Qwen3.6-35B-A3B-UD-IQ4_XS (16.51GB, 4.25bpw, 40层, 256专家×8激活) |
| 引擎 | llama.cpp b10375 (CUDA 12.4) |

## 🚀 当前最优配置 (v16.0)

```bash
llama-server.exe \
  -m Qwen3.6-35B-A3B-IQ4_XS.gguf \
  --host 0.0.0.0 --port 8082 \
  -dev CUDA0 \
  -t 24 -ngl 41 -ncmoe 27 \
  --override-kv qwen35moe.expert_used_count=int:7 \
  -fa 1 --load-mode none \
  -c 24576 -np 2 \
  -ctk q8_0 -ctv q8_0 \
  -ub 2048 \
  --cache-ram 4096 --cache-idle-slots \
  -fit off --prio 2 --no-host -kvu
```

### 实测性能
| 指标 | 数值 |
|---|---|
| 短 prompt decode | **42.2 t/s** (40.7-42.8) |
| 7K 长 prompt prefill | **695 t/s** (vs 基线 498, +40%) |
| 7K 长 prompt gen | 40.0 t/s |
| 上下文 | 24576 tokens/单slot (vs 8192, **3倍**) |
| 2 请求并发组合吞吐 | **50+ t/s** (vs 单请求 42.5, +18%) |
| VRAM | ~7.6GB / 8GB |

## 📁 目录

| 文件 | 内容 |
|---|---|
| [docs/tuning-guide.md](docs/tuning-guide.md) | 13 项已部署参数详解 + A/B 数据 |
| [docs/excluded-parameters.md](docs/excluded-parameters.md) | 40+ 项已验证不适用参数（含数据） |
| [docs/ab-methodology.md](docs/ab-methodology.md) | A/B 测试方法论 + 陷阱 |
| [docs/hardware-boundaries.md](docs/hardware-boundaries.md) | 硬件瓶颈调查（串行/PCIe/新方案） |
| [docs/decision-tree.md](docs/decision-tree.md) | 优化决策树 |

## ⚡ 快速收益点（按性价比排序）

1. **`-kvu` + `-c 24576`** — 上下文 3 倍零速度损失（旧配置 >8K 请求直接 400 拒绝）
2. **`-ub 2048`** — prefill +40%（batch 单独调优，不要 b+ub 同改）
3. **`--override-kv qwen35moe.expert_used_count=int:7`** — decode +12.3%，PPL 仅 +2%
4. **`-t 24`（物理核数）** — 比默认 16 快 +21.7%（MoE CPU 专家场景）
5. **`-ncmoe 27`** — VRAM 物理极限最优点

## 📊 版本演进

| 版本 | 变更 | 日期 |
|---|---|---|
| v16.0 | `-ub 2048` prefill +40% | 2026-08-16 |
| v15.0 | `-kvu -c 24576` 上下文 3 倍 | 2026-08-16 |
| v13.1 | `--prio 2` +2.4% | 2026-08-13 |
| v13.0 | `-t 24` 物理核数 +21.7% | 2026-08-13 |
| v12.1 | e7 专家减量 +12.3% | 2026-08-13 |

## 📄 License

MIT
