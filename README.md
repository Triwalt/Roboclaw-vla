# vla

`vla` 是面向 OpenClaw VLA 模型评测、服务化验证和后续模型接入的独立资料仓库，作为 `Roboclaw` 的 submodule 引入。

## 当前定位

- 保存 VLA 测评报告、汇报交付物和接口契约索引。
- 记录 `SmolVLA`、`XVLA` 等候选模型在当前目标平台上的离线精度、性能和稳定性结论。
- 为后续 VLA service、benchmark scripts、模型 adapter 独立演进预留仓库边界。

当前 VLA 运行时代码仍保留在父仓库 `Roboclaw` 中，避免破坏现有 `openclaw` 包导入路径。关键父仓库路径包括：

- `src/openclaw/communication/vla_backend.py`
- `src/openclaw/communication/vla_service.py`
- `src/openclaw/communication/vla_service_client.py`
- `src/openclaw/controller/vla_task_controller.py`
- `src/openclaw/controller/task_orchestrator.py`
- `src/openclaw/contracts.py`

## 当前可用材料

- `docs/README.md`：文档索引。
- `docs/bringup/vla-evaluation-handoff-2026-05-10/`：VLA 测评汇报与工作同步交付包。
- `docs/research/RESEARCH-004-vla-model-evaluation-summary.md`：VLA 专项测评完整报告。
- `docs/subsystem/vla_subsystem_blueprint.md`：VLA 子模块边界、职责和后续演进建议。

## 当前结论

- `XVLA` 在当前 LIBERO holdout 上离线动作预测更接近专家轨迹。
- `SmolVLA` 在当前 `intel-cup` 主机上更适合作为服务可用性基线。
- 当前结果不等价于实机任务成功率，因为尚未接入 D435i live stream 和 ARX X5 实机动作。

## 后续方向

1. 将目标工位 replay 数据集和 VLA benchmark scripts 沉淀到本仓库。
2. 在父仓库契约稳定后，评估将 VLA adapter 和 service 实现迁入本仓库。
3. 保持 VLA 输出必须经过 OpenClaw 安全决策链，不直接透传到底层机械臂。

