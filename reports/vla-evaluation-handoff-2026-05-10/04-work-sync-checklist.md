# 工作同步清单

> **用途**: 交接、站会同步、后续任务拆解

## 已确认完成

| 项目 | 状态 | 说明 |
|---|---|---|
| VLA 环境审计 | 已完成 | 已确认 Python、Torch、Transformers、LeRobot、OpenVINO、XPU 路径 |
| SmolVLA 真实推理 | 已完成 | `lerobot/smolvla_base` 可真实推理，依赖离线 cache |
| XVLA 真实推理 | 已完成 | `lerobot/xvla-libero` 可真实推理 |
| LIBERO holdout 离线评测 | 已完成 | 12 样本 x 6 扰动，每模型 72 请求 |
| SmolVLA 长稳性测试 | 已完成 | 30/30 请求成功 |
| XVLA 长稳性尝试 | 已完成 | 当前主机 blocked，进程会被系统杀掉 |
| OpenClaw 聚合评测接口 | 已完成 | `POST /api/v1/openclaw/tasks/evaluate-modules` |
| VLA 异步推理接口 | 已完成 | `POST /api/v1/vla/tasks/infer-sample` |
| request_id trace | 已完成 | 系统 loopback trace 完整性为 100% |
| 故障注入 | 已完成 | busy、load failure、timeout、accelerator unavailable 等场景已有结构化结果 |

## 当前明确结论

- `XVLA` 离线动作预测更准。
- `SmolVLA` 当前主机上服务运行更稳。
- 当前系统主线的主要阻塞不是 OpenClaw 协议，而是 `XVLA` 在线服务稳定性。
- 本轮结果不能外推为实机任务成功率。
- VLA 输出不能绕过 SafetyGuard 直接下发到底层机械臂。

## 尚未完成

| 项目 | 原因 | 建议负责人关注点 |
|---|---|---|
| `SmolVLA-only` 100 次 soak | 当前只有 30 次结果 | 扩大稳定性样本 |
| `XVLA-only` 独立 soak | 当前交替测试会污染判断 | 分离模型自身问题和交替加载问题 |
| `XVLA` 资源峰值记录 | 当前只确认被系统杀掉 | 记录 RSS、XPU memory、内核日志 |
| 目标工位 replay 数据集 | 当前使用 LIBERO holdout | 建立更贴近真实场景的离线评测 |
| D435i live stream 接入 | 本轮未包含 | 验证真实相机输入和前处理一致性 |
| ARX X5 实机安全链 | 本轮 arm 是 stub | 建立真实安全放行链 |
| GreenVLA 测评 | 当前仅确认仓库存在 | 需要独立接入与可复现结果 |

## 需要同步给同事的注意事项

- 不要把 `XVLA` 的离线精度优势说成“已适合上线”。
- 不要把 `SmolVLA` 的 30 次 soak 说成“生产稳定”。
- 不要把 LIBERO RMSE 说成“抓取成功率”。
- 不要把 100 次 loopback 的时延均值说成全量平均，因为只有 1 条成功请求。
- 可以强调当前系统失败可观测性较好：状态正确性和 trace 完整性均为 100%。

## 建议分工

| 工作流 | 主要任务 |
|---|---|
| VLA/Data | 扩展模型 soak、整理目标工位 replay、补 GreenVLA 接入评测 |
| Runtime/Infra | 增加进程监督、自动重启、资源峰值记录 |
| Vision/Camera | 接入 D435i live stream，检查 VLA 输入前处理一致性 |
| Safety/Arm | 将 arm stub 保护动作扩展为真实安全放行链 |
| Reporting | 维护本交付包和完整测评报告，避免结论口径漂移 |

