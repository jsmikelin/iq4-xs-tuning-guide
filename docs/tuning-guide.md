# 13 项已部署参数详解

所有参数均经过 A/B 实测（同环境、同 prompt、同 max_tokens 对比），GPU 独占测试。

## 1. Flash Attention `-fa 1`
- 必开，零副作用
- 减少 KV cache VRAM + 加速长 prompt

## 2. ncmoe 部分专家 GPU `-ncmoe 27`
- 40 层中 13 层专家放 GPU，27 层专家放 CPU
- V-curve 实测确认 27 = VRAM 物理极限最优点：

| ncmoe | GPU专家层 | VRAM | decode | 结论 |
|:---:|:---:|:---:|:---:|---|
| 24 | 16 | 8.3GB (溢出) | 17.3 t/s | ❌ swap 暴跌 |
| 25 | 15 | 8.1GB (溢出) | 17.6 t/s | ❌ swap 暴跌 |
| **27** | **13** | **7.9GB** | **36.5 t/s** | ✅ 最优点 |
| 30 | 10 | 6.8GB | 30.4 t/s | ❌ CPU 往返增加 |

## 3. KV cache q8_0 `-ctk q8_0 -ctv q8_0`
- KV 16bit→8bit 量化，+16% 速度，质量损失极小

## 4. `--load-mode none`
- 懒加载，+6.4%，省 3.6GB RAM

## 5. `-t 24`（物理核数）🏆
- **t16=32.95 → t24=40.09 (+21.7%) → t32=37.55**
- 机理：-ncmoe 27 把 27 层专家放 CPU → CPU 线程数 = 专家并行度
- 超线程（t32）对 MoE 矩阵计算无益反害（缓存/调度争抢）
- **铁律：MoE 模型 benchmark 必须显式 -t <物理核数>，默认 t16 低估 ~20%**

## 6. `-ub 2048`（ubatch 调优）🏆
- **prefill 498→695 t/s (+40%)，decode 零损失**
- A/B 数据：

| 配置 | 短prompt gen | 7K prefill | 7K gen |
|---|---|---|---|
| -ub 1024 (基线) | 42.2 t/s | 498 t/s | 39.6 t/s |
| **-ub 2048** | **42.2 t/s** | **695 t/s (+40%)** | **40.0 t/s** |
| -ub 4096 | 41.0 t/s | 687 t/s | 38.8 t/s (-3%) |

- ⚠️ **教训**：v1.2.0 曾测 "ub=2048,b=2048 -9%" 是 b+ub 同改的混合变量结论——**单独改 -ub 2048（b 保持默认 2048）是 +40%**。A/B 必须单变量！

## 7. expert_used_count=7 专家减量 🏆
`--override-kv qwen35moe.expert_used_count=int:7`
- 默认激活 8/256 专家 → 强制 7 个
- **生产实测 +12.3% (36.23→40.69 t/s)，PPL 仅 +2.0%**
- A3B MoE 冗余度高，8→7 只丢 12.5% 专家

| 配置 | PPL (ctx=2048) | tg128 | PPLΔ |
|:---:|:---:|:---:|:---:|
| e8 (基线) | 5.4640 | 36.23 | — |
| **e7** | **5.5748** | **37.70** | **+2.0%** |

## 8. `--prio 2` 进程优先级
- +2.4% (8轮复测 38.70 vs 37.80，方向一致非噪声)

## 9. `-kvu` KV-unified + `-c 24576` 🏆
- KV 统一池：单 slot 上下文 8192→24576（**3 倍**），速度零损失
- 旧配置 >8K 请求 400 拒绝，现在可处理 24K+

## 10. `-np 2` 双 slot
- 支持 2 请求并发，连续批处理自动生效
- **并发实测：2 请求组合吞吐 45.7-53.8 t/s（均值 50.0）vs 单请求 42.5 = +18%**
- 机理：2 序列同 batch → CPU 专家矩阵 batch 维度复用 + GPU attention 批量化

## 11. `--cache-idle-slots`
- 空闲 slot 保留 KV 缓存，重复请求近乎瞬时（实测 7K 重复 prompt wall 1.7s）

## 12. `--cache-ram 4096`
- KV cache 放 CPU RAM（Qwen3.6 hybrid SSM 仅 10/40 层有 attention KV，总量极小）

## 13. `-fit off`
- 关闭 fit 模式，避免动态调整干扰

## 质量保障
- 所有调用需传 `"chat_template_kwargs":{"enable_thinking":false}` —— 否则模型生成 reasoning_content 消耗全部 token 预算，content 返回空（有效输出 = 0）
