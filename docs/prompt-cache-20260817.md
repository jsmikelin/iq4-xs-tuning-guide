# 🏆 cache_prompt 3倍提速 + CPU 双slot 全系 A/B (2026-08-17)

> **Qwen3.6-35B-A3B-IQ4_XS + llama.cpp b10375 + RTX 4060 8GB 实测**

## 1. cache_prompt: 重复请求 3 倍提速 🏆

### 现象
Hermes/agent 每轮对话都重复发送**相同的系统提示词 + 历史**（数千 tokens），每次全量 prefill —— 浪费 70%+ 时间。

### 方案
请求体加 `cache_prompt: True`（llama.cpp 原生支持，OpenAI 兼容端点 `/v1/chat/completions` 同样接受）。

### 实测数据

**标准测速（短请求）**:
| 请求 | cache_prompt | 耗时 | prefill tokens |
|:--|:--|:--|:--|
| 第 1 次 | False | 2.58s | 220 |
| 第 2 次 | True | **0.86s** | **4** |
| 第 3 次 | True | **0.85s** | **4** |

**Hermes 主对话模式（4K tokens 系统提示词 + 变化用户消息）**:
| 轮次 | 耗时 | prefill tokens |
|:--|:--|:--|
| 第 1 轮（冷启动） | 3.14s | 1106 |
| 第 2 轮 | **1.00s** | **21** |
| 第 3 轮 | **0.98s** | **21** |

**结论**: 相同前缀命中 KV 缓存后 prefill 从 1106 tokens → 21 tokens, 总耗时 3.14s → 0.98s (**3.2 倍提速**)。

### 实现要点
- 仅对**本地 llama.cpp (8082)** 注入 `cache_prompt` — OpenAI 官方 API 拒绝未知字段
- 客户端注入位置: chat-completions transport 的 extra_body 组装处（主对话）+ auxiliary client（辅助调用）
- 依赖 server 端 `--cache-idle-slots`（已启用）保持缓存槽位

### 注意
- KV 缓存会随 slot 被其他请求占用而失效（并发越大命中率越低）
- 固定系统提示词的 agent 场景收益最大；完全随机的 cron 脚本收益较小

## 2. CPU 双slot 全系 A/B（-no-kvu 系列）

**用户问题**: "CPU 能否用双并发 slot 的方法论提升执行性能？" → 全系实测。

### 配置语法要点
`-no-kvu` 下每 slot ctx = `-c ÷ np`。想要每 slot 24576 必须 `-c 49152 -np 2`；直接 `-c 24576` 会腰斩到 12288。

### 完整对比表（同环境, v16.0 参数单变量）

| 配置 | slots | n_ctx_each | 单并发 | 2并发组合 | 长prompt | VRAM |
|:--|:--|:--|:--|:--|:--|:--|
| **-kvu -c 24576 (生产)** | 2 | 24576 | 43.3 | **62.2** | 41.7 | 7594 MiB |
| -no-kvu -c 24576 | 2 | 12288 | 44.0 | 62.6 | 43.6 | 7546 MiB |
| -no-kvu -c 49152 | 2 | 24576 | 44.5 | 63.0 | 44.0 | 7848 MiB |
| -no-kvu -c 98304 | 2 | **49152** | 44.3 | 62.7 | 42.5 | 7904 MiB |

### 关键结论
1. **短请求场景**: CPU 双slot（-no-kvu）与生产 -kvu **性能持平**（±3% 噪声），上下文可按需放大到 49152/slot
2. **但长请求 prefill 慢 6.6 倍**: 同长度 ~3-5K 公平对比, -no-kvu 94 t/s vs -kvu 621-695 t/s — KV 在 CPU RAM, 每 token 跨 PCIe 写 KV → 带宽瓶颈
3. **44K 超长请求**: -no-kvu 可跑 (49.4 t/s, 573s); 生产 -kvu 每 slot 24576 上限直接 400 拒绝
4. **方法论教训**: 大上下文 A/B 必须**同长度对比** — 跨长度对比会得出误导性倍数（如"慢14倍"是 44K vs 7K 的错误对比, 同长度实际 6.6 倍）

### 最终配置建议
| 场景 | 配置 |
|:--|:--|
| Hermes 主对话 / 常规生产 | **-kvu -c 24576 -np 2** (62.2 t/s 组合, VRAM 最省) |
| 高并发短请求批处理 | -no-kvu -c 24576 -np 4 (4并发 57.6 t/s +23%) |
| 偶尔超长上下文 (24K-49K) | -no-kvu -c 98304 -np 2 (慢但可用) |
