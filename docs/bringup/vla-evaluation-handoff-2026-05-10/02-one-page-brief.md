# VLA 模型测评一页纸摘要

> **日期**: 2026-05-10 | **适用场景**: 会前预读、群内同步、汇报第一页

## 背景

本轮工作围绕 OpenClaw 项目的 VLA 模型接入与测评展开。VLA 指 Vision-Language-Action 模型，即输入视觉观察、任务文本和机器人状态后，输出动作序列。本轮目标是确认候选 VLA 模型是否能在当前 `intel-cup` 主机上完成真实推理、离线评测、长稳性测试和系统通信验证。

## 已测模型

| 模型 | 标识 | 当前定位 |
|---|---|---|
| SmolVLA | `lerobot/smolvla_base` | 当前主机上的服务可用性基线 |
| XVLA | `lerobot/xvla-libero` | 离线精度基线和后续优化目标 |

`GreenVLA` 仅确认远端仓库存在，当前没有可归档测评结果，不列入已测模型。

## 关键结果

| 维度 | SmolVLA | XVLA |
|---|---|---|
| Clean Chunk RMSE | 0.972252 | 0.466172 |
| Beat Repeat Baseline Rate | 0.000000 | 0.500000 |
| Mean Infer | 1.754524 s | 3.848381 s |
| P95 Infer | 3.078370 s | 5.256000 s |
| Same-seed Diff | 0.000000 | 0.000000 |
| 长稳性 | 30/30 成功 | 当前主机 blocked |

## 结论

- 精度：`XVLA` 明显更接近专家动作轨迹。
- 速度：`SmolVLA` 单次推理更快。
- 稳定性：`SmolVLA` 当前更适合作为服务化基线。
- 系统接入：VLA 异步接口、OpenClaw 聚合评测接口、request_id trace 和结构化错误传播已打通。
- 当前最大阻塞：`XVLA` 在当前主机上的长稳服务运行不足。

## 对外口径

当前不是“某个模型全面胜出”，而是：

- `XVLA` 更准，适合作为离线精度基线。
- `SmolVLA` 更稳，适合作为当前主机服务可用性基线。

本轮结果不等价于实机任务成功率，因为尚未接入 D435i live stream 和 ARX X5 实机动作。

## 下一步

1. 拆分复跑 `SmolVLA-only`、`XVLA-only`、`交替策略` 三组 soak。
2. 记录 `XVLA` 被系统杀掉时的 RSS、XPU memory 和内核日志。
3. 构建目标工位 replay 数据集。
4. 将 OpenClaw 保护动作链从 arm stub 扩展到真实安全放行链。

