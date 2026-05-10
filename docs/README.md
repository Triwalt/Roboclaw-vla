# vla docs

本目录存放 VLA 子模块的测评报告、bringup 交付材料、接口契约索引和后续 subsystem 设计记录。

## 当前文档

- `bringup/vla-evaluation-handoff-2026-05-10/`：给同事汇报与工作同步的 VLA 测评交付包。
- `research/RESEARCH-004-vla-model-evaluation-summary.md`：VLA 专项测评完整报告。
- `subsystem/vla_subsystem_blueprint.md`：VLA 子模块定位、边界、目录规划和后续迁移建议。

## 与父仓库的关系

当前 VLA 接口、契约和运行时代码仍在父仓库 `Roboclaw` 中。本仓库负责保存 VLA 领域材料和交付物，并通过索引指向父仓库实现位置。

后续当 VLA service 和模型 adapter 进入独立演进阶段，可逐步把稳定实现迁移到本仓库的 `src/` 与 `tests/`。

