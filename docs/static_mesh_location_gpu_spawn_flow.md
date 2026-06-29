# Niagara GPU Static Mesh Location 粒子 Spawn 流程

**日期:** 2026-06-24
**分支:** UE-5.5.4 源码阅读
**关联 commit:** 无（基于 UnrealEngine 5.5.4 官方源码静态分析，未做改动）
**作者:** yangxu.li

> 本文解析 Niagara GPU emitter 在 **粒子 Spawn 阶段** 如何用 `UNiagaraDataInterfaceStaticMesh` 在静态网格表面采样位置。只讲 GPU 侧，CPU/VectorVM 路径不涉及。读完能回答：spawn 这条数据从"CPU 决定生成几个"到"每个新粒子在世界空间拿到一个表面点"的完整链路。

---

## 0. 一句话概括

GPU spawn 不是一个独立 shader——它是每个 sim stage compute shader 里 `else if (bRunSpawnLogic)` 分支；`StaticMeshLocation` 节点图被 `FNiagaraHlslTranslator` 翻译成 `SimulateMapSpawn(Context)` 内的内联 DI 调用，新粒子线程在此调用 `RandomTriangle`/`GetTriangleWS` 完成面积加权采样、重心插值与世界变换。

---

## 1. 涉及文件 / 关键文件索引

| 文件 | 符号（函数/类/字段） | 职责 |
|---|---|---|
| `NiagaraGpuComputeDispatch.cpp` | `FNiagaraGpuComputeDispatch::PreInitViews()` | 渲染管线早期入口：先 `PrepareAllTicks` 规划整帧，再 `ExecuteTicks(PreInitViews)` |
| `NiagaraGpuComputeDispatch.cpp` | `FNiagaraGpuComputeDispatch::PrepareAllTicks()` | 帧首一次性规划所有 TickStage，逐 proxy 调 `PrepareTicksForProxy` |
| `NiagaraGpuComputeDispatch.cpp` | `FNiagaraGpuComputeDispatch::PrepareTicksForProxy()` | **规划**：遍历 emitter proxy，算 spawn 数、分组、建 `FNiagaraGpuDispatchInstance`，填 `DispatchListPerStage[stage]` |
| `NiagaraGpuComputeDispatch.cpp` | `FNiagaraGpuComputeDispatch::PostInitViews()` / `PostRenderOpaque()` / `PreRender()` | 后续渲染阶段入口：只调 `ExecuteTicks(对应 stage)`，不再 Prepare |
| `NiagaraGpuComputeDispatch.cpp` | `FNiagaraGpuComputeDispatch::ExecuteTicks()` | **执行**：按 TickStage 遍历 `DispatchListPerStage[stage]`，对每个 dispatch group 走 Pre/Dispatch/Post |
| `NiagaraGpuComputeDispatch.cpp` | `FNiagaraGpuComputeDispatch::DispatchStage()` | 计算 dispatch 维度、传 `NumSpawnedInstances` 等 uniform、下发 compute |
| `NiagaraGpuComputeDispatch.cpp` | `FNiagaraGpuComputeDispatch::PreStageInterface()` | 每 stage 执行前调各 DI proxy 的 `PreStage()`，绑定 buffer/uniform |
| `NiagaraDataInterfaceStaticMesh.cpp` | `FInstanceData_RenderThread::Init()` | DI GPU 数据上传：复用 `FStaticMeshLODResources` 的 RHI SRV，构建 alias 表 buffer |
| `NiagaraDataInterfaceStaticMesh.cpp` | `UNiagaraDataInterfaceStaticMesh::SetShaderParameters()` | DI GPU 参数绑定：把 SRV / 矩阵赋给 `ShaderParameters`，对应 `.ush` 模板的 uniform |
| `NiagaraHlslTranslator.cpp` | `FNiagaraHlslTranslator`（生成 `MainCS`） | 把节点图翻译成 HLSL，spawn/update 两分支拼在同一 `main` |
| `NiagaraGpuComputeDispatch.h` | `FNiagaraGpuDispatchGroup` / `FNiagaraGpuDispatchList` / `FNiagaraGpuDispatchInstance` | 规划产物：group 分组、stage 列表、单次 dispatch 数据（含 `SimStageData`） |
| `NiagaraEmitterInstanceShader.usf` | `SetupExecIndexAndSpawnInfoForGPU()` | 新粒子线程把 `GLinearThreadId` 翻译成"组内新粒子索引"+ spawn 时间 |
| `NiagaraEmitterInstanceShader.usf` | `NiagaraRandomBaryCoord()` | sqrt 技巧生成面积均匀的重心坐标 |
| `NiagaraDataInterfaceStaticMeshTemplate.ush` | `RandomTriangle_{P}()` / `RandomSection_{P}()` / `RandomSectionTriangle_{P}()` | Alias 法两级选三角 + 生成重心 |
| `NiagaraDataInterfaceStaticMeshTemplate.ush` | `UniformTriangle_{P}()` | 逐三角 Alias 加权（启用均匀采样时） |
| `NiagaraDataInterfaceStaticMeshTemplate.ush` | `GetTriangle_{P}()` / `GetTriangleWS_{P}()` | 重心插值得局部点 → 世界变换 → 帧差算速度 |
| `NiagaraDataInterfaceStaticMeshTemplate.ush` | `GetVertex_{P}()` / `GetVertexWS_{P}()` | 顶点级采样（Location 模块顶点模式的入口） |
| `NiagaraDataInterfaceStaticMesh.cpp` | `UNiagaraDataInterfaceStaticMesh::GetCommonHLSL()` | shader 头插入 `.ush` 模板 `#include` + 声明 DI buffer |
| `NiagaraDataInterfaceStaticMesh.cpp` | `UNiagaraDataInterfaceStaticMesh::GetFunctionHLSL()` | 为每个 DI 函数生成 `{InstanceFunctionName}` 适配 wrapper |
| `NiagaraDataInterfaceStaticMesh.cpp` | `UNiagaraDataInterfaceStaticMesh::GetVMExternalFunction()` | CPU 路径绑定（仅对照，本文不展开） |
| `NiagaraDataInterfaceStaticMesh.cpp` | `FNDISectionInfo` / `FNDISectionAreaWeightedSampler` | section 级 Alias 表数据（CPU 烘焙，GPU 查表） |

