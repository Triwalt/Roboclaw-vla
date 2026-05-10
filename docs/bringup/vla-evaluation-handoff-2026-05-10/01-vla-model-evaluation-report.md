# RESEARCH-004: VLA 模型专项测评工作与结果总结

> **版本**: v1.1.0 | **状态**: 评审中 | **关联**: RESEARCH-003-intel-cup-offline-system-evaluation-report.md | **作者**: Codex | **最后更新**: 2026-05-10

---

## 1. 文档目的

本文档汇总当前已经确认的 VLA 模型测评工作、实验条件、结果数据和工程结论。它面向后续选型、复测、实机前置检查和阶段性汇报使用。

考虑到本文档会作为汇报材料使用，正文中对 VLA、LIBERO、RMSE、soak、seed、XPU、loopback、故障注入等概念和参数做了补充解释。读者不需要直接参与过本轮测试，也可以理解每组指标代表什么、能说明什么、不能说明什么。

本文档只纳入已经完成并有明确记录的结果，不把未接入的模型资产、未完成的真实相机链路或未执行的实机动作写成已验证能力。

当前已确认完成测评的 VLA 模型为：

- `lerobot/smolvla_base`，下文简称 `SmolVLA`
- `lerobot/xvla-libero`，下文简称 `XVLA`

这里的 VLA 指 Vision-Language-Action 模型，即输入视觉观察、语言任务描述和机器人状态后，输出动作序列的模型。与只回答问题的视觉语言模型不同，VLA 的输出面向机器人执行，因此除了“看懂”和“理解指令”，还要关注动作预测质量、推理耗时、长时间服务稳定性和失败后的安全处理。

`~/GreenVLA` 仓库在远端主机上存在，但当前没有可归档的专项测评结果，因此不列为已完成测评模型。

---

## 2. 已确认结论摘要

| 维度 | 已确认结论 |
|---|---|
| 离线精度 | `XVLA` 在当前 LIBERO holdout 上显著优于 `SmolVLA` |
| 推理速度 | `SmolVLA` 单次推理更快，服务响应更适合当前主机 |
| 稳定性 | `SmolVLA` 已通过 30 次 soak；`XVLA` 长稳 soak 在当前主机上被阻塞 |
| 可复现性 | 两个模型 same-seed 输出均为 0 diff |
| 扰动鲁棒性 | `XVLA` 对当前人工扰动几乎无变化；`SmolVLA` 对噪声和 seed 更敏感 |
| 系统集成 | VLA 异步推理接口、OpenClaw 聚合评测接口和结构化错误传播已打通 |
| 当前推荐 | `SmolVLA` 作为服务可用性基线，`XVLA` 作为离线精度基线和优化目标 |

一句话结论：当前不是“某个模型全面胜出”，而是 `XVLA` 更准、`SmolVLA` 更稳；实机前应先把 VLA 服务长稳性和目标工况数据补齐。

### 2.1 汇报口径

本次汇报建议按“三个问题”展开：

1. 模型能不能跑：两类 VLA 模型均已完成真实推理路径验证。
2. 模型跑得好不好：`XVLA` 离线动作预测更接近专家轨迹，`SmolVLA` 精度较弱。
3. 模型能不能稳定服务化：`SmolVLA` 当前更稳，`XVLA` 当前主机上长稳性不足。

因此，本文档中的“精度结论”和“工程可用结论”需要分开理解。`XVLA` 更适合作为研究和离线精度基线，`SmolVLA` 更适合作为当前主机上的服务可用性基线。

### 2.2 关键概念解释

