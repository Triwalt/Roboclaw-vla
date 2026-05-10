# VLA 子模块蓝图

> **版本**: v1.0.0 | **状态**: 草稿 | **日期**: 2026-05-11

## 目标

`vla` 子模块负责沉淀 Vision-Language-Action 模型相关的评测、交付、数据和后续模型接入能力。它和父仓库 `Roboclaw` 的关系应保持清晰：

- 父仓库负责系统集成、服务编排、安全决策和当前运行时代码。
- `vla` 子模块负责 VLA 领域材料、模型评测、模型 adapter 演进和交付同步。

## 当前边界

当前只迁入文档和交付物，不迁移运行时代码。原因是父仓库中 `src/openclaw` 仍是统一 Python 包，直接迁出会影响现有导入路径、测试和服务启动。

当前父仓库仍拥有：

- `src/openclaw/contracts.py`
- `src/openclaw/communication/vla_backend.py`
- `src/openclaw/communication/vla_service.py`
- `src/openclaw/communication/vla_service_client.py`
- `src/openclaw/controller/vla_task_controller.py`
- `src/openclaw/tools/vla_inference_worker.py`
- `src/openclaw/tools/system_eval.py`

## 建议目录规划

```text
vla/
  README.md
  docs/
    README.md
    bringup/
      vla-evaluation-handoff-YYYY-MM-DD/
    research/
      RESEARCH-xxx-*.md
    subsystem/
      vla_subsystem_blueprint.md
  src/
    openclaw_vla/
      adapters/
      benchmarks/
      datasets/
  tests/
    unit/
    integration/
```

`src/` 和 `tests/` 目前暂不创建，待契约稳定后再迁入。

## 迁移原则

1. 先迁文档、报告、benchmark 说明和模型评测材料。
2. 再迁纯离线工具，例如数据集 manifest 生成、结果汇总、图表生成。
3. 最后迁运行时代码，例如 VLA adapter 和 service 实现。
4. 契约稳定前，不复制 `contracts.py`，避免父仓库和子模块 schema 分叉。

## 当前交付入口

- `docs/bringup/vla-evaluation-handoff-2026-05-10/README.md`
- `docs/research/RESEARCH-004-vla-model-evaluation-summary.md`

## 后续任务

1. 将目标工位 replay 数据集规范写入 `docs/research/` 或 `docs/bringup/`。
2. 为 `SmolVLA-only`、`XVLA-only`、`交替策略` 三类 soak 结果建立统一归档格式。
3. 在父仓库接口稳定后，评估建立 `openclaw-vla` Python 包。

