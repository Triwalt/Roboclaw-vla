# VLA Evaluation Handoff Package

> **版本**: v1.1.0 | **状态**: 可交付 | **日期**: 2026-05-11 | **用途**: 同事汇报与工作同步

## 使用方式

本文件夹整理了当前 VLA 模型测评工作的汇报材料和工作同步材料。建议按以下顺序使用：

1. 先读 `02-one-page-brief.md`，快速掌握结论。
2. 汇报时使用 `03-presentation-script.md`。
3. 工作交接时使用 `04-work-sync-checklist.md`。
4. 安排后续任务时使用 `05-next-steps-and-risks.md`。
5. 需要追溯证据时查看 `06-evidence-index.md`。
6. 需要定位接口和契约改动时查看 `07-interface-contract-map.md`。
7. 需要完整背景和详细数据时查看 `01-vla-model-evaluation-report.md`。

## 文件清单

| 文件 | 用途 |
|---|---|
| `01-vla-model-evaluation-report.md` | 完整 VLA 专项测评报告 |
| `02-one-page-brief.md` | 一页纸摘要，适合发群或会前预读 |
| `03-presentation-script.md` | 汇报讲稿和建议话术 |
| `04-work-sync-checklist.md` | 工作同步清单，说明已完成、未完成、需确认事项 |
| `05-next-steps-and-risks.md` | 下一步计划、风险、门槛 |
| `06-evidence-index.md` | 原始报告、代码、测试和远端产物索引 |
| `07-interface-contract-map.md` | 接口、契约、异步任务和状态映射位置索引 |

## 核心结论

当前已确认完成测评的 VLA 模型是 `SmolVLA` 和 `XVLA`。`XVLA` 离线动作预测更接近专家轨迹，适合作为离线精度基线；`SmolVLA` 在当前主机上更稳定，适合作为服务可用性基线。

本轮结果不等价于实机任务成功率，因为尚未接入 D435i live stream 和 ARX X5 实机动作。

## 接口与契约位置

接口和契约实现仍在父仓库 `Steaming-Fresh/Roboclaw` 的主代码树中，不在本 `vla` submodule 内。本 submodule 只保存 VLA 测评报告、交付包和接口契约索引。

核心位置：

- `src/openclaw/contracts.py`
- `src/openclaw/communication/vla_service.py`
- `src/openclaw/communication/vla_service_client.py`
- `src/openclaw/communication/openclaw_service.py`
- `src/openclaw/communication/openclaw_service_client.py`
- `src/openclaw/controller/vla_task_controller.py`
- `src/openclaw/controller/task_orchestrator.py`
- `src/openclaw/utils/async_tasks.py`