---

## 2. 背景 / 概念

### 2.0 引擎预制模块、脚本与外部数据源

理解本文需先厘清四个概念层，全部基于 UE 自带资产/类：

| 层 | 是什么 | 谁提供 | 含 mesh 数据吗 |
|---|---|---|---|
| `UNiagaraDataInterfaceStaticMesh`（C++ 类） | DI 类：暴露函数 + 持有数据上传逻辑 | UE 引擎 | 运行时从你指定的 mesh 取 |
| `StaticMeshLocation.uasset`（节点图模块） | 预连好的脚本，调 DI 函数 | UE 引擎（`Engine/Plugins/FX/Niagara/Content/Modules/Spawn/Location/`） | 否，纯调用 |
| 你的 emitter（`.uasset`） | 引用模块 + 给 DI 指定 mesh 参数 | 使用者 | 指定哪个 mesh |
| 你的 system（`.uasset`） | 包含 emitter | 使用者 | — |

**关键澄清**：`StaticMeshLocation.uasset` **不是代码、不是 C++、不是 shader**，而是一个节点图脚本资产——内容就是"调 `RandomTriangle` → 调 `GetTriangleWS` → 写 Position 属性"这串节点的预制版本。你在 emitter 的 Spawn 阶点点 "+" 选 "Location → Static Mesh Location" 即引用此模块。它也可双击打开查看内部节点。

**脚本** = `.uasset` 里的节点图（`UNiagaraScript` 的图表示），无论引擎预制还是使用者手连都是同一物。脚本里可显式调用的 DI 节点，背后就是 DI 暴露的函数。

**外部数据源 vs 内部数据**：界限是"数据来源在不在粒子属性系统内"。

| | 内部数据 | 外部数据（DI） |
|---|---|---|
| 数据在哪 | 粒子缓冲（RW 缓冲：Position/Velocity/Color/Age…） | DI 持有的 buffer/uniform/资产 |
| 谁管理 | Niagara VM/ComputeShader 属性系统直接读写 | DI 自身上传、刷新、销毁 |
| 访问方式 | `Context.Particles.Position = xxx` 直接读写 | 经 DI 函数调用，DI 内部读自己的 buffer |
| 读写性 | 读写 | 采样只读（mesh 顶点等） |

"外部"指**粒子属性系统外部**，非 UE 外部。以 `GetVertex` 为例：脚本调用返回的顶点位置来自 DI 的 `_PositionBuffer`（每帧从 `FStaticMeshLODResources` 上传的 SRV，粒子脚本从不可直接访问），而写回的 `Position` 是粒子缓冲属性（脚本任意行可读写）。

`StaticMeshLocation.uasset` 本身**不含任何 mesh 数据**——它是"无数据脚本"：模块（UE 预制）定义"怎么取"（算法流程），使用者在 emitter 里指定"取谁的"（mesh 引用、Source、空间参数）。这种"预制流程 + 运行期注入数据"的分离是 DI 体系的设计基础。

### 2.1 什么是 StaticMesh DI（Data Interface）

StaticMesh DI（`UNiagaraDataInterfaceStaticMesh`）是 Niagara 体系的**数据源插件**——在粒子脚本与外部数据之间做接口。它不是"给你一个 mesh"，而是**暴露一组 mesh 表面查询函数 + 持有这些函数的输入数据**。同类 DI 还有 `NiagaraDataInterfaceSkeletalMesh`、`NiagaraDataInterfaceTexture`、`NiagaraDataInterfaceColorCurve`——同一模式的不同实例。本文所述"StaticMesh Location"即指用 StaticMesh DI 在 mesh 表面采样的 emitter。

DI 同时扮演三个角色，下面逐一配真实代码说明。

#### 角色 1：函数定义者——告诉 Niagara"我提供哪些函数、签名是什么"

DI 通过 `GetFunctionsInternal()`（`NiagaraDataInterfaceStaticMesh.cpp`）把自己的函数注册成 `FNiagaraFunctionSignature`（函数名 + 输入输出类型）。这是粒子脚本能在节点图里搜到 `RandomTriangle`、`GetVertex` 等节点的根本原因——编辑器枚举所有 DI 的签名建节点菜单。

