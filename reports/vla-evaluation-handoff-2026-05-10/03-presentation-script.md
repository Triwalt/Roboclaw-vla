# VLA 模型测评汇报讲稿

> **用途**: 5-10 分钟阶段性汇报 | **听众**: 项目同事、工程负责人、后续接手人

## 1. 开场

这次汇报主要同步当前 VLA 模型测评工作的阶段性结论。VLA 是 Vision-Language-Action 模型，用来根据视觉输入、任务文本和机器人状态预测动作序列。我们本轮关注三个问题：模型能不能跑、动作预测效果如何、能不能稳定作为服务接入 OpenClaw。

## 2. 已测范围

本轮已确认完成测评的模型有两个：

- `SmolVLA`，模型标识是 `lerobot/smolvla_base`。
- `XVLA`，模型标识是 `lerobot/xvla-libero`。

远端主机上虽然存在 `GreenVLA` 仓库，但目前没有可归档的专项测评结果，所以这次不把它作为已测模型汇报。

测试环境是远端主机 `intel-cup`。OpenClaw 主运行面是 Python 3.10，VLA 运行面是 `~/miniforge3/envs/vla_xpu_min`，使用 Python 3.12、Torch `2.11.0+xpu`、Transformers `5.3.0` 和 LeRobot `0.5.2`。

## 3. 怎么测

我们做了四类测评：

1. 环境审计：确认 Python、Torch、LeRobot、OpenVINO、XPU 和模型 cache 可用。
2. 离线动作预测：用 12 个 LIBERO holdout 样本，按 6 类扰动组合，每个模型共 72 次请求。
3. 长稳性测试：连续请求模型服务，观察成功率、延迟漂移和内存变化。
4. 系统通信测试：通过 OpenClaw 聚合接口走 Vision、VLA、Arm Stub，并验证故障注入和结构化错误传播。

这里要强调，当前是离线评测和本机服务通信评测，不是实机任务成功率测试。

## 4. 核心指标解释

本轮最核心的离线精度指标是 RMSE，也就是模型预测动作和专家动作之间的均方根误差。RMSE 越低，说明模型输出越接近专家轨迹。

其中：

- `First-step RMSE` 看第一步动作。
- `Chunk RMSE` 看整段动作序列，是最核心的指标。
- `Future-step RMSE` 看后续动作延展。
- `Beat Repeat Baseline Rate` 看模型是否超过“重复当前动作”这个简单基线。
- `P95 Infer` 看长尾推理延迟，比平均值更能反映服务体验。
- `soak` 是长稳性测试，用来观察连续请求下服务是否稳定。

## 5. 主要结果

离线精度上，`XVLA` 明显优于 `SmolVLA`：

- `SmolVLA` clean chunk RMSE 是 `0.972252`。
- `XVLA` clean chunk RMSE 是 `0.466172`。
- `XVLA` 的 beat repeat baseline rate 是 `0.5`，`SmolVLA` 是 `0.0`。

这说明在当前 LIBERO holdout 上，`XVLA` 的动作预测更接近专家轨迹。

但工程可用性上，`SmolVLA` 当前更稳：

- `SmolVLA` 完成了 30 次 soak，30 次全部成功。
- `XVLA` 在当前主机上进行长稳 soak 时被阻塞，相关进程会被系统杀掉。

所以当前不能简单说哪个模型全面胜出。更准确的结论是：`XVLA` 更准，`SmolVLA` 更稳。

## 6. 系统集成结果

系统集成方面，VLA 异步推理接口、OpenClaw 聚合评测接口、request_id trace 和结构化错误传播已经打通。

100 次交替 loopback 的功能成功率只有 1%，主要原因是 `XVLA` 请求拖挂了 VLA 服务。但任务状态正确性和 request_id trace 完整性都是 100%。这说明虽然功能成功率低，但失败路径是可观测、可追踪、可保护退化的。

这对机械臂系统很重要，因为失败被结构化捕获并进入保护动作，比失败后无响应或误执行更安全。

## 7. 当前建议

当前建议如下：

- `SmolVLA` 作为当前主机上的服务可用性基线。
- `XVLA` 作为离线精度基线和后续优化目标。
- 暂不把 `XVLA` 直接设为在线默认服务模型。
- 不把本轮 LIBERO RMSE 直接等同于实机任务成功率。
- 不绕过 SafetyGuard 直接把 VLA action chunk 转成 ARX X5 执行动作。

## 8. 下一步

下一步建议分三条线：

1. 稳定性：拆分复跑 `SmolVLA-only`、`XVLA-only`、`交替策略` 三组 soak。
2. 资源定位：记录 `XVLA` 被系统杀掉时的 RSS、XPU memory 和内核日志。
3. 真实工况：构建目标工位 replay 数据集，再接入 D435i live stream 和 ARX X5 实机安全链。

## 9. 结束语

本轮工作的价值在于已经把“模型精度问题”和“工程服务化问题”拆开了。`XVLA` 给出了更好的离线精度上限，`SmolVLA` 给出了当前主机上可继续推进的服务基线。下一阶段重点不是盲目上实机，而是先把服务长稳性、目标工况 replay 和安全放行链补齐。