| 概念 | 解释 | 本文中的作用 |
|---|---|---|
| VLA | Vision-Language-Action，视觉-语言-动作模型 | 用于从图像、任务文本和机器人状态预测动作序列 |
| LeRobot | Hugging Face 生态中的机器人学习工具链 | 本轮 VLA 模型加载、预处理和推理依赖该运行时 |
| LIBERO | 机器人操作任务 benchmark 数据集 | 本轮用其 holdout 样本做离线动作预测评估 |
| holdout 样本 | 评测时保留、没有用于当前测试调参的样本 | 用于观察模型在未参与调试样本上的表现 |
| action chunk | 一段连续动作序列，而不是单个动作 | 更接近机器人实际执行时的短时动作计划 |
| expert action | 数据集中专家轨迹或参考动作 | 用作离线比较目标，计算模型输出和参考动作的距离 |
| offline evaluation | 离线评测，不连接真实机械臂执行 | 能评估动作预测相似度，但不能直接等价于实机成功率 |
| soak | 长稳性测试，连续多次请求观察是否稳定 | 用于判断模型服务是否能承受持续调用 |
| loopback | 本机回环 HTTP 调用，地址绑定在 `127.0.0.1` | 用于验证服务边界和通信协议，不引入外部网络变量 |
| fault injection | 故障注入，人为制造超时、繁忙、加载失败等情况 | 用于确认系统是否能结构化报错并进入保护动作 |

### 2.3 指标和参数解释

| 指标或参数 | 含义 | 如何解读 |
|---|---|---|
| RMSE | Root Mean Square Error，均方根误差 | 越低表示模型动作越接近专家动作 |
| First-step RMSE | 只比较动作序列第一步的误差 | 反映模型第一下动作是否接近参考动作 |
| Chunk RMSE | 比较整段 action chunk 的平均误差 | 反映整段短时动作计划质量，是本轮最核心的离线精度指标 |
| Future-step RMSE | 比较第一步之后未来动作的误差 | 反映模型后续动作延展是否稳定 |
| Beat Repeat Baseline Rate | 超过“重复当前动作”简单基线的比例 | 越高说明模型越可能学到比静态重复更有效的动作策略 |
| Mean Infer | 平均推理时间 | 越低表示平均响应更快 |
| P95 Infer | 95 分位推理时间 | 用于观察长尾延迟，在线服务比平均值更关注该指标 |
| Mean Load | 平均模型加载时间 | 影响冷启动、模型切换和服务恢复速度 |
| RSS | 进程常驻内存 | 用于观察内存占用和是否存在内存漂移 |
| XPU memory | Intel XPU 设备侧显存或加速器内存 | 用于观察硬件加速路径的资源压力 |
| same-seed diff | 固定随机种子时输出差异 | 越接近 0 表示可复现性越好 |
| varied-seed diff | 改变随机种子时输出差异 | 越大表示模型对随机性越敏感 |
| `xpu` | Intel GPU/NPU 相关加速执行路径 | 本轮 VLA 推理主要按该设备参数运行 |

---

## 3. 评测环境与资产

本节说明测试在哪里跑、用什么运行时跑、测了哪些模型和数据。汇报时需要强调：本轮是“离线评测 + 本机服务通信评测”，还不是 D435i 实时视觉和 ARX X5 实机动作闭环。

### 3.1 主机与运行环境

| 项目 | 确认状态 |
|---|---|
| 远端主机 | `intel-cup` |
| OpenClaw 主运行面 | Python 3.10 |
| VLA 运行面 | `~/miniforge3/envs/vla_xpu_min`，Python 3.12 |
| Torch | `2.11.0+xpu` |
| Transformers | `5.3.0` |
| LeRobot | `0.5.2` |
| OpenVINO 设备 | `CPU / GPU / NPU` 可见 |
| VLA 设备参数 | 主要使用 `xpu` |
| SmolVLA 资产模式 | `HF_HUB_OFFLINE=1` 配合本地 cache |

### 3.2 模型与数据

`SmolVLA` 和 `XVLA` 都来自 LeRobot 生态，但两者定位不同。本轮不是只比较“谁的 RMSE 更低”，还同时比较“谁能在当前主机上长期作为服务运行”。