```cpp
// 讲解性摘录（不会入库，注释中文）：注册函数签名
void UNiagaraDataInterfaceStaticMesh::GetFunctionsInternal(TArray<FNiagaraFunctionSignature>& OutFunctions) const
{
    FNiagaraFunctionSignature BaseSignature;
    BaseSignature.Inputs.Emplace(FNiagaraTypeDefinition(GetClass()), TEXT("StaticMesh"));
    BaseSignature.bMemberFunction = true;          // 是成员函数，调用时需绑 DI 实例
    // ...
    GetVertexSamplingFunctions(OutFunctions, BaseSignature);
    GetTriangleSamplingFunctions(OutFunctions, BaseSignature);
    GetSocketSamplingFunctions(OutFunctions, BaseSignature);
    // ...
}

// 单个签名示例：GetVertex(int Vertex) -> (Position, Velocity, Normal, Bitangent, Tangent)
FNiagaraFunctionSignature Sig = BaseSignature;
Sig.Inputs.Emplace(FNiagaraTypeDefinition::GetIntDef(), TEXT("Vertex"));
Sig.Outputs.Emplace(FNiagaraTypeDefinition::GetPositionDef(), TEXT("Position"));
Sig.Outputs.Emplace(FNiagaraTypeDefinition::GetVec3Def(), TEXT("Velocity"));
Sig.Outputs.Emplace(FNiagaraTypeDefinition::GetVec3Def(), TEXT("Normal"));
// ...
OutFunctions.Add_GetRef(Sig).Name = GetVertexName;   // Name = "GetVertex"
```

注册只是声明"有这么个函数"。函数体在哪实现，取决于脚本编译目标——见角色 2。

#### 角色 2：算法实现者——GPU 与 CPU 各一套实现，同名同签名

**GPU 路径**：函数体在 `NiagaraDataInterfaceStaticMeshTemplate.ush` 的 HLSL 模板里。编译期 `GetFunctionHLSL()`（`NiagaraDataInterfaceStaticMesh.cpp`）把它文本拼接进 emitter 的 compute shader。例 `GetVertex`：

```hlsl
// 讲解性摘录：GPU 实现，读 _PositionBuffer 取顶点位置
void GetVertex_{P}(int Vertex, out float3 Position, out float3 Velocity,
                   out float3 Normal, out float3 Bitangent, out float3 Tangent)
{
    Vertex = clamp(Vertex, 0, {P}_NumVertices - 1);
    Position.x = {P}_PositionBuffer[Vertex * 3 + 0];   // packed float buffer
    Position.y = {P}_PositionBuffer[Vertex * 3 + 1];
    Position.z = {P}_PositionBuffer[Vertex * 3 + 2];
    // ... 从 _TangentBuffer 解出 TBN
}
```

`{P}`（ParameterName）是文本宏，编译期替换成 DI 实例符号（如 `Mesh`），故多个 DI 实例不重名（`GetVertex_Mesh` / `GetVertex_SecondMesh`）。

**CPU 路径**：函数体是 `.cpp` 里的 C++ lambda，由 `GetVMExternalFunction()` 按函数名绑定。例 `GetVertex` 的 CPU 实现 `VMGetVertex` 直接读 `FStaticMeshLODResources`：

```cpp
// 讲解性摘录：CPU 实现，直接读 CPU 侧渲染数据
// VMGetVertex 内部经 GetLocalTrianglePosition 之类访问：
Position  = LODResource->VertexBuffers.PositionVertexBuffer.VertexPosition(Index0) * BaryCoord.X;
Position += LODResource->VertexBuffers.PositionVertexBuffer.VertexPosition(Index1) * BaryCoord.Y;
Position += LODResource->VertexBuffers.PositionVertexBuffer.VertexPosition(Index2) * BaryCoord.Z;
```

**两条路径同算法、不同数据源**：GPU 读 SRV buffer，CPU 读 `FStaticMeshLODResources`。这是 DI 设计的核心复用——节点图一份，编译时分叉。

#### 角色 3：数据持有者——把 mesh 渲染数据搬成 DI 的可消费形态

DI 不直接持有 `UStaticMesh`，而是把所需的渲染数据缓存到 `FInstanceData_GameThread`，再上传成 SRV/uniform。两个阶段均位于 `NiagaraDataInterfaceStaticMesh.cpp`：

- **缓存 LOD 资源**：`FStaticMeshCpuHelper::GetCurrentLOD()` / `GetCurrentLODWithVertexColorOverrides()`——从 `UStaticMesh->GetRenderData()->LODResources[CachedLODIdx]` 取 `FStaticMeshLODResources`，缓存到 `FInstanceData_GameThread`。
- **上传成 SRV**：`FInstanceData_RenderThread::Init()`——把缓存的 LOD 资源搬成 GPU SRV（即下方代码摘录），核心是 `GetSRV()` 复用 mesh 渲染数据的 RHI SRV，不拷贝顶点数据。

