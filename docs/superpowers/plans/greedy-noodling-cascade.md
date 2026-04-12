## Context
用户希望我直接修正文档中已确认的 6 处一致性问题，使 [docs/designs/模块设计文档.md](../../projects/go/agent-eval-server/docs/designs/模块设计文档.md) 与 [docs/designs/details/scheduler/](../../projects/go/agent-eval-server/docs/designs/details/scheduler/) 下详细设计文档在职责边界、事件流、状态语义、依赖关系、命名与接口契约上完全一致。当前文档中的主要问题包括：Dispatcher 职责描述过窄、执行器失联后的恢复链路未闭合、TaskException 语义与可恢复异常冲突、TaskManager 依赖图多出 StageStore、MasterStateStore 命名不统一，以及 StageSplitter 详细设计未体现批量 CreateStages。

目标是做一次纯文档修正，不改代码，只统一口径、流程图、事件表、伪代码和职责表，形成一套自洽的调度服务设计说明，避免后续实现时依据不同文档得出相互矛盾的结论。

## Recommended approach
按“先统一事件契约，再统一各模块详细设计，最后回收至模块设计总览”的顺序修改，避免顶层与细节来回打架。

### 1. 先统一事件契约
优先修改以下文件：
- `docs/designs/details/scheduler/事件总线设计.md`

修改要点：
- 新增 `SubTaskExceptionEvent` 事件定义、事件内容结构、发布者/订阅者说明
- 明确发布者为 `Dispatcher`，订阅者为 `DAGScheduler`
- 更新事件类型表、模块集成表、事件流图，形成显式链路：
  `ExecutorOfflineEvent -> Dispatcher -> SubTaskExceptionEvent -> DAGScheduler -> NodesReadyEvent -> QuotaAllocator -> AllocationEvent -> Dispatcher`

### 2. 修正 DAGScheduler 的恢复语义
修改以下文件：
- `docs/designs/details/scheduler/DAG调度器设计.md`

修改要点：
- 将 `SubTaskExceptionEvent` 加入订阅事件
- 增加 `handleSubTaskException` 逻辑描述与伪代码
- 将“异常子任务恢复”从“下一轮节点释放时发现”改为“收到异常事件后立即重新评估节点状态”
- 在事件交互表、流程图、职责边界中明确：
  - `Dispatcher` 只负责清理执行态并发布异常事件
  - `DAGScheduler` 负责重评估节点、决定是否重新释放节点
- 若节点仍可重试，则重新发布 `NodesReadyEvent`

### 3. 扩充 Dispatcher 职责边界
修改以下文件：
- `docs/designs/details/scheduler/子任务分发器设计.md`

修改要点：
- 将职责从“只负责分发”扩展为“负责分发 + 已分发子任务的执行态清理”
- 明确其处理两类补偿场景：
  - `ExecutorOfflineEvent`：清理 `Allocated/Running` 子任务并发布 `SubTaskExceptionEvent`
  - `TaskCancelEvent`：取消已分发子任务并清理执行绑定
- 保留“不负责 DAG 重试决策”的边界
- 更新职责边界、方法说明、事件交互表、流程图与相关伪代码

### 4. 收紧 TaskException 语义
修改以下文件：
- `docs/designs/details/scheduler/任务管理器设计.md`

修改要点：
- 将 `TaskException` 语义统一为“不可恢复的系统级异常”
- 删除或改写状态图中“执行服务失联 -> TaskException”的转移
- 补充说明：执行器失联优先映射为子任务异常，不直接改变任务终态
- 保持任务终态仍由 `TaskFinishedEvent / TaskFailedEvent` 驱动
- 若文中存在把 TaskManager 描述为直接处理阶段或执行器恢复的表述，一并收紧

### 5. 修正 StageSplitter 的批量创建契约
修改以下文件：
- `docs/designs/details/scheduler/阶段拆分器设计.md`

修改要点：
- 将 `createStages` 伪代码从逐条 `CreateStage` 改为组装 `[]*Stage` 后一次性 `CreateStages`
- 在方法表、事件处理步骤、协作说明中改为“批量创建阶段记录”
- 与模块设计文档中的 `StageStore` 接口定义保持一致

### 6. 统一主从存储命名
修改以下文件：
- `docs/designs/details/scheduler/主从模式设计.md`
- `docs/designs/模块设计文档.md`

修改要点：
- 统一使用 `MasterStateStore`
- 清理所有 `MasterStateStorage` 残留
- 确保职责表、依赖表、结构体字段和流程图名称一致

### 7. 最后统一模块设计总览
最后修改以下文件：
- `docs/designs/模块设计文档.md`

修改要点：
- 在模块职责表中扩充 `Dispatcher` 的职责描述
- 在模块依赖图中删除 `TaskManager --> StageStore`
- 在 EventBus 集成表中加入 `SubTaskExceptionEvent`
- 在执行服务失联处理流程中改为显式事件驱动恢复链路
- 调整对 `TaskException` 的语义描述，避免与可恢复异常冲突
- 将阶段拆分相关描述统一为“批量创建 stages 表记录”
- 统一 `MasterStateStore` 命名

## Critical files to modify
- `docs/designs/模块设计文档.md`
- `docs/designs/details/scheduler/事件总线设计.md`
- `docs/designs/details/scheduler/DAG调度器设计.md`
- `docs/designs/details/scheduler/子任务分发器设计.md`
- `docs/designs/details/scheduler/任务管理器设计.md`
- `docs/designs/details/scheduler/阶段拆分器设计.md`
- `docs/designs/details/scheduler/主从模式设计.md`

## Existing patterns / references to reuse
- 模块总览中的 EventBus 集成表与核心流程章节可作为最终统一出口：
  - `docs/designs/模块设计文档.md`
- 当前详细设计中已存在的异常/清理相关逻辑描述，可在文档上收敛而非重写架构：
  - `docs/designs/details/scheduler/子任务分发器设计.md`
  - `docs/designs/details/scheduler/DAG调度器设计.md`
  - `docs/designs/details/scheduler/任务管理器设计.md`
- StageStore 契约已在模块设计中定义，详细设计应向其对齐：
  - `docs/designs/模块设计文档.md`
  - `docs/designs/details/scheduler/阶段拆分器设计.md`

## Verification
完成文档修改后，逐项核对：

1. **事件一致性**
- `SubTaskExceptionEvent` 同时出现在：
  - 事件定义表
  - 事件内容结构
  - Dispatcher / DAGScheduler 的事件交互表
  - 模块设计总览的 EventBus 集成表与失联恢复流程

2. **职责一致性**
- Dispatcher 在所有文档中都被描述为：
  - 负责执行服务选择与分发
  - 负责执行态清理（offline/cancel）
  - 不负责 DAG 重试策略
- DAGScheduler 在所有文档中都被描述为：
  - 负责异常子任务触发后的节点重评估与重新释放

3. **状态语义一致性**
- 不再存在“执行服务失联直接导致 TaskException”的描述
- `TaskException` 只表示不可恢复系统异常

4. **依赖一致性**
- 模块依赖图中不存在 `TaskManager -> StageStore`
- TaskManager 的职责表和详细设计结构体不包含 StageStore

5. **命名一致性**
- 不再存在 `MasterStateStorage`
- 全部统一为 `MasterStateStore`

6. **接口契约一致性**
- `阶段拆分器设计.md` 明确使用 `CreateStages`
- 不再把逐条 `CreateStage` 当作主设计口径

7. **图文一致性**
- Mermaid 图、职责表、事件表、伪代码中的流程方向一致
- 特别复核：执行器失联恢复流程、模块依赖图、事件总线流转图