| 对象 | 路径或标识 | 用途 |
|---|---|---|
| SmolVLA | `lerobot/smolvla_base` | 服务可用性与离线动作预测基线 |
| XVLA | `lerobot/xvla-libero` | 离线精度基线 |
| 主评测样本 | `~/remote_eval/libero_holdout_eval.pt` | 12 个 LIBERO holdout 样本 |
| 辅助样本 | `~/remote_eval/libero_real_samples.pt` | 缓存样本质检，不计入主分数 |

### 3.3 原始结果产物

远端正式产物路径如下：

- `~/remote_eval/vla_benchmark_report.json`
- `~/remote_eval/system_communication_report.json`
- `~/remote_eval/system_eval_summary.md`
- `~/remote_eval/vision_benchmark_report.json`

其中本文档主要引用 `vla_benchmark_report.json` 和 `system_communication_report.json` 的归纳结果；系统级背景可参考 `docs/research/RESEARCH-003-intel-cup-offline-system-evaluation-report.md`。

---

## 4. 已完成测评工作

本节按工作流列出已经完成的验证项。它回答的是“我们到底做了哪些测试”，而不是只给最终分数。

### 4.1 环境审计

已完成内容：

- 检查 Python、Torch、Transformers、LeRobot、OpenVINO 版本。
- 检查 `torch.xpu.is_available()`。
- 检查 Hugging Face cache 是否满足离线运行。
- 检查 `/dev/dri/renderD128` 与 OpenVINO `available_devices`。
- 验证 VLA service 可在独立环境中启动。

确认结果：

- `SmolVLA` 和 `XVLA` 都已完成真实推理路径验证。
- `SmolVLA` 依赖离线 cache 才能稳定载入。
- `XVLA` 可完成真实推理，但长稳服务运行受主机资源影响明显。

### 4.2 VLA 离线动作预测评测

离线动作预测评测的核心思想是：给模型同样的观察和任务描述，让模型输出一段动作序列，再和数据集中的专家动作序列比较。这个过程不驱动真实机械臂，因此可以低风险地比较模型趋势，但不能直接替代实机任务成功率。

主评测设计：

- 样本数：12 个 LIBERO holdout 样本。
- 每模型请求数：72。
- 组合方式：`12 样本 x 6 扰动`。

扰动集合：

- `clean`
- `brightness_down`
- `gaussian_noise`
- `center_occlusion`
- `state_noise`
- `prompt_paraphrase`

输出指标：

- `first-step RMSE`
- `chunk RMSE`
- `future-step RMSE`
- `beat_repeat_baseline_rate`
- `load_s`
- `mean / p95 infer_s`
- RSS / XPU memory
- same-seed reproducibility
- varied-seed variability

### 4.3 长稳性测评

长稳性测评关注“模型作为服务连续被调用时是否仍然健康”。这对机器人系统很重要，因为实机不是只调用一次模型，而是会持续接收感知、规划和安全检查请求。

已完成内容：

- `SmolVLA` 独立 soak。
- `XVLA` soak 尝试。
- 记录请求成功数、平均耗时、p95、延迟漂移和 RSS 变化。

确认结果：

- `SmolVLA` 30 次请求全部成功。
- `XVLA` 当前主机上未产出成功的长稳 soak 结果，相关进程会被系统杀掉。

### 4.4 系统通信测评

系统通信测评关注“模块之间能否按约定互相调用，并在失败时给出可追踪、可处理的结果”。本轮使用独立进程和 HTTP loopback，目的是更接近未来服务化部署形态。

已完成内容：

- 使用独立服务进程而不是进程内 smoke。
- 通过 OpenClaw 聚合接口触发 Vision + VLA + Arm Stub。
- 进行 100 次正式 HTTP loopback soak。
- 进行故障注入，确认 VLA 失败路径是否结构化返回。

涉及接口：