```cpp
// 讲解性摘录：InitRD 把渲染数据搬成 GPU 可消费的 SRV
if (GpuInitializeData.LODResource)
{
    // 索引/顶点/切线/UV/颜色 buffer —— 直接复用 mesh 渲染数据的 RHI SRV
    MeshPositionBufferSRV = GpuInitializeData.LODResource->VertexBuffers.PositionVertexBuffer.GetSRV();
    MeshTangentBufferSRV  = GpuInitializeData.LODResource->VertexBuffers.StaticMeshVertexBuffer.GetTangentsSRV();
    MeshUVBufferSRV       = GpuInitializeData.LODResource->VertexBuffers.StaticMeshVertexBuffer.GetTexCoordsSRV();
    // 面积加权采样表（alias table），CPU 烘焙的资产数据
    MeshUniformSamplingTriangleSRV = GpuInitializeData.bGpuUniformDistribution
        ? GpuInitializeData.LODResource->AreaWeightedSectionSamplersBuffer.GetBufferSRV() : nullptr;
    // 计数
    NumTriangles.X = GpuInitializeData.LODResource->IndexBuffer.GetNumIndices() / 3;
    NumVertices    = GpuInitializeData.LODResource->VertexBuffers.PositionVertexBuffer.GetNumVertices();
}
```

这些 SRV 就是角色 2 GPU 函数里读的 `_PositionBuffer` / `_TangentBuffer` 等。section 级 alias 表 (`_SectionInfos`) 由 `FNDISectionAreaWeightedSampler::Init()` 从 `FStaticMeshSectionAreaWeightedTriangleSamplerArray` 构建，存 `(FirstTriangle, NumTriangles, Prob, Alias)`。

本图说明：粒子脚本通过 DI 函数访问 mesh 数据，DI 同时提供算法与数据。

```mermaid
flowchart LR
    A["Niagara 粒子脚本<br/>调用取位置"] -->|调用函数| B["StaticMesh DI<br/>RandomTriangle / GetTriangleWS"]
    B --> C["_PositionBuffer 顶点位置"]
    B --> D["_IndexBuffer 三角索引"]
    B --> E["_SectionInfos 别名表"]
    B --> F["_InstanceTransform 世界矩阵"]
```

三角色协作闭环：角色 1 注册签名 → 节点图能调用 → 脚本编译时按目标选角色 2 的 GPU 或 CPU 实现 → 实现读取角色 3 上传的数据 → 粒子拿到结果。

- **Spawn 阶段 = `bRunSpawnLogic` 分支**：Niagara 的 spawn 与 update 不分两个 dispatch，而是同一个 compute shader 内的两个布尔 uniform 分支。dispatch 维度 = `DestinationNumInstances`（老粒子 N + 新粒子 M）；老粒子线程走 `bRunUpdateLogic`，新粒子线程走 `bRunSpawnLogic`。
- **StaticMesh DI 的 GPU 端是函数库模板**：见 §2.0，`NiagaraDataInterfaceStaticMeshTemplate.ush` 带 `{ParameterName}` 宏占位，编译期被文本拼接进每个 emitter 的 compute shader。
- **采样与取值分离**：`RandomTriangle` 只返回 `(Tri, BaryCoord)`，`GetTriangleWS` 再据此取位置。同一采样点可重复取 Position/Normal/UV/Color，随机数可复现。
- **面积加权 = 两级 Alias Method**：section 级 + triangle 级，O(1) 查表，避免大三角形被低估。
- **速度由帧差反推**：不存速度缓冲，`(当前帧世界位置 − 上一帧世界位置) × InvDeltaSeconds`。

---

## 3. 数据流 / 流程图

本图说明：一帧内 GPU 仿真的调用链——Prepare（规划）只在 `PreInitViews` 发生一次，Execute（执行）按渲染阶段分散；图里标出每个函数所在文件，便于定位。`DispatchListPerStage` 是规划产物、各阶段共享的数据桥梁。

```mermaid
flowchart TD
    subgraph PreInitViews["PreInitViews（渲染早期，规划+执行一次）"]
        A["PreInitViews()<br/>NiagaraGpuComputeDispatch.cpp"] --> B["PrepareAllTicks()<br/>NiagaraGpuComputeDispatch.cpp"]
        B --> C["PrepareTicksForProxy()<br/>NiagaraGpuComputeDispatch.cpp<br/>★ 规划：算 spawn 数/分组/建 DispatchInstance"]
        C -->|填充| BR["DispatchListPerStage[各 stage]<br/>FNiagaraGpuDispatchList / Group / Instance<br/>NiagaraGpuComputeDispatch.h"]
        A --> D1["ExecuteTicks(PreInitViews)<br/>NiagaraGpuComputeDispatch.cpp"]
    end

    subgraph Later["后续渲染阶段（只执行，不规划）"]
        D2["PostInitViews → ExecuteTicks(PostInitViews)"]
        D3["PostRenderOpaque → ExecuteTicks(PostOpaqueRender)"]
        D4["PreRender → ExecuteTicks(...)"]
    end

    BR -.各阶段共享.-> D1
    BR -.各阶段共享.-> D2
    BR -.各阶段共享.-> D3
    BR -.各阶段共享.-> D4

    D1 --> E["ExecuteTicks 内：<br/>Pre → DispatchStage → Post<br/>NiagaraGpuComputeDispatch.cpp"]
    D2 --> E
    D3 --> E
    D4 --> E
```

本图说明：`ExecuteTicks` 内部对每个 dispatch group 的 Pre/Dispatch/Post 三段，及其中 DI 的参与点。

