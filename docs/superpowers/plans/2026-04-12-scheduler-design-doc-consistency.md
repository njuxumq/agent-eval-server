# 调度设计文档一致性修正方案

## 背景
用户希望我直接修改调度服务相关设计文档，使顶层模块设计文档与调度服务详细设计文档在逻辑上保持一致。当前文档在若干关键规则上存在冲突，包括：`stage_id` 到底是 `int` 还是 `string`、`TaskManager` 还是 `StageSplitter` 负责创建阶段记录、谁发布 `StageCompletedEvent` / `StageFailedEvent`、阶段作用域事件是否始终携带 `task_id + stage_id`、`inference/eval` 子任务数量到底是 `N` 还是 `M × N`、执行器失联后子任务如何回到可调度状态等。这些冲突都不是措辞问题，而是会直接影响实现落地的设计分歧。

## 推荐修正方向
采用一套统一的调度设计规则，并将所有受影响文档修正到同一口径。建议采用以下统一规则：
1. `stage_id` 为任务内有序 `int`，从 1 开始。
2. 阶段作用域事件统一携带 `task_id + stage_id`。
3. `TaskManager` 负责创建 `tasks` 和 `task_configs`；`StageSplitter` 负责创建 `stages`。
4. `DAGScheduler` 负责发布 `StageCompletedEvent` 和 `StageFailedEvent`；`StageTransitioner` 负责消费这些事件并决定下一阶段或任务终态。
5. 当前设计中每个阶段对应一个 `StageDAGNode`；未来多节点扩展要明确标注为“未来扩展”，不能与当前实现混写。
6. 对应用评测场景，`inference` 与 `eval` 阶段子任务数量为 `M × N`。
7. `Dispatcher` 不发布完成类事件。
8. 执行器失联后的处理流程必须明确说明子任务如何回到可调度状态，不能只写“等待重新调度”而不说明谁来触发。

## 需要修改的关键文件
- `docs/designs/模块设计文档.md`
- `docs/designs/details/scheduler/事件总线设计.md`
- `docs/designs/details/scheduler/任务管理器设计.md`
- `docs/designs/details/scheduler/阶段拆分器设计.md`
- `docs/designs/details/scheduler/阶段转换器设计.md`
- `docs/designs/details/scheduler/DAG构建器设计.md`
- `docs/designs/details/scheduler/DAG调度器设计.md`
- `docs/designs/details/scheduler/配额分配器设计.md`
- `docs/designs/details/scheduler/子任务分发器设计.md`
- `docs/designs/details/scheduler/执行服务管理器设计.md`
- `docs/designs/details/scheduler/主从模式设计.md`

## 需要保留并统一的现有设计锚点
- 事件载荷定义表目前以 `docs/designs/details/scheduler/事件总线设计.md:75-175` 为中心，但其它文档中的发布/消费示例没有完全遵循它。
- 阶段创建职责目前在 `docs/designs/模块设计文档.md:124-126` 与 `docs/designs/details/scheduler/阶段拆分器设计.md:4-10` 之间冲突。
- 阶段终态事件发布职责目前在 `docs/designs/模块设计文档.md:257-260` 与 `docs/designs/details/scheduler/事件总线设计.md:228-229` 之间冲突。
- `M × N` 子任务语义目前写在 `docs/designs/模块设计文档.md:640-658`，但被 `docs/designs/details/scheduler/DAG构建器设计.md:255-275` 中的示例否定。

## 具体修正计划

### 1. 修正 `docs/designs/模块设计文档.md`
- 更新调度服务职责表：去掉 `TaskManager` 的“阶段预创建”职责，补上 `DAGScheduler` 发布阶段完成/失败事件的职责。
- 重写任务初始化流程：明确 `TaskManager` 只创建 `tasks` 和 `task_configs`，然后由 `StageSplitter` 创建 `stages`。
- 重写阶段执行流程和状态聚合流程：明确 `SubTaskFinishedEvent` 由 `TaskManager` 在子任务状态落库后发布，然后由 `DAGScheduler` 聚合节点与阶段状态，最后由 `StageTransitioner` 决定下一阶段或任务终态。
- 更新跨模块接口契约示例，使阶段相关接口与详细设计采用同一种 `stage_id` 标识方案。
- 保持应用评测场景 `M × N` 子任务数量定义不变。
- 明确执行器失联后的子任务恢复路径，而不是只写“等待重新调度”。

### 2. 修正 `docs/designs/details/scheduler/事件总线设计.md`
- 将其收敛为唯一的事件契约事实来源。
- 保持所有阶段作用域事件载荷统一为 `TaskId string + StageId int`（适用时）。
- 统一发布者/订阅者关系：
  - `TaskManager`：`TaskCreatedEvent`、`TaskCancelEvent`、`SubTaskFinishedEvent`
  - `StageSplitter`：`StageSplitCompleteEvent`
  - `StageTransitioner`：`StageStartedEvent`、`TaskFinishedEvent`、`TaskFailedEvent`
  - `DAGBuilder`：`DAGBuildCompleteEvent`
  - `DAGScheduler`：`NodesReadyEvent`、`DAGNodeFinishedEvent`、`StageCompletedEvent`、`StageFailedEvent`
  - `QuotaAllocator`：`AllocationEvent`
  - `Dispatcher`：不发布业务完成事件
  - `ExecutorManager`：`ExecutorOfflineEvent`