- `POST /api/v1/vla/tasks/infer-sample`
- `GET /api/v1/vla/tasks/{task_id}`
- `POST /api/v1/openclaw/tasks/evaluate-modules`
- `GET /api/v1/openclaw/tasks/{task_id}`

确认结果：

- request_id 可以跨 OpenClaw、Vision、VLA、Arm Stub 传播。
- VLA 失败不会静默吞掉，而是转换为结构化错误。
- OpenClaw 在 VLA 失败时会进入保护性动作，例如 `hold_position`。

---

## 5. VLA 离线精度结果

本节只讨论离线动作预测质量。这里的“精度”不是视觉检测精度，也不是实机抓取成功率，而是模型动作序列与专家动作序列之间的数值距离。

### 5.1 Clean 指标对比

`clean` 表示不对输入图像、状态或任务文本做额外扰动，是最基础的对照条件。

| 模型 | First-step RMSE | Chunk RMSE | Future-step RMSE | Beat Repeat Baseline Rate | Mean Infer (s) | P95 Infer (s) |
|---|---:|---:|---:|---:|---:|---:|
| SmolVLA | 1.061404 | 0.972252 | 0.969649 | 0.000000 | 1.754524 | 3.078370 |
| XVLA | 0.471396 | 0.466172 | 0.465850 | 0.500000 | 3.848381 | 5.256000 |

解读：

- `XVLA` 在三个 RMSE 指标上均明显低于 `SmolVLA`。
- `XVLA` clean 样本中有一半超过 repeat baseline。
- `SmolVLA` clean 样本未超过 repeat baseline，说明当前数据集上动作质量不足以作为精度优势模型。
- `SmolVLA` 推理更快，`XVLA` 推理更慢但动作更接近专家轨迹。
- 汇报时可简化为：`XVLA` 的动作更像专家，`SmolVLA` 的响应更快。

### 5.2 扰动结果

扰动测试用于观察模型对输入变化的敏感程度。这里的扰动是人工合成的轻量扰动，目的是做初步鲁棒性筛查。

| 扰动 | SmolVLA Chunk RMSE 相对 clean | XVLA Chunk RMSE 相对 clean |
|---|---:|---:|
| `brightness_down` | +0.081% | +0.003% |
| `gaussian_noise` | +3.171% | +0.000% |
| `center_occlusion` | -2.389% | -0.007% |
| `state_noise` | -0.483% | +0.000% |
| `prompt_paraphrase` | -5.261% | +0.001% |

解读：

- `XVLA` 对当前人工扰动几乎不敏感，表现出更稳定的离线动作预测。
- `SmolVLA` 在 `gaussian_noise` 下退化最明显。
- `SmolVLA` 在部分扰动下指标反而变好，不能解释为扰动有益；更合理的解释是样本量较小、扰动较弱和模型随机性共同造成局部波动。
- 汇报时应避免说“扰动提升了 SmolVLA”，更准确的说法是“当前小样本扰动下没有形成稳定退化规律，但噪声项已有可见退化”。

---

## 6. 性能与复现性结果

本节关注模型运行成本和输出稳定性。机器人在线系统不仅要求动作预测质量，还要求模型能在可接受时间内响应，并且相同条件下不要出现不可解释的随机漂移。

| 模型 | Mean Load (s) | Mean Sample Infer (s) | Max Same-seed Diff | Mean Varied-seed Diff |
|---|---:|---:|---:|---:|
| SmolVLA | 17.074773 | 1.452024 | 0.000000 | 0.560380 |
| XVLA | 26.602483 | 19.653128 | 0.000000 | 0.000055 |

解读：

- 两个模型 same-seed 输出均可复现。
- `SmolVLA` varied-seed 差异明显，说明 seed 对输出影响更大。
- `XVLA` varied-seed 差异极小，当前样本下几乎无 seed 漂移。
- `XVLA` 加载和单样本推理耗时明显更高，服务化成本更大。
- 汇报时可以把这组数据解释为：`XVLA` 更稳地输出相似动作，但启动和运行都更重；`SmolVLA` 更轻，但随机性更明显。

