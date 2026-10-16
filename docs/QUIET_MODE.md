# 安静模式（外部控制器 + 启动参数）

安静模式 = 在后台以较低资源运行不急于出结果的长期任务；用户有其他高负载工作时立即让出资源，负载结束后再恢复。

MLTF 本身**不监测负载、不管理任务**。它只提供“按参数运行”的能力，由外部控制器检测整机负载并切换参数。任务进度（哪些请求已完成）由调用方保存，MLTF 不做断点。

## 可用于限制资源的参数

这些参数都在**启动时**设定，运行中不能改。

| 参数 | 限制的资源 | 安静模式建议值 |
|---|---|---|
| `--gpu-duty-percent N` | GPU 占空比（1–100）。每条命令结束后暂停 `命令耗时 × (100−N)/N` | `20`–`30` |
| `--max-concurrent-requests` | 并发请求数（1–4） | `1` |
| `--max-context` | 最大上下文，决定 KV 占用 | 按任务需要缩小 |
| `--max-memory` | 引擎内存上限（不能低于权重，约 31 GiB） | 视情况 |
| `--idle-reclaim-percent` | 空闲 30 秒后，整机内存占用达到该比例就释放空 KV | `30` |

`--gpu-duty-percent` 是为安静模式新增的：nice/QoS 对 Metal 计算基本无效，只有让引擎自己停顿，GPU 才会真正降速。每条命令之后的暂停是 `命令耗时 × (100−N)/N`，但**单次暂停上限 250 ms**：实测发现，暂停过长时 GPU 会降频，下一条命令耗时被放大约 9 倍，公式又据此算出更长的暂停，形成恶性循环（N=25 时 256 个 token 用了约 4 分钟，仅 0.9 tok/s）。加上限后 decode 命令耗时恢复到约 95 ms。所以 decode 阶段的有效占空比下限约 28%（95 ms 命令配 250 ms 暂停），N 小于 28 时不会再慢下去。

## 控制器的做法（路径 A+B：切换即重启）

1. 控制器定期读取整机 CPU、GPU（`ioreg` 的 AGXAccelerator `Device Utilization %`）和内存压力（`memory_pressure -Q`）。
2. **进入高负载**：停止 MLTF 服务（`mltf service stop` 或 `launchctl bootout`），约 30 GiB 权重和 GPU 全部释放。
3. **负载低于阈值并持续 N 秒**（带滞回，避免抖动）：用安静参数启动服务，例如
   `mltf serve --gpu-duty-percent 25 --max-concurrent-requests 1 --idle-reclaim-percent 30`，
   然后让客户端继续发送未完成的请求（请求可用 `priority: "background"`）。
4. 被打断的请求在重启后由客户端重发；已完成的由客户端跳过。
5. 判定时要排除 MLTF 自己造成的 GPU/CPU 占用，否则安静模式会把自己当成高负载。例如只在服务停止或请求间隙采样，或使用进程级 CPU 占用。

代价：每次切换都要重新加载模型（数十秒），内存中的前缀缓存丢失（配置 `--ssd-root` 可保留部分）。

## 后续设想：运行时调参接口（路径 C，未实现）

目标：不重启、不重新加载模型，秒级切换工作档位。

- 新增管理接口，例如 `POST /admin/profile`，字段：`max_concurrent_requests`（`0` 表示暂停接收新请求）、`gpu_duty_percent`、`reclaim_now`（立即释放空 KV，不等 30 秒）。接口默认只对本机、并要求 API key。
- Python 层：`WaitingAdmission` 的容量改为可调（当前创建后固定）；暂停时不取消在途请求，只停止放行新请求。
- 原生层：`FrameType` 新增一种控制帧（如 `ConfigUpdate`），携带 `dutyPercent` 等；`Engine` 在 owner 线程上更新 `config_.dutyPercent`，`reclaim_now` 复用现有 `MemoryPressurePolicy` 的零目标释放。协议两侧都要改，并要更新特性位（`protocol::kNativeFeatureBits`）和兼容测试。
- 限制：权重（约 30 GiB）仍常驻，退出安静模式只能释放 GPU 和 KV，不能释放权重；需要彻底腾出内存时仍要停服务。
- 控制器可在两档之间选择：短时高负载用“暂停 + 热切换”，长时间高负载用“停服务”。

## 实测（M3 Max 128 GB，Qwen3.8-27B q8c，上下文 8K，并发 1）

同一请求（256 token，temperature 0），生成速度取 3 次中位数：

| `--gpu-duty-percent` | 生成速度 | 相对 100% |
|---:|---:|---:|
| 100（不限速） | 26.0 tok/s | 100% |
| 50 | 15.4 tok/s | 59% |
| 25（带 250 ms 暂停上限） | 8.7 tok/s | 33% |

注意：`ioreg` 的 `Device Utilization %` 在三档下都在 88–99%，不能用来衡量限速效果，GPU 功耗和温度未测量（需要 `powermetrics`，要 sudo）。已验证的只有“同一任务变慢、GPU 有空闲间隙”，不能据此宣称省电。

## 验证

- `python3 -m unittest tests.test_gpu_duty_percent`：参数解析和透传。
- 实测：同一请求在 `--gpu-duty-percent` 100/50/25 下比较生成速度和 GPU 利用率，见 PR 描述。
