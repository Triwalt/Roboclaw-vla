# 接口与契约变更索引

> **用途**: 给同事快速定位接口、契约和异步任务相关改动 | **日期**: 2026-05-11

## 总览

当前接口、契约相关变更仍在父仓库 `Steaming-Fresh/Roboclaw` 的主代码树中，不在 `vla` submodule 内。`vla` submodule 只保存 VLA 测评报告与交付物。

这些接口改动服务于两条链路：

1. VLA 样本级异步推理链路：提交一个 VLA 样本推理任务，轮询任务状态，获取动作预测结果。
2. OpenClaw 聚合模块评测链路：由 OpenClaw 统一触发 Vision、VLA、Arm Stub，并返回结构化模块评测结果。

## 核心文件位置

| 类别 | 父仓库路径 | 作用 |
|---|---|---|
| 数据契约 | `src/openclaw/contracts.py` | 定义 VLA policy、扰动、任务状态、推理请求、推理结果、模块评测结果等 Pydantic 模型 |
| VLA 服务端接口 | `src/openclaw/communication/vla_service.py` | 新增 VLA 异步推理任务提交、查询、取消接口 |
| VLA 客户端 | `src/openclaw/communication/vla_service_client.py` | 封装 VLA 任务提交、轮询、取消和 retryable poll error 重试 |
| OpenClaw 服务端接口 | `src/openclaw/communication/openclaw_service.py` | 新增 OpenClaw 聚合模块评测任务提交、查询、取消接口 |
| OpenClaw 客户端 | `src/openclaw/communication/openclaw_service_client.py` | 封装 OpenClaw 聚合评测任务提交、轮询、取消 |
| VLA 任务控制器 | `src/openclaw/controller/vla_task_controller.py` | 管理 VLA 异步推理队列，默认 `max_pending=1` |
| OpenClaw 编排器 | `src/openclaw/controller/task_orchestrator.py` | 编排 Vision、VLA、Arm Stub，并映射安全决策和任务状态 |
| 异步任务基础设施 | `src/openclaw/utils/async_tasks.py` | 通用异步任务队列、任务状态、取消和 RESOURCE_BUSY 逻辑 |
| 系统评测工具 | `src/openclaw/tools/system_eval.py` | 使用独立服务进程调用上述接口进行 soak 与故障注入 |

## 新增或扩展的数据契约

位置：`src/openclaw/contracts.py`

| 契约 | 说明 |
|---|---|
| `TaskStatus` | 通用异步任务状态：`accepted / running / succeeded / rejected / failed / cancelling / cancelled` |
| `TaskAccepted` | 任务提交后的统一响应，包含 `task_id`、`service`、`request_id` |
| `TaskState` | 任务查询基础状态，包含提交、开始、完成时间和错误 |
| `VLAPolicy` | 当前支持 `smolvla` 与 `xvla` |
| `VLAPerturbation` | VLA 离线评测扰动枚举：clean、亮度降低、噪声、中心遮挡、状态噪声、prompt paraphrase |
| `VLAInferenceRequest` | VLA 样本推理请求，包含 policy、model_id、sample bundle、sample index、device、perturbation、timeout |
| `VLAInferenceResult` | VLA 推理结果，包含 action chunk、expert action chunk、timings、memory、details |
| `VLAInferenceTaskState` | VLA 推理任务查询响应 |
| `ModuleEvaluationRequest` | OpenClaw 聚合评测请求，包含 hazard 图像、VLA 样本、policy、device、是否运行 arm stub |
| `ModuleEvaluationResult` | 聚合评测结果，包含 Vision 结果、VLA 结果、安全决策、Arm 结果、时延和错误列表 |
| `ModuleEvaluationTaskState` | OpenClaw 聚合评测任务查询响应 |
| `SafetyDecision` | OpenClaw 聚合评测中的保护动作决策 |
| `LatencyBreakdown` | Vision、VLA、Arm、总时延拆分 |
| `HazardDetection` / `BoundingBox` | Vision detector 输出的检测框与置信度结构 |

## VLA 异步推理接口

服务端位置：`src/openclaw/communication/vla_service.py`

| 方法 | 路径 | 请求 | 响应 | 说明 |
|---|---|---|---|---|
| `POST` | `/api/v1/vla/tasks/infer-sample` | `VLAInferenceRequest` | `TaskAccepted` | 提交一个 VLA 样本推理任务 |
| `GET` | `/api/v1/vla/tasks/{task_id}` | path: `task_id` | `VLAInferenceTaskState` | 查询任务状态和结果 |
| `POST` | `/api/v1/vla/tasks/{task_id}/cancel` | path: `task_id` | `VLAInferenceTaskState` | 取消任务 |

客户端位置：`src/openclaw/communication/vla_service_client.py`