---

## 7. 长稳性结果

长稳性结果是当前选型最关键的工程指标之一。离线精度高的模型，如果无法在目标主机上长期稳定运行，就不能直接作为默认在线服务。

### 7.1 SmolVLA soak

| 指标 | 值 |
|---|---:|
| Requests | 30 |
| Successful Requests | 30 |
| Failed Requests | 0 |
| Mean Total (s) | 1.459181 |
| P95 Total (s) | 1.544636 |
| Latency Drift Last5 vs First5 | 12.097% |
| RSS Delta (MB) | 0.633 |

结论：

- `SmolVLA` 已具备当前主机上的离线服务级重复执行能力。
- 30 次请求中没有失败。
- RSS 变化很小，未观察到明显内存漂移。
- 这说明 `SmolVLA` 至少已经达到“可继续扩大测试规模”的状态，但 30 次还不能等同于生产级稳定。

### 7.2 XVLA soak

当前确认状态：

- `blocked_on_host`
- `xvla_soak_current.json` 未成功产出。
- 已确认该主机上进行 XVLA 长稳 soak 时相关进程会被系统杀掉。

结论：

- `XVLA` 当前只能作为离线精度参考。
- `XVLA` 暂不适合作为当前 `intel-cup` 主机上的长期在线服务默认模型。
- 这不否定 `XVLA` 的模型能力，而是说明当前软硬件组合还没有满足它的服务化运行条件。

---

## 8. 系统集成与故障注入结果

本节回答“模型接入系统后，服务之间能不能稳定协作”。它和第 5 节的离线精度不同，重点是接口、状态、trace 和失败保护。

### 8.1 100 次 loopback 结果

| 指标 | 值 |
|---|---:|
| Requests | 100 |
| Successful Requests | 1 |
| Request Success Rate | 0.010000 |
| Task Status Correctness | 1.000000 |
| Request ID Trace Completeness | 1.000000 |
| Mean Vision Latency (ms) | 1604.133 |
| Mean VLA Latency (ms) | 22513.895 |
| Mean End-to-end Latency (ms) | 24214.765 |

重要说明：

- 上表中的时延均值只基于成功请求集合。
- 本轮只有 1 条成功请求，因此时延均值实际对应第一条 `SmolVLA` 成功请求。
- 第二条 `XVLA` 请求在失败前持续约 `301943.113 ms`，这是更能反映问题的长尾现象。
- 因此，这组时延不能作为“100 次请求平均性能”对外陈述，只能作为“唯一成功请求的性能画像”使用。

按策略拆分：

| Policy | Requests | Successful | 主要现象 |
|---|---:|---:|---|
| SmolVLA | 50 | 1 | 首次成功，随后被 XVLA 失稳拖挂后进入结构化失败 |
| XVLA | 50 | 0 | 交替请求触发 VLA 服务失稳，后续全部失败 |

结论：

- 系统级失败主因不是 OpenClaw 协议或任务状态映射。
- 主要问题是 `XVLA` 在当前主机上无法稳定承担在线服务角色。
- 即使请求失败，系统仍能保持任务状态正确性和 trace 完整性。
- 汇报时建议把这个结果表述为：功能成功率低，但失败可观测性和安全退化路径成立。

### 8.2 故障注入

故障注入不是为了追求成功率，而是为了验证系统面对坏情况时是否可控。对机械臂系统来说，失败能被识别、能被追踪、能触发保护动作，比失败后无响应或误执行更重要。

| 场景 | 确认结果 |
|---|---|
| `vla_busy` | 正确返回 `RESOURCE_BUSY` |
| `smolvla_load_failure` | 正确返回 `MODEL_LOAD_FAILED` |
| `vla_timeout` | 正确返回 `TIMEOUT` |
| `vision_accelerator_unavailable` | 正确返回 `DEPENDENCY_UNAVAILABLE` |
| `vision_empty_detection` | 返回结构化空 hazard 结果，`max_risk_level=low` |

