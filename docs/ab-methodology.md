# A/B 测试方法论

测试任何参数变更的标准流程，全部实测经验总结。

## 标准流程

1. **基线**：当前配置跑 5 轮 (64-256 tok)，记录均值
2. **杀 server**：`taskkill /F /IM llama-server.exe`
3. **改参数重启**：等 health check 通过
4. **触发模型加载**：发 1 条小请求 (max_tokens=5)
5. **跑相同 5 轮**：相同 prompt，相同 max_tokens
6. **对比**：t/s, range, VRAM, Free RAM, 稳定性

## 🔴 陷阱（全部实测踩过）

### 1. 守护进程自动拉起（最坑）
如果 server 由守护脚本管理（如 Windows 计划任务 + bat 循环），**杀进程后会被自动拉起** → 双进程 → 测速被污染 (40→2.6 t/s)。
```bash
# A/B 前必须禁用守护
schtasks /change /tn <任务名> /disable
# 测完重新启用
schtasks /change /tn <任务名> /enable
```

### 2. 单变量原则
**A/B 必须一次只改一个参数**。反例：v1.2.0 测 "ub=2048,b=2048 -9%" 得出错误结论，后来单独测 `-ub 2048`（b 保持默认）发现是 **+40%**。混合变量 = 无效数据。

### 3. 后台请求污染
后台长生成请求 (3000-token) 仍在 slot 中运行时会污染测量 (30.5 vs 干净 41.2 t/s)。测速前等请求清空 (sleep ≥8s)。

### 4. GPU 独占
双 server 同时跑 prefill 会暴跌 (0.71 t/s)。测速必须确认只有一个 server 进程。

### 5. MoE 必须显式 -t
llama-bench 默认 t16，在 24 核机器上低估 ~20%。MoE 模型 (-ncmoe>0) benchmark 必须 `-t <物理核数>`。

### 6. 温度/状态
热态 (65°C+) 比冷态慢 ~2 t/s。对比不同日期的数据注意温度差异。

### 7. llama-bench 不支持 --override-kv
那是 llama-server 专属参数。llama-bench 公平对比必须显式 `-ncmoe N`（不传默认 0 = 全专家 CPU，数字完全失真：IQ4 tg128 从 36.0 掉到 7.04 t/s）。

## 标准 benchmark 命令

```bash
# 短 prompt
curl -s http://127.0.0.1:8082/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"messages":[{"role":"user","content":"Write a Python function to check if a number is prime."}],"max_tokens":128,"temperature":0.0}' \
  | jq '.timings'

# llama-bench 三件套（同参数可比）
llama-bench -m model.gguf -ngl 41 -t 24 -ncmoe 27 -fa 1 -p 512 -n 128
```

## 测量要点

- **llama-perplexity 语料量门槛**：`-c N` 需要 ≥2×N tokens（`-c 8192` 需 ≥16384 tokens）
- **PPL 对比必须同语料同 ctx**：跨 ctx 的 PPL 不可比
- **sha256 验证陷阱**：HF LFS OID 就是裸 sha256（无 `sha256:` 前缀），误加前缀会误报"文件损坏"
- **Windows 健康检查用 socket 而非 urllib**：urllib 可能挂起，用 `socket.create_connection` + 裸 HTTP GET /health