```mermaid
flowchart TD
    S["ExecuteTicks(stage)<br/>NiagaraGpuComputeDispatch.cpp"] --> G["遍历 DispatchListPerStage[stage] 的每个 DispatchGroup"]
    G --> P1["Pre Stage<br/>PreStageInterface() → DI.PreStage()<br/>NiagaraDataInterfaceStaticMesh.cpp 上传 mesh SRV/矩阵"]
    P1 --> P2["Buffer 状态转换 SRV→UAV"]
    P2 --> DS["DispatchStage()<br/>算 DispatchCount=N+M, 下发 NumSpawnedInstances=M"]
    DS --> GPU["GPU compute MainCS<br/>spawn/update 同 shader 两分支<br/>NiagaraEmitterInstanceShader.usf + .ush 模板"]
    GPU --> P3["Post Stage<br/>PostStageInterface() → DI.PostStage()<br/>Buffer 转回 SRV, SetDataToRender"]
    P3 --> G
```

本图说明：从 CPU 决定 spawn 数量，到 GPU 每个新粒子线程在 mesh 表面拿到世界位置，逐级数据流向。

```mermaid
flowchart TD
    A["CPU: PrepareTicksForProxy()<br/>算 DestinationNumInstances = N + M"] --> B["CPU: PreStageInterface()<br/>DI.PreStage 上传 mesh buffers + 实例矩阵"]
    B --> C["CPU: DispatchStage()<br/>DispatchCount = DestinationNumInstances（直接复用）<br/>NumSpawnedInstances = M = Dest − Src"]
    C --> D["GPU dispatch compute<br/>线程 GLinearThreadId ∈ 0..N+M-1"]
    D --> E{"bRunUpdateLogic?"}
    E -- 是 --> F["走 update 分支<br/>(老粒子, 略)"]
    E -- 否 --> G{"bRunSpawnLogic?"}
    G -- 是 --> H["SetupExecIndexAndSpawnInfoForGPU()<br/>GGPUExecIndex = ThreadId - GSpawnStartInstance"]
    H --> I["SimulateMapSpawn(Context)<br/>= 节点图翻译出的 spawn 脚本"]
    I --> J["RandomTriangle_Mesh()<br/>Alias 选 section → Alias 选三角 → sqrt 重心"]
    J --> K["GetTriangleWS_Mesh()<br/>读 buffer → 重心插值 → 实例矩阵 → 帧差速度"]
    K --> L["Context.Particles.Position = Pos<br/>Velocity = Vel"]
    L --> M["WriteDataSets()<br/>写回粒子存储缓冲"]
```

本图说明：spawn 线程内部时序——索引翻译、spawn 脚本执行、属性写回。

```mermaid
sequenceDiagram
    participant CPU as FNiagaraGpuComputeDispatch
    participant Shader as MainCS (新粒子线程)
    participant DI as StaticMesh DI 模板函数

    CPU->>Shader: dispatch, NumSpawnedInstances=M, GSpawnStartInstance=N
    Shader->>Shader: SetupExecIndexAndSpawnInfoForGPU() 算组内索引+spawn时间
    Shader->>Shader: InitSpawnVariables() 写属性默认值
    Shader->>DI: RandomTriangle_Mesh(RandInfo)
    DI-->>Shader: (Tri, BaryCoord)
    Shader->>DI: GetTriangleWS_Mesh(Tri, BaryCoord)
    DI->>DI: 读 IndexBuffer/PositionBuffer/SectionInfos
    DI->>DI: 重心插值得 LocalPos
    DI->>DI: mul(LocalPos, InstanceTransform) → WS
    DI-->>Shader: Position, Velocity, Normal, ...
    Shader->>Shader: WriteDataSets() 写回缓冲
```

---

## 3.1 Prepare（规划）与 Execute（执行）的分离

`PrepareTicksForProxy` 与 `ExecuteTicks` 是一帧 GPU 仿真里的"画施工图"与"按图施工"。理解这点就懂了为什么涉及这么多文件、它们怎么串起来。

**规划只发生一次，固定在 `PreInitViews`**：`PreInitViews()` 先调 `PrepareAllTicks()`，后者遍历每个 TickStage 的每个 `ComputeProxy` 调 `PrepareTicksForProxy()`。`PrepareTicksForProxy` 内部一个 `for (iTickStage=0..Max)` 循环把**整帧所有 stage 的 dispatch 列表都填好**，存进 `DispatchListPerStage[stage]`（结构定义在 `NiagaraGpuComputeDispatch.h` 的 `FNiagaraGpuDispatchList` / `FNiagaraGpuDispatchGroup` / `FNiagaraGpuDispatchInstance`）。

**执行按渲染阶段分散**：`PreInitViews` 紧接着 `ExecuteTicks(PreInitViews)`，后续 `PostInitViews` / `PostRenderOpaque` / `PreRender` 各调一次 `ExecuteTicks(对应 stage)`，但**都不再 Prepare**，直接消费帧首规划好的 `DispatchListPerStage[stage]`。`ExecuteTicks` 开头 `if (!DispatchList.HasWork()) return`——某阶段没活就跳过。

**`PrepareTicksForProxy` 具体填了什么**（这些都决定了 §4 的 spawn 计算）：为每个 emitter proxy 的每个 sim stage iteration 建一个 `FNiagaraGpuDispatchInstance`，其 `SimStageData` 含：