VLA 相关结论：

- VLA 队列可拒绝超出并发限制的任务。
- 模型加载失败和推理超时可被结构化表达。
- 上层 OpenClaw 可以基于失败结果做保护性决策，而不是继续下发风险动作。

---

## 9. 已落地的工程支撑

本节说明“为了完成测评，系统里已经补齐了哪些工程能力”。这些能力不是单纯的测试脚本，而是后续继续做模型复测、服务化验证和实机前置检查的基础设施。

### 9.1 VLA 后端能力

代码路径：

- `src/openclaw/communication/vla_backend.py`
- `src/openclaw/controller/vla_task_controller.py`
- `src/openclaw/communication/vla_service.py`
- `src/openclaw/tools/vla_inference_worker.py`

已落地能力：

- 支持 `smolvla` 和 `xvla` 两类 `VLAPolicy`。
- 支持默认模型映射：
  - `smolvla -> lerobot/smolvla_base`
  - `xvla -> lerobot/xvla-libero`
- 支持异步 VLA 推理任务。
- VLA 推理队列 `max_pending=1`，用于避免并发压垮模型进程。
- 默认使用隔离 worker 执行真实推理，降低主服务被模型推理拖死的风险。
- 切换 active policy 时会释放旧 policy，并尝试清理 XPU cache。
- 支持扰动样本生成和 action chunk RMSE 评估所需输出。

汇报解读：

- 异步任务接口使 VLA 推理不会阻塞 OpenClaw 主请求线程。
- `max_pending=1` 是有意设计的保护策略，避免多个重模型并发推理把主机资源打满。
- 隔离 worker 的作用是把真实模型推理放在独立进程中执行，即使模型推理异常，也尽量不拖垮 VLA 主服务。
- policy 切换时释放旧模型，是为了降低 `SmolVLA` 和 `XVLA` 交替测试时的内存压力。

### 9.2 系统评测工具

代码路径：

- `src/openclaw/tools/system_eval.py`
- `src/openclaw/tools/service_runner.py`

已落地能力：

- 独立启动 `arm / vision / vla / runtime / openclaw` 五个服务。
- 使用真实 HTTP loopback，而不是进程内假调用。
- 将 soak 和 faults 分阶段起服务，避免资源污染。
- 输出服务日志路径、逐请求记录、聚合成功率、trace 完整性和故障注入结果。

汇报解读：

- 独立服务进程比进程内 mock 更接近真实部署形态。
- soak 和 faults 分阶段启动，可以避免前一轮资源失稳影响后一轮故障注入判断。
- 逐请求记录使失败链路可以追踪到具体 request_id、模型策略、服务状态和错误码。

### 9.3 单元测试覆盖

相关测试路径：

- `tests/unit/test_vla_backend.py`
- `tests/unit/test_async_module_eval.py`
- `tests/unit/test_vla_service.py`
- `tests/unit/test_openclaw_stack.py`

已覆盖重点：

- VLA policy cache 复用。
- policy 切换时旧模型释放。
- VLA 异步任务接口返回推理结果。
- OpenClaw 聚合评测跨服务流转。
- retryable poll error 重试。
- 内部失败结果映射为外层 failed task state。

汇报解读：

- 当前不仅有一次性实验结果，也有单元测试保护关键接口行为。
- 测试重点覆盖的是缓存、异步任务、错误映射和跨服务编排，这些都是 VLA 服务化最容易出问题的部分。

---

## 10. 选型建议

本节给出当前阶段的建议，而不是最终定型。由于本轮尚未接入真实 D435i 流和 ARX X5 实机动作，建议应理解为“进入下一阶段验证的默认路线”。

### 10.1 当前默认策略

