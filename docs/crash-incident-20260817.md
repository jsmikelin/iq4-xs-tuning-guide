# 🚨 0xC0000374 崩溃事故全解 (2026-08-17)

> **Qwen3.6-35B-A3B-IQ4_XS + llama.cpp b10375 + RTX 4060 8GB 环境实测**
> 事故持续 2.5+ 小时，排除 15+ 项，最终由一次真正系统重启解决。

## 事故现象

所有 llama-server/llama-cli 加载模型后必崩：

- bash exit 127 = Windows **0xC0000374 (STATUS_HEAP_CORRUPTION)**
- minidump 确认崩溃地址在 ntdll.dll 堆损坏检测点（RtlReportCriticalFailure，是报告点非源头）
- 崩溃点（--log-file 无缓冲日志定位）：`resolve_fused_ops` 成功 → `sched_reserve` 完成 → **warmup 首次执行计算图时崩**
- 任何 -c 值（36864/32768/49152/24576/4096）、任何模式（含纯 CPU `-ngl 0`）全部崩溃

## 根因链

1. **并发 CUDA 初始化**：测试进程与看门狗 schtasks 拉起的生产进程（间隔 9 秒）同时初始化 CUDA → 第二个进程 0xC0000374 堆损坏
2. **污染共享状态**：崩溃污染 CUDA 驱动共享状态 → 之后**所有** llama 进程继承崩溃（Ollama 独立进程正常，但日志出现 `could not determine compute capability for CUDA device`）
3. **唯一幸存者 = 最先启动者**：看门狗先拉起的进程（21:21:51）成功加载，测试进程（21:22:00）崩 —— 证明与 -c 值无关

## 🔴 伪重启陷阱（关键教训）

用户报告"已重启电脑"后崩溃依旧 —— 因为 Windows **快速启动 (Fast Startup)**：

- 快速启动 = 关机时内核会话休眠 + 开机恢复，**CUDA 驱动状态原封不动**
- 判定方法：`Get-CimInstance Win32_OperatingSystem | LastBootUpTime` 未变成当天 = 伪重启
- **修复**：`HiberbootEnabled=0`（HKLM:\SYSTEM\CurrentControlSet\Control\Session Manager\Power）+ 真正 `shutdown /r`

## 排查清单（15+ 项全排除）

| 排除项 | 方法 |
|:--|:--|
| -c 值区间 | 36864/32768/49152/24576/4096 全崩（含曾成功的 49152） |
| Ollama 显存预占 | 停 Ollama 后仍崩 |
| GPU 驱动损坏 | pnputil /restart-device 后仍崩；Ollama 0.5b 推理成功证明硬件健康 |
| 模型文件损坏 | NO_BUFFERING 物理读（绕过缓存）GGUF magic + 随机块全通过 |
| 环境变量 | 无 LLAMA_ARG_* / GGML_* 污染 |
| 资源不足 | RAM 20.5GB 可用 / 页面文件 87GB |
| mmap 路径 | --no-mmap 仍崩 |
| JIT 缓存损坏 | 无 llama.cpp 缓存文件 |
| shell 环境 | schtasks 独立干净任务也崩 |
| 模型文件缓存页 | 复制新文件仍崩 |
| -fit on 默认 | -fit off 仍崩 |
| warmup | --no-warmup 仍崩 |

## 日志无输出的假象

stdout/stderr 重定向是 4KB 块缓冲，崩溃时未 flush → 看似"零输出"。
必须 `--log-file`（无缓冲写文件）或从事件日志/minidump 找真崩溃点。

## 恢复验证

真正重启后 → llama-cli 单次推理 → 生产 bat → health + slots + 推理三连确认。

---

# 🔬 2×36864 大上下文 A/B (2026-08-17)

**用户问题**："2 slots × 36864 非标配置是否是崩溃原因？" → **不是崩溃原因，但性能腰斩。**

## A/B 结果（重启后同环境，v16.0 参数单变量 -c）

| 配置 | slots | n_ctx_each | 1并发 | 2并发(每请求) | 2并发组合吞吐 | 长prompt | VRAM |
|:--|:--|:--|:--|:--|:--|:--|:--|
| **-kvu -np 2 -c 24576 (生产)** | 2 | 24576 | **43.3** | **31.1** | **62.2 t/s (+44%)** | **41.7** | 7594 MiB |
| -kvu -np 2 -c 36864 (测试) | 2 | **36864** | **19-21** (-52%) | **12.1** | **24.2 t/s** (-61%) | **21.0** (-50%) | 6698 MiB |

## 根因

-c 36864 × 2 slots 的 KV 池超出显存可容纳量 → `--cache-ram 4096` 限制触发 KV 溢出到 CPU RAM → 每 token 跨 PCIe 拉 KV → decode 减半。VRAM 反而更低 (6698 vs 7594) = KV 被挤出显存的证据。

## 结论

2×36864 能启动、能推理、不崩溃，但性能 -50~60%。**大上下文 (24K+ 单请求) 场景可临时切 2×49152；常规生产铁定维持 2×24576**（单并发 43.3 t/s，双并发组合 62.2 t/s）。

## 附加坑

`taskkill /F /PID` 杀 bash wrapper 不会杀 llama-server 子进程 → 残留双进程抢同一端口（slots 显示旧配置）。清理必须 `taskkill /F /IM llama-server.exe` 按镜像名杀。
