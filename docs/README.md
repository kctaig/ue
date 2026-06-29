# docs/ 文档索引

**本项目：** UnrealEngine 5.5.4 源码阅读笔记（Niagara GPU 侧）
**规范：** 见 [`documentation_guide.md`](./documentation_guide.md)
**新建文档起点：** [`doc_template.md`](./doc_template.md)

本文件是 `docs/` 目录的**单一事实来源**。新增文档必须在此登记。

---

## 文档目录

| 文档 | 主题 | 类型 |
|---|---|---|
| [niagara_system_tick_and_execution_flow.md](./niagara_system_tick_and_execution_flow.md) | Niagara 系统 Tick 与执行流程（WorldManager→SystemSimulation→Instance→Emitter） | flow |
| [static_mesh_location_gpu_spawn_flow.md](./static_mesh_location_gpu_spawn_flow.md) | Niagara GPU Static Mesh Location 粒子 Spawn 流程 | flow |
| [niagara_add_velocity_gpu_spawn_flow.md](./niagara_add_velocity_gpu_spawn_flow.md) | Niagara GPU Add Velocity（传统 VM）粒子 Spawn 流程 | flow |
| [niagara_static_mesh_di_internals.md](./niagara_static_mesh_di_internals.md) | Niagara Static Mesh DataInterface 类内部实现详解 | architecture |
| [niagara_gpu_sim_to_render_data_flow.md](./niagara_gpu_sim_to_render_data_flow.md) | Niagara GPU 模拟与渲染之间的数据处理逻辑 | flow |

---

## 规范文件

| 文件 | 用途 |
|---|---|
| [documentation_guide.md](./documentation_guide.md) | 文档编写规范（命名/骨架/图/引用/元数据） |
| [doc_template.md](./doc_template.md) | 新建文档可复制模板 |

---

## 说明

- 本目录源自从官方 UnrealEngine 5.5.4 源码做的静态分析，文档对应"关联 commit"标注的代码版本。
- 源码引用一律用「文件 · 符号」，不写行号（见规范 §6）。
- 流程/结构图一律 mermaid（见规范 §7）。