| 场景 | 推荐模型 | 理由 |
|---|---|---|
| 当前主机上的服务化默认模型 | SmolVLA | 30 次 soak 全成功，推理更快，资源压力更低 |
| 离线精度对照与研究基线 | XVLA | RMSE 明显更低，扰动结果更稳定 |
| 100 次以上在线集成 soak | 暂不使用 XVLA 默认上线 | 当前主机上长稳性不足 |
| 实机动作前置验证 | SmolVLA + 保护性规则 | 模型输出必须经过 SafetyGuard，不直接透传到底层机械臂 |

对外表述建议：

- `XVLA`：作为“精度表现更好的候选模型”，后续重点解决服务化和资源问题。
- `SmolVLA`：作为“当前可稳定跑通服务链路的候选模型”，后续重点补齐目标工况数据和更长 soak。
- OpenClaw：保持模型输出和机械臂执行之间的安全隔离，不让模型直接控制底层动作。

### 10.2 不建议事项

- 不建议把 `XVLA` 直接设为当前主机的在线默认 VLA 服务。
- 不建议用本轮 LIBERO holdout RMSE 直接宣称真实机械臂任务成功率。
- 不建议把 hazard 图像检测结果和 LIBERO VLA 样本拼成“联合任务成功率”。
- 不建议绕过 SafetyGuard 直接把 VLA action chunk 转为 ARX X5 执行动作。

原因说明：

- `XVLA` 的离线精度优势已经确认，但当前主机上的在线稳定性尚未确认。
- LIBERO 离线数据不等于目标工位真实任务数据，不能直接推导实机成功率。
- hazard 检测和 VLA action chunk 来自不同评测输入，强行合并成联合成功率会误导结论。
- 机械臂动作有安全风险，模型输出必须经过规则层、状态检查和安全放行链。

---

## 11. 风险与局限

本节是汇报时需要主动说明的边界。主动说明边界可以避免结果被过度解读，也能清楚交代下一阶段为什么还需要继续测试。

### 11.1 数据局限

- VLA 主评测数据来自 LIBERO holdout，不是目标工位真实采集数据。
- 样本数为 12，适合作为工程 smoke 与初步比较，不足以支撑最终模型结论。
- `prompt_paraphrase` 等扰动规则较简单，不能代表完整自然语言鲁棒性。

影响：

- 当前结果适合判断模型相对趋势，不适合作为最终上线验收依据。
- 后续必须构建目标工位 replay 数据，才能判断模型是否适配真实任务。

### 11.2 系统局限

- 当前结果不包含 D435i live stream。
- 当前结果不包含 ARX X5 实机控制。
- 当前系统通信评测中的 arm 仍是 protected stub。
- `XVLA` 长稳性受当前主机资源约束，仍需进一步拆解是内存、XPU、模型实现还是进程管理问题。

影响：

- 当前系统结果只能说明服务接口和离线链路成立。
- 是否能进入真实抓取、移动、避障或安全闭锁，还需要实机前置验证。

### 11.3 指标解释局限

- RMSE 只能衡量 action chunk 与专家轨迹的距离，不等价于真实任务成功。
- `SmolVLA` 部分扰动指标变好不能解释为模型更鲁棒。
- 系统 loopback 中的平均时延只来自成功样本集合，不能当作全量请求平均值。

影响：

- 对外汇报应避免使用“成功率”描述 RMSE。
- 对外汇报 100 次 loopback 时，应同时说明“功能成功率低”和“失败处理路径成立”。

---

## 12. 下一步工作

下一步建议分成三条线推进：模型服务稳定性、目标工况数据、实机安全闭环。三条线缺一不可。

### 12.1 短期

