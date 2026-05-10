# 证据索引

> **用途**: 追溯数据来源、代码实现、测试覆盖和远端产物

## 本交付包内文件

| 文件 | 说明 |
|---|---|
| `01-vla-model-evaluation-report.md` | 完整 VLA 专项测评报告快照 |
| `02-one-page-brief.md` | 一页纸摘要 |
| `03-presentation-script.md` | 汇报讲稿 |
| `04-work-sync-checklist.md` | 工作同步清单 |
| `05-next-steps-and-risks.md` | 下一步与风险 |
| `06-evidence-index.md` | 当前索引文件 |
| `07-interface-contract-map.md` | 接口、契约、异步任务和状态映射索引 |

## 仓库内正式文档

| 路径 | 说明 |
|---|---|
| `docs/research/RESEARCH-004-vla-model-evaluation-summary.md` | VLA 专项测评工作与结果总结 |
| `docs/research/RESEARCH-003-intel-cup-offline-system-evaluation-report.md` | OpenClaw + Vision + VLA 离线系统评测报告 |
| `docs/guides/intel-openclaw-vla-npu-smoke-2026-04-23.md` | Intel bring-up smoke 记录 |
| `docs/research/RESEARCH-002-model-and-technical-options.md` | 早期模型与技术选型分析 |

## 远端原始产物

这些文件位于远端主机 `intel-cup`，用于追溯原始 JSON 和系统汇总结果：

| 路径 | 说明 |
|---|---|
| `~/remote_eval/vla_benchmark_report.json` | VLA 离线精度、性能、扰动和复现性结果 |
| `~/remote_eval/system_communication_report.json` | OpenClaw + Vision + VLA + Arm Stub 系统通信结果 |
| `~/remote_eval/system_eval_summary.md` | 系统评测汇总 |
| `~/remote_eval/vision_benchmark_report.json` | Vision benchmark 结果 |
| `~/remote_eval/libero_holdout_eval.pt` | 主评测 VLA 样本包 |
| `~/remote_eval/libero_real_samples.pt` | 辅助样本包 |

## 关键代码路径

| 路径 | 说明 |
|---|---|
| `src/openclaw/communication/vla_backend.py` | VLA 后端，包含 SmolVLA、XVLA 加载、推理、扰动和隔离 worker 调用 |
| `src/openclaw/contracts.py` | VLA、Vision、OpenClaw 聚合评测、异步任务状态和错误响应契约 |
| `src/openclaw/controller/vla_task_controller.py` | VLA 异步任务控制器 |
| `src/openclaw/communication/vla_service.py` | VLA HTTP 服务接口 |
| `src/openclaw/communication/vla_service_client.py` | VLA 服务客户端 |
| `src/openclaw/communication/openclaw_service.py` | OpenClaw 聚合评测 HTTP 服务接口 |
| `src/openclaw/communication/openclaw_service_client.py` | OpenClaw 聚合评测服务客户端 |
| `src/openclaw/tools/vla_inference_worker.py` | 独立 VLA 推理 worker |
| `src/openclaw/tools/system_eval.py` | 独立服务进程系统评测工具 |
| `src/openclaw/tools/service_runner.py` | 各服务启动入口 |
| `src/openclaw/controller/task_orchestrator.py` | OpenClaw 聚合编排与安全决策 |
| `src/openclaw/utils/async_tasks.py` | 通用异步任务队列、任务取消和 RESOURCE_BUSY 保护逻辑 |

## 测试覆盖

| 路径 | 覆盖重点 |
|---|---|
| `tests/unit/test_vla_backend.py` | VLA policy cache 复用和切换释放 |
| `tests/unit/test_async_module_eval.py` | VLA 异步任务、OpenClaw 聚合评测、失败状态映射 |
| `tests/unit/test_vla_service.py` | VLA 服务基础行为 |
| `tests/unit/test_openclaw_stack.py` | OpenClaw、Vision、VLA、Arm 的进程内服务链路 |

## 关键接口

| 接口 | 用途 |
|---|---|
| `POST /api/v1/vla/tasks/infer-sample` | 提交 VLA 样本推理任务 |
| `GET /api/v1/vla/tasks/{task_id}` | 查询 VLA 推理任务状态 |
| `POST /api/v1/vla/tasks/{task_id}/cancel` | 取消 VLA 推理任务 |
| `POST /api/v1/openclaw/tasks/evaluate-modules` | 提交 OpenClaw 聚合模块评测任务 |
| `GET /api/v1/openclaw/tasks/{task_id}` | 查询 OpenClaw 聚合评测任务状态 |
| `POST /api/v1/openclaw/tasks/{task_id}/cancel` | 取消 OpenClaw 聚合评测任务 |

## 关键契约

| 契约 | 位置 | 说明 |
|---|---|---|
| `VLAInferenceRequest` | `src/openclaw/contracts.py` | VLA 样本级推理请求 |
| `VLAInferenceResult` | `src/openclaw/contracts.py` | VLA 动作预测结果 |
| `VLAInferenceTaskState` | `src/openclaw/contracts.py` | VLA 异步任务查询响应 |
| `ModuleEvaluationRequest` | `src/openclaw/contracts.py` | OpenClaw 聚合评测请求 |
| `ModuleEvaluationResult` | `src/openclaw/contracts.py` | OpenClaw 聚合评测结果 |
| `ModuleEvaluationTaskState` | `src/openclaw/contracts.py` | OpenClaw 聚合异步任务查询响应 |
| `TaskAccepted` | `src/openclaw/contracts.py` | 异步任务提交响应 |
| `TaskStatus` | `src/openclaw/contracts.py` | 异步任务生命周期状态 |

## 关键数据点

| 数据点 | 值 |
|---|---:|
| SmolVLA Clean Chunk RMSE | 0.972252 |
| XVLA Clean Chunk RMSE | 0.466172 |
| SmolVLA Mean Infer | 1.754524 s |
| XVLA Mean Infer | 3.848381 s |
| SmolVLA Soak | 30/30 成功 |
| XVLA Soak | blocked_on_host |
| 100 次 loopback 成功请求 | 1 |
| Task Status Correctness | 1.000000 |
| Request ID Trace Completeness | 1.000000 |