- `bFirstStage`——是否 spawn stage（spawn 数在此判定）
- `Source` / `Destination`——本 stage 读/写的 `FNiagaraDataBuffer`
- `StageMetaData`——迭代源、是否写粒子等
- `DispatchArgs.ElementCount`——dispatch 维度雏形

也就是说，`DispatchStage` 里 `InstancesToSpawn = DestinationNumInstances − SourceNumInstances`、`DispatchCount = DestinationNumInstances` 这些值的**源头就在 `PrepareTicksForProxy`**，`ExecuteTicks` 只是取出来用。

**为什么这样分**：① 规划不依赖场景渲染状态，帧首算一次即可；执行要对的渲染时机（collision 需 GBuffer 要等 PostOpaque）。② spawn 数等规划值一帧内不变，集中算避免重复。③ 规划用直接 RHI 命令分配 buffer，执行包成 RDG pass 让框架管依赖。

---

## 4. 逐项详解（以及"为什么"）

### 4.1 CPU 决定 spawn 数量

`FNiagaraGpuComputeDispatch::PrepareTicksForProxy()` 每帧算：

```
CurrentNumInstances = PrevNumInstances + SpawnRateInstances + EventSpawnTotal
```

填入 `FNiagaraSimStageData`。对 spawn stage（`bFirstStage == true`），在 `DispatchStage()` 里：

```cpp
// 讲解性摘录（不会入库）
if (SimStageData.bFirstStage)
{
    InstancesToSpawn = SimStageData.DestinationNumInstances - SimStageData.SourceNumInstances;
}
SimStageData.Destination->SetNumSpawnedInstances(InstancesToSpawn);
```

- `SourceNumInstances` = 老粒子 N
- `DestinationNumInstances` = N + M
- `InstancesToSpawn` = M，作为 uniform `NumSpawnedInstances` 下发 GPU

**为什么 dispatch 维度是 N+M 而不是 M**：粒子缓冲连续存储，新粒子紧接老粒子。合并到一个 dispatch，老线程走 update、新线程走 spawn，省一次 buffer 读写与一次 dispatch 开销。

### 4.2 DI 数据绑定（PreStageInterface）

StaticMesh DI 的 GPU 绑定是两段式，均位于 `NiagaraDataInterfaceStaticMesh.cpp`：

**第 1 段：上传 SRV** —— `FInstanceData_RenderThread::Init()`。把 mesh 渲染数据搬成 GPU SRV，核心是复用 `FStaticMeshLODResources` 现成的 RHI SRV，不拷贝顶点数据：

- 顶点/索引/切线/UV buffer → SRV：`GetSRV()` 取 `PositionVertexBuffer`、`StaticMeshVertexBuffer`（Tangents/TexCoords）、`IndexBuffer`
- section 级 alias 表 → PF_R32G32B32A32_UINT buffer（`SectionInfos.Initialize`），逐三角 alias 表取 `AreaWeightedSectionSamplersBuffer.GetSRV()`
- 计数 `NumTriangles`、`NumVertices`

**第 2 段：绑参数** —— `UNiagaraDataInterfaceStaticMesh::SetShaderParameters()`。把第 1 段上传好的 SRV 赋给 `ShaderParameters`，对应 `.ush` 模板声明的 uniform：

- `_PositionBuffer` / `_IndexBuffer` / `_TangentBuffer` / `_UVBuffer` / `_ColorBuffer` ← 各 SRV
- `_UniformSamplingTriangles` / `_SectionInfos` / `_FilteredAndUnfilteredSections` ← alias 表/section SRV
- `_InstanceTransform` / `_InstanceTransformInverseTransposed` / `_InstancePreviousTransform` / `_InstancePreviousTransformInverseTransposed` ← 当前/上一帧矩阵
- `_InvDeltaSeconds` ← `1/DeltaSeconds`
- `_InstanceRotation`、`_InstancePreviousRotation`、socket 数据等

**调度入口**：`FNiagaraGpuComputeDispatch::PreStageInterface()`（`NiagaraGpuComputeDispatch.cpp`）遍历每个 DI proxy 调 `PreStage()`，StaticMesh DI 走基类通用流程后触发第 2 段绑定。

调用关系：`PreStageInterface` → 各 DI 的 `PreStage()` → StaticMesh DI 经 `SetShaderParameters()` 绑定 `Init()` 上传好的 SRV。

**为什么每帧刷矩阵**：mesh 运动时新粒子必须采样当前帧世界位置；上一帧矩阵与当前帧配对才能算出表面速度，粒子继承。`SetShaderParameters` 里 `InstanceTransform` 来自当前帧 `Transform`，`InstancePreviousTransform` 来自 `PrevTransform`。

### 4.3 GPU 线程入口——spawn/update 同体

`FNiagaraHlslTranslator` 为每个 emitter 动态生成 `MainCS`，spawn/update 两分支同函数（摘录自生成逻辑）：

```cpp
// 讲解性示例：注释可中文
BRANCH
if (bRunUpdateLogic) {            // 老粒子
    SetupExecIndexForGPU();
    InitConstants(Context);
    LoadUpdateVariables(Context, GLinearThreadId);
    ReadDataSets(Context);
    Simulate{Stage}(Context);
    WriteDataSets(Context);
}
else if (bRunSpawnLogic) {        // 新粒子 ← SPAWN
    SetupExecIndexAndSpawnInfoForGPU();
    InitConstants(Context);
    InitSpawnVariables(Context);
    ReadDataSets(Context);
    Context.MapSpawn.Particles.UniqueID =
        Engine_Emitter_TotalSpawnedParticles + GLinearThreadId - GSpawnStartInstance;
    ConditionalInterpolateParameters(Context);
    SimulateMapSpawn(Context);    // ★ 节点图 spawn 脚本在此
    TransferAttributes(Context);
    WriteDataSets(Context);
}
```