客户端新增方法：

- `infer_sample(request)`
- `get_task(task_id)`
- `cancel_task(task_id)`
- `wait_for_task(task_id, poll_interval_s=1.0, timeout_s=120.0)`

重要行为：

- `wait_for_task` 会轮询到终态：`succeeded / failed / rejected / cancelled`。
- 对 retryable 的 `DEPENDENCY_UNAVAILABLE` 和 `TIMEOUT` 轮询错误会短暂重试。
- 超过等待时间后会尝试取消任务。

## OpenClaw 聚合评测接口

服务端位置：`src/openclaw/communication/openclaw_service.py`

| 方法 | 路径 | 请求 | 响应 | 说明 |
|---|---|---|---|---|
| `POST` | `/api/v1/openclaw/tasks/evaluate-modules` | `ModuleEvaluationRequest` | `TaskAccepted` | 提交 Vision + VLA + Arm Stub 聚合评测任务 |
| `GET` | `/api/v1/openclaw/tasks/{task_id}` | path: `task_id` | `ModuleEvaluationTaskState` | 查询聚合评测任务状态和结果 |
| `POST` | `/api/v1/openclaw/tasks/{task_id}/cancel` | path: `task_id` | `ModuleEvaluationTaskState` | 取消聚合评测任务 |

客户端位置：`src/openclaw/communication/openclaw_service_client.py`

客户端新增方法：

- `evaluate_modules(request)`
- `get_task(task_id)`
- `cancel_task(task_id)`
- `wait_for_task(task_id, poll_interval_s=1.0, timeout_s=180.0)`

## 异步任务队列契约

位置：`src/openclaw/utils/async_tasks.py`

`AsyncTaskQueue` 是当前 VLA 和 OpenClaw 聚合评测共用的异步任务基础设施。

| 行为 | 说明 |
|---|---|
| `submit` | 接收 `request_id` 和 job factory，返回 `TaskAccepted` |
| `get` | 查询 `StoredTask` |
| `cancel` | 取消 accepted 或 running 任务 |
| `max_pending` | 限制排队任务数，超限返回 `RESOURCE_BUSY` |
| `RESOURCE_BUSY` | retryable 错误，用于保护重模型服务不被并发压垮 |
| `should_cancel` | job 内部可检查取消状态 |

当前使用方式：

- VLA 推理队列：`AsyncTaskQueue("vla", max_pending=1)`
- OpenClaw 聚合评测队列：`AsyncTaskQueue("openclaw", max_pending=4)`

## OpenClaw 编排和安全决策

位置：`src/openclaw/controller/task_orchestrator.py`

新增聚合评测流程：

1. 查询 Runtime health。
2. 调用 Vision 服务，处理 `hazard_image_path`。
3. 调用 VLA 服务，提交 `VLAInferenceRequest` 并轮询结果。
4. 根据 Vision 风险、VLA 状态和错误列表生成 `SafetyDecision`。
5. 如果 `run_arm_stub=true` 且安全动作需要执行，则调用 Arm Stub。
6. 返回 `ModuleEvaluationResult`。

安全决策规则：

| 条件 | 动作 |
|---|---|
| Vision 返回 high 或 critical hazard | `stop_arm` |
| VLA 失败、缺失或系统错误 | `hold_position` |
| Vision 无阻塞风险且 VLA 成功 | `none` |

任务状态映射：

- 如果内部 `ModuleEvaluationResult.status` 是 failed，即使异步 job 本身完成，也会把外层 `ModuleEvaluationTaskState.status` 映射为 `failed`。
- 这样可以避免“任务执行完成”被误解为“模块评测成功”。

## 测试覆盖位置

| 测试 | 覆盖内容 |
|---|---|
| `tests/unit/test_async_module_eval.py` | VLA 异步任务、OpenClaw 聚合任务、retryable poll error、失败状态映射 |
| `tests/unit/test_vla_backend.py` | VLA policy cache 复用与切换释放 |
| `tests/unit/test_vla_service.py` | VLA 服务基础计划接口 |
| `tests/unit/test_openclaw_stack.py` | OpenClaw、Vision、VLA、Arm 的进程内链路 |

## 当前交接重点

- 接口和契约实现都在父仓库，不在 `vla` submodule。
- `vla` submodule 是汇报和交付材料仓库，用来解释这些接口如何服务 VLA 测评。
- 如果同事要改 API schema，优先看 `src/openclaw/contracts.py`。
- 如果同事要改 HTTP 路由，优先看 `vla_service.py` 和 `openclaw_service.py`。
- 如果同事要改轮询、取消、超时、重试，优先看两个 service client 和 `AsyncTaskQueue`。
- 如果同事要改安全动作或聚合逻辑，优先看 `TaskOrchestrator`。