1. 为 `XVLA` 增加进程监督、自动重启和资源上限记录。
2. 记录 `XVLA` 被系统杀掉时的内核日志、RSS 峰值和 XPU memory 峰值。
3. 将 100 次系统 soak 拆成 `SmolVLA-only`、`XVLA-only`、`交替策略` 三组，避免交叉污染。
4. 复跑 `SmolVLA` 100 次服务级 soak，确认当前 30 次结果能否扩展。

短期目标：

- 分清 `XVLA` 失败是模型资源压力、设备路径、进程管理还是交替加载导致。
- 把 `SmolVLA` 从 30 次稳定扩展到更有说服力的 100 次稳定。

### 12.2 中期

1. 构建目标工位 replay 数据集，替代纯 LIBERO holdout。
2. 增加真实 D435i 图像输入后的 VLA 前处理一致性检查。
3. 为 `SmolVLA / XVLA` 增加按任务类型分桶的指标统计。
4. 评估是否引入更小的 XVLA 配置或更大内存主机。

中期目标：

- 将评测从通用 benchmark 迁移到目标工位 replay。
- 让模型比较更贴近真实场景，而不是只依赖 LIBERO。

### 12.3 实机前置门槛

推进到 ARX X5 实机动作前，建议至少满足：

1. 默认 VLA 服务 100 次连续请求成功率达到可接受阈值。
2. VLA 失败路径全部能稳定返回结构化错误。
3. OpenClaw 保护动作链从 arm stub 扩展到真实安全放行链。
4. 建立目标工位数据集上的离线 replay 指标，不只依赖 LIBERO。

实机前置门槛的含义：

- VLA 不是越早接实机越好，而是要先证明服务稳定、失败可控、保护动作可靠。
- 只有当离线 replay、服务 soak 和安全链路都达到门槛后，才适合进入真实机械臂动作验证。

---

## 13. 证据索引

本节列出本文档结论对应的仓库证据和原始报告入口。汇报时可以把它作为“可追溯性”说明。

| 证据 | 路径 |
|---|---|
| 系统级离线评测报告 | `docs/research/RESEARCH-003-intel-cup-offline-system-evaluation-report.md` |
| VLA 后端实现 | `src/openclaw/communication/vla_backend.py` |
| VLA 异步任务控制器 | `src/openclaw/controller/vla_task_controller.py` |
| VLA 推理隔离 worker | `src/openclaw/tools/vla_inference_worker.py` |
| 系统评测工具 | `src/openclaw/tools/system_eval.py` |
| VLA 后端单元测试 | `tests/unit/test_vla_backend.py` |
| 异步模块评测单元测试 | `tests/unit/test_async_module_eval.py` |

---

## 14. 汇报摘要版本

如果需要在会议中口头汇报，可使用以下摘要：

本轮已完成 `SmolVLA` 和 `XVLA` 两个 VLA 模型在 `intel-cup` 主机上的离线推理、扰动、复现性、长稳性和系统通信验证。离线精度上，`XVLA` 的 action chunk RMSE 明显低于 `SmolVLA`，说明它在 LIBERO holdout 上更接近专家轨迹；但工程可用性上，`SmolVLA` 已完成 30 次 soak 全成功，而 `XVLA` 在当前主机上长稳运行会被系统杀掉。因此当前建议是：`XVLA` 作为离线精度基线和优化目标，`SmolVLA` 作为当前服务可用性基线。

系统集成方面，VLA 异步推理接口、OpenClaw 聚合评测接口、request_id trace 和结构化错误传播已经打通。100 次交替 loopback 的功能成功率只有 1%，主因是 `XVLA` 服务失稳；但任务状态正确性和 trace 完整性均为 100%，说明失败路径可观测、可追踪，并能触发保护性决策。

本轮结果不等价于实机任务成功率，因为还没有接入 D435i live stream 和 ARX X5 实机动作。下一阶段应优先完成 `SmolVLA-only / XVLA-only / 交替策略` 的拆分 soak、记录 `XVLA` 资源失稳原因，并构建目标工位 replay 数据集，再推进实机前置验证。