**`SimulateMapSpawn(Context)` 就是节点图**：用户在编辑器连的 `RandomTriangle → GetTriangleWS → Position 写回` 被翻译成内联 HLSL 调用嵌在此函数里。

### 4.4 新粒子索引翻译

`SetupExecIndexAndSpawnInfoForGPU()`：

```hlsl
// 讲解性示例：注释可中文
GGPUExecIndex = GLinearThreadId - GSpawnStartInstance;  // 0..M-1 = "我是第几个新粒子"
// 遍历 EmitterSpawnInfoOffsets 找当前 spawn group
Emitter_SpawnInterval       = EmitterSpawnInfoParams[Idx].x;
Emitter_InterpSpawnStartDt  = EmitterSpawnInfoParams[Idx].y;
Emitter_SpawnGroup          = asint(EmitterSpawnInfoParams[Idx].z);
GroupSpawnStartIndex        = asint(EmitterSpawnInfoParams[Idx].w);
GGPUExecIndex               = GGPUExecIndex - GroupSpawnStartIndex;
```

- `GSpawnStartInstance` = N（老粒子数），uniform 下发
- `NIAGARA_MAX_GPU_SPAWN_INFOS` 是编译期常数，支持多个 spawn group（不同 burst）；遍历找组，组数很小，O(组数)
- spawn 脚本通过 `ExecIndex()` 拿到的是组内索引，对脚本透明

### 4.5 面积加权选三角——两级 Alias Method

`RandomTriangle_{P}()` = `RandomSection_{P}()` + `RandomSectionTriangle_{P}()`。

**第 1 级：选 section（Alias 法，O(1)）**

```hlsl
// 讲解性示例
Section = NiagaraRandomInt(RandInfo, SectionCount);
float Prob = asfloat({P}_SectionInfos[Section].z);  // .z = Prob
int   Alias = {P}_SectionInfos[Section].w;          // .w = Alias
Section = NiagaraRandomFloat(RandInfo) < Prob ? Section : Alias;
```

`_SectionInfos` 是 `uint4`：`(FirstTriangle, NumTriangles, Prob, Alias)`。CPU 在 `FNDISectionAreaWeightedSampler` 烘焙好 Prob/Alias（继承 `FWeightedRandomSampler`），GPU 只查表 + 比较。大 section 按面积比例被选中。

**第 2 级：section 内选三角**

```hlsl
// 讲解性示例
int SectionTriangleOffset = {P}_SectionInfos[Section].x;   // FirstTriangle
Tri = NiagaraRandomInt(RandInfo, SectionTriangleCount) + SectionTriangleOffset;
[branch]
if ({P}_HasUniformSampling)        // uniform 常量驱动, 真实统一分支, 无 warp 发散
{
    Tri = UniformTriangle_{P}(RandInfo, Tri, SectionTriangleOffset);
}
```

`UniformTriangle_{P}()` 对**单个三角形**再做一次 Alias（`_UniformSamplingTriangles`，`uint2(Prob, Alias)`）。未烘焙采样表的资产走快路径跳过。

**为什么两级 alias**：把"按面积加权"压成 O(1)，数千粒子并发采样不退化。`[branch]` + uniform 常量保证无 warp 发散。

### 4.6 重心坐标——sqrt 技巧

`NiagaraRandomBaryCoord()`：

```hlsl
// 讲解性示例
float2 r = float2(NiagaraRandomFloat(RandInfo), NiagaraRandomFloat(RandInfo));
float sqrt0 = sqrt(r.x);
return float3(1.0 - sqrt0, sqrt0 * (1.0 - r.y), r.y * sqrt0);
```

直接映射重心 `(1-u-v, u, v)` 会向顶点聚集；`sqrt(r.x)` 撑开分布得面积均匀采样。三分量和恒为 1 且 ≥ 0。

### 4.7 取位置与世界变换

`GetTriangle_{P}()`（局部）→ `GetTriangleWS_{P}()`（世界）：

```hlsl
// 讲解性示例
// GetTriangle: 读三顶点, 重心插值
int Index[3]; GetTriangleIndices_{P}(Tri, Index[0], Index[1], Index[2]);
float3 Positions[3]; GetVertex_{P}(Index[0..2], Positions[0..2], ...);
Position = Positions[0]*Bary.x + Positions[1]*Bary.y + Positions[2]*Bary.z;

// GetTriangleWS: 世界变换 + 速度
Position          = mul(float4(LocalPos, 1), {P}_InstanceTransform).xyz;
float3 PrevPos    = mul(float4(LocalPos, 1), {P}_InstancePreviousTransform).xyz;
Velocity          = (Position - PrevPos) * {P}_InvDeltaSeconds;
Normal            = normalize(mul(float4(Normal,   0), {P}_InstanceTransformInverseTransposed).xyz);
```