- 删除重复或残缺的尾部内容，保证文档本身结构干净。

### 3. 修正 `任务管理器设计.md`
- 保持 `TaskManager` 的职责边界聚焦在任务/子任务持久化、任务状态变更以及落库后的事件发布。
- 保留 `UpdateSubTaskStatus` 作为 `SubTaskFinishedEvent` 的发布位置。
- 移除或弱化“TaskManager 负责阶段统计或阶段终态决策”的表述，除非文档中真的定义了相关逻辑。
- 调整状态来源表，使其与事件驱动职责分工一致。

### 4. 修正 `阶段拆分器设计.md`
- 明确 `StageSplitter` 是 `stages` 记录的创建者。
- 明确 `stage_id` 是任务内从 1 开始递增的 `int`。
- 保证此文档不再暗示自己负责子任务或 DAG 节点创建。

### 5. 修正 `阶段转换器设计.md`
- 保持 `StageTransitioner` 负责启动首个非跳过阶段以及阶段间转换。
- 明确它是 `StageCompletedEvent` / `StageFailedEvent` 的消费者，而不是发布者。
- 如果文中存在依赖 `stageId == 0` 的哨兵式表达，用更清晰的“是否找到下一阶段”叙述替代。

### 6. 修正 `DAG构建器设计.md`
- 将所有 `stageId string` 的接口与示例修正为 `stageId int`，除非是在局部拼接存储键时显式转字符串。
- 统一使用 `StageDAGNode` / `StageDAGNodeStore` 这一命名，不再混用 `DAGNode`。
- 确保 `DAGBuildCompleteEvent` 发布示例补齐 `TaskId`。
- 修正内置策略和示例，使应用评测的 `inference` / `eval` 阶段体现 `M × N`，而不是只体现 `N`。
- 明确当前实现是“每阶段一个 `StageDAGNode`”，`Dependencies` 只是未来扩展预留。

### 7. 修正 `DAG调度器设计.md`
- 修正 `LoadStageDAG(taskId, stageId int)` 与事件处理示例中只传 `StageId` 的不一致问题。
- 保留 `taskId-stageId` 仅作为内部缓存 key，不要让它替代公开事件契约。
- 补齐 `NodesReadyEvent` 发布示例中的 `TaskId`。
- 将阶段清理逻辑正确区分 `StageCompletedEvent` 与 `StageFailedEvent`，避免把两种事件都按同一个内容结构强转。
- 在流程描述中明确阶段完成/失败事件由 `DAGScheduler` 发布。

### 8. 修正 `配额分配器设计.md`
- 将本地结构中的 `StageId` 统一为 `int`。
- 让 `AllocationEvent` 的示例补齐 `TaskId`。
- 让本地 `Allocation` 结构与事件总线契约保持一致；既然执行器选择属于 `Dispatcher`，则 `Allocation` 只应携带 `SubTaskId`。
- 将 `SubTaskFinishedEvent` 的来源统一描述为 `TaskManager`。

### 9. 修正 `子任务分发器设计.md`
- 删除任何暗示 `Dispatcher` 会发布 `SubTaskFinishedEvent` 的表述。
- 保持 `Dispatcher` 的职责只包括执行器选择和分发。
- 让执行器失联后的恢复策略与全局统一规则保持一致。
- 避免重复展开 `ExecutorManager` 的内部实现，只保留 Dispatcher 需要依赖的部分。

### 10. 修正 `执行服务管理器设计.md`
- 保持 `ExecutorOfflineEvent` 的发布语义与 Dispatcher 的消费预期一致。
- 将该模块定位为执行服务注册 / 健康检查 / 负载选择的控制组件，而不是业务事件聚合链的一部分。

### 11. 修正 `主从模式设计.md`
- 明确 `MasterSlave` 只负责调度器生命周期门控，不参与业务事件流。
- 将它与 `DAGScheduler` 的关系表述为 start/stop 或 enable/disable，而不是业务依赖链的一环。

## 修改后的验证方式
1. 通读 `docs/designs/模块设计文档.md`，确认：
   - 任务初始化流程中 `TaskManager` 只创建任务和配置；
   - 阶段创建归属 `StageSplitter`；
   - 阶段完成/失败事件归属 `DAGScheduler`；
   - 应用评测 `inference/eval` 子任务数量仍是 `M × N`。
2. 通读 `docs/designs/details/scheduler/事件总线设计.md`，确认所有阶段作用域事件都使用统一的 `TaskId + StageId(int)` 契约。
3. 在 `docs/designs/details/scheduler/*.md` 中检查并消除以下残留不一致：
   - `StageId string`
   - `Build(stageId string`
   - `Dispatcher` 发布 `SubTaskFinishedEvent`
4. 重新串读受影响的详细设计文档，确认整条链路前后一致：
   - `TaskCreatedEvent` → `StageSplitCompleteEvent` → `StageStartedEvent` → `DAGBuildCompleteEvent` → `NodesReadyEvent` → `AllocationEvent` → 分发执行 → `SubTaskFinishedEvent` → `DAGNodeFinishedEvent` → `StageCompletedEvent/StageFailedEvent` → `TaskFinishedEvent/TaskFailedEvent`。