**三个工程要点**：
1. 位置用正向矩阵 `w=1`，法线/切线用逆转置矩阵 `w=0`——非均匀缩放下法线仍垂直表面。
2. 速度 = 帧差 × InvDeltaSeconds；`GetTriangle_` 内部 Velocity 清零，真实速度在 WS 版由帧差算。
3. LWC 大世界下位置需叠加 `_SystemLWCTile` 处理精度。

### 4.8 节点图 → HLSL 的桥接（编译期）

- `GetCommonHLSL()`：shader 头插 `.ush` 模板 `#include`，声明所有 DI buffer/uniform。
- `GetFunctionHLSL()`：为每个 DI 调用生成 `{InstanceFunctionName}` wrapper，包住模板函数并补 `MakeRandInfo()` 等细节：

```cpp
// 讲解性摘录
TEXT("void {InstanceFunctionName}(out MeshTriCoordinate OutTriCoord) "
     "{ RandomTriangle_{ParameterName}(MakeRandInfo(), OutTriCoord.Tri, OutTriCoord.BaryCoord); }")
```

`{ParameterName}` = DI 实例符号（如 `Mesh`），`{InstanceFunctionName}` = 节点唯一符号。同一 shader 多个 DI 实例靠宏实例化不重名（`GetTriangleWS_Mesh`、`GetTriangleWS_SecondMesh`）。

---

## 5. 工程化要点小结

| 设计 | 原因 |
|---|---|
| spawn/update 同 shader 两分支 | 粒子缓冲连续存储，合并 dispatch 省 buffer 读写与开销 |
| `.ush` 模板 + 文本宏拼接 | compute shader 无跨 shader 函数链接，只能文本嵌入复用算法 |
| 两级 Alias Method | 面积加权选三角压成 O(1)，GPU 并发友好 |
| `[branch]` + uniform 常量 | `HasUniformSampling` 是 dispatch 级常量，统一分支无发散 |
| 采样与取值分离 | 同一采样点可重复取多属性，随机可复现 |
| 双变换矩阵算速度 | 不存速度缓冲，mesh 运动时粒子自动继承表面速度 |
| 法线用逆转置矩阵 | 非均匀缩放下保持几何正确 |
| sqrt 重心坐标 | 三角形内面积均匀，避免顶点聚集 |

---

## 6. 已知问题

`GetTriangle_{P}()` 中 Normal/Bitangent/Tangent 的第三顶点权重写作 `*BaryCoord.x`（应为 `*BaryCoord.z`）。Position 那行正确。三分量和为 1，故 TBN 插值略偏但不发散。精确复现 TBN 时需按 `z` 修正。

---

## N. 验证 / 落地清单

本文为源码阅读类文档（无改动），验证清单用于确认文档描述仍与源码一致：

- [ ] `RandomTriangle_{P}` 仍为 `RandomSection_{P}` + `RandomSectionTriangle_{P}` 组合
- [ ] `DispatchStage` 仍以 `DestinationNumInstances` 为 1D dispatch 维度
- [ ] `bFirstStage` 仍用于判定 spawn（`InstancesToSpawn = Dest − Src`）
- [ ] `SetupExecIndexAndSpawnInfoForGPU` 仍由 `FNiagaraHlslTranslator` 生成于 spawn 分支
- [ ] `GetCommonHLSL` 仍 `#include` `NiagaraDataInterfaceStaticMeshTemplate.ush`

---

## 附录

### 术语表

| 术语 | 含义 |
|---|---|
| Spawn stage | `bFirstStage == true` 的 sim stage，负责生成新粒子 |
| Alias Method (Vose) | 加权随机采样 O(1) 算法，用 Prob/Alias 两表 |
| BaryCoord | 重心坐标 `(x, y, z)`，和为 1，定位三角形内一点 |
| DI | Data Interface，Niagara 暴露给脚本的外部数据源 |
| ISM | Instanced Static Mesh，实例化静态网格 |
| LWC | Large World Coordinates，大世界坐标精度方案 |

### 自检命令

```bash
# 校验 spawn 分支生成逻辑仍存在
grep -n "SetupExecIndexAndSpawnInfoForGPU" Engine/Plugins/FX/Niagara/Source/NiagaraEditor/Private/NiagaraHlslTranslator.cpp

# 校验 spawn 数量计算
grep -n "InstancesToSpawn = SimStageData.DestinationNumInstances - SimStageData.SourceNumInstances" \
  Engine/Plugins/FX/Niagara/Source/Niagara/Private/NiagaraGpuComputeDispatch.cpp

# 校验两级 Alias 入口
grep -n "RandomSection_\|RandomSectionTriangle_\|UniformTriangle_" \
  Engine/Plugins/FX/Niagara/Shaders/Private/NiagaraDataInterfaceStaticMeshTemplate.ush

# 校验 DI shader 桥接
grep -n "GetCommonHLSL\|GetFunctionHLSL" \
  Engine/Plugins/FX/Niagara/Source/Niagara/Private/DataInterface/NiagaraDataInterfaceStaticMesh.cpp

# 校验已知问题（TBN 第三顶点权重）
grep -n "BaryCoord.x + Positions\[1\]\*BaryCoord.y\|Bitangents\[2\]\*BaryCoord.x" \
  Engine/Plugins/FX/Niagara/Shaders/Private/NiagaraDataInterfaceStaticMeshTemplate.ush
```
