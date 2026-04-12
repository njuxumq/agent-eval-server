# DAG调度器设计

## 一、概述

管理阶段内 DAG 图的调度执行，根据事件驱动执行依赖检查、节点释放、状态聚合。

**职责边界**：
- 管理多个阶段的 StageDAG（内存结构）
- 只关注 DAG 调度逻辑，不关注子任务执行细节
- 通过 EventBus 接收事件，处理调度决策
- 负责在子任务进入可恢复异常态后重新评估节点，并决定是否重新释放节点
- 阶段完成后清理 StageDAG
- 发布批量节点就绪事件（NodesReadyEvent）

---

## 二、结构与数据模型

### 2.1 StageDAG（内存层双向图）

StageDAG 采用双向图设计，实现 O(1) 查询上下游关系。

```go
type StageDAG struct {
    taskId       string                      // 所属任务ID（用于按任务查询和清理）
    stageId      int                         // 所属阶段编号（从1开始，任务内递增）

    // 双向图结构
    parents      map[string][]string         // nodeId → 父节点（依赖谁）
    children     map[string][]string         // nodeId → 子节点（谁依赖我）

    // 节点状态
    nodes        map[string]*DAGNodeState    // nodeId → 状态

    // 根节点列表（无依赖的节点）
    roots        []string

    // 反向映射（性能优化）
    subTaskToNode map[string]string          // subTaskId → nodeId，O(1) 查询
}

type DAGNodeState struct {
    NodeId      string
    NodeType    string
    Status      DAGNodeStatus
    SubTaskIds  []string
    LastUpdated time.Time
}

type DAGNodeStatus int

const (
    Blocked   = 0  // 等待依赖完成
    Ready     = 1  // 依赖满足，可释放
    Running   = 2  // 执行中
    Finished  = 3  // 完成
    Failed    = 4  // 失败
)
```

**双向图优势**：

| 操作           | 存储层（单向）    | 内存层（双向）          |
| -------------- | ----------------- | ----------------------- |
| 查询上游依赖   | 遍历 Dependencies | `parents[nodeId]` O(1)  |
| 查询下游子节点 | 扫描全表          | `children[nodeId]` O(1) |

### 2.2 DAGScheduler

```go
type DAGScheduler struct {
    storage        DAGNodeStorage
    subTaskStorage SubTaskStorage
    eventBus       EventBus

    stageDAGs      map[string]*StageDAG      // key: "taskId-stageId"
    events         chan *Event               // 事件队列（串行处理）
    logger         *zap.Logger
    stopCh         chan struct{}
}
```

**设计要点**：
- `stageDAGs` 的 key 为组合格式 `taskId-stageId`（如 `task-abc-1`）
- stageId 为 Int 类型（1, 2, 3...），从1开始递增
- 事件串行处理，无需额外锁

---

## 三、核心逻辑

### 3.1 事件处理循环

```go
func (s *DAGScheduler) Start() {
    for {
        select {
        case event := <-s.events:
            s.handleEvent(event)
        case <-s.stopCh:
            return
        }
    }
}

func (s *DAGScheduler) handleEvent(event *Event) {
    switch event.Type {
    case DAGBuildCompleteEvent:
        s.handleDAGBuildComplete(event)
    case DAGNodeFinishedEvent:
        s.handleDAGNodeFinished(event)
    case SubTaskFinishedEvent:
        s.handleSubTaskFinished(event)
    case SubTaskExceptionEvent:
        s.handleSubTaskException(event)
    case StageCompletedEvent, StageFailedEvent:
        s.handleStageComplete(event)
    }
}
```

### 3.2 LoadStageDAG（构建 StageDAG）

```go
func (s *DAGScheduler) LoadStageDAG(taskId string, stageId int) (*StageDAG, error) {
    // 1. 从 Storage 加载 DAGNode
    dagNodes := s.storage.GetDAGNodesByStage(taskId, stageId)

    // 2. 构建双向图
    compositeKey := fmt.Sprintf("%s-%d", taskId, stageId)
    stageDAG := &StageDAG{
        taskId:        taskId,
        stageId:       stageId,
        parents:       make(map[string][]string),
        children:      make(map[string][]string),
        nodes:         make(map[string]*DAGNodeState),
        roots:         make([]string, 0),
        subTaskToNode: make(map[string]string),  // 反向映射
    }

    for _, node := range dagNodes {
        stageDAG.nodes[node.NodeId] = &DAGNodeState{
            NodeId: node.NodeId, NodeType: node.NodeType,
            Status: Blocked, SubTaskIds: node.SubTaskIds,
        }

        // 构建反向映射（性能优化）
        for _, subTaskId := range node.SubTaskIds {
            stageDAG.subTaskToNode[subTaskId] = node.NodeId
        }

        stageDAG.parents[node.NodeId] = node.Dependencies
        for _, parentId := range node.Dependencies {
            stageDAG.children[parentId] = append(stageDAG.children[parentId], node.NodeId)
        }

        if len(node.Dependencies) == 0 {
            stageDAG.roots = append(stageDAG.roots, node.NodeId)
        }
    }

    // 3. 存入内存
    s.stageDAGs[compositeKey] = stageDAG

    return stageDAG, nil
}
```

---

### 3.3 handleDAGBuildComplete

```go
func (s *DAGScheduler) handleDAGBuildComplete(event *Event) {
    content := event.Content.(*DAGBuildCompleteEventContent)
    taskId := content.TaskId
    stageId := content.StageId
    stageDAG := s.LoadStageDAG(taskId, stageId)

    // 收集所有根节点的子任务ID
    readySubTaskIds := []string{}
    for _, nodeId := range stageDAG.roots {
        nodeState := stageDAG.nodes[nodeId]
        nodeState.Status = Ready
        s.storage.UpdateDAGNodeStatus(nodeId, int(Ready))
        readySubTaskIds = append(readySubTaskIds, nodeState.SubTaskIds...)
    }

    // 批量发布 NodesReadyEvent
    if len(readySubTaskIds) > 0 {
        s.eventBus.Publish(&Event{
            Type: NodesReadyEvent,
            Content: &NodesReadyEventContent{
                TaskId:     taskId,
                StageId:    stageId,
                SubTaskIds: readySubTaskIds,
            },
        })
    }
}
```

### 3.4 handleDAGNodeFinished

```go
func (s *DAGScheduler) handleDAGNodeFinished(event *Event) {
    content := event.Content.(*DAGNodeFinishedEventContent)
    compositeKey := fmt.Sprintf("%s-%d", content.TaskId, content.StageId)
    stageDAG := s.stageDAGs[compositeKey]
    nodeState := stageDAG.nodes[content.NodeId]
    nodeState.Status = Finished

    // 查找子节点（双向图 O(1)）
    readySubTaskIds := []string{}
    for _, childNodeId := range stageDAG.children[content.NodeId] {
        if s.checkDependenciesMet(stageDAG, childNodeId) {
            childState := stageDAG.nodes[childNodeId]
            childState.Status = Ready
            s.storage.UpdateDAGNodeStatus(childNodeId, int(Ready))
            readySubTaskIds = append(readySubTaskIds, childState.SubTaskIds...)
        }
    }

    // 批量发布 NodesReadyEvent
    if len(readySubTaskIds) > 0 {
        s.eventBus.Publish(&Event{
            Type: NodesReadyEvent,
            Content: &NodesReadyEventContent{
                TaskId:     content.TaskId,
                StageId:    content.StageId,
                SubTaskIds: readySubTaskIds,
            },
        })
    }
}
```

### 3.5 checkDependenciesMet

```go
func (s *DAGScheduler) checkDependenciesMet(stageDAG *StageDAG, nodeId string) bool {
    for _, parentId := range stageDAG.parents[nodeId] {
        if stageDAG.nodes[parentId].Status != Finished {
            return false
        }
    }
    return true
}
```

### 3.6 handleSubTaskFinished

```go
func (s *DAGScheduler) handleSubTaskFinished(event *Event) {
    content := event.Content.(*SubTaskFinishedEventContent)
    compositeKey := fmt.Sprintf("%s-%d", content.TaskId, content.StageId)
    stageDAG := s.stageDAGs[compositeKey]
    nodeId := s.findNodeBySubTaskId(stageDAG, content.SubTaskId)

    nodeStatus := s.aggregateNodeStatus(stageDAG, nodeId)
    if nodeStatus == Finished || nodeStatus == Failed {
        stageDAG.nodes[nodeId].Status = nodeStatus
        s.storage.UpdateDAGNodeStatus(nodeId, int(nodeStatus))
        s.eventBus.Publish(&Event{
            Type: DAGNodeFinishedEvent,
            Content: &DAGNodeFinishedEventContent{
                TaskId:  content.TaskId,
                StageId: content.StageId,
                NodeId:  nodeId,
                Status:  int(nodeStatus),
            },
        })

        // 阶段完成判定：所有节点都完成
        if s.checkStageComplete(stageDAG) {
            s.eventBus.Publish(&Event{
                Type: StageCompletedEvent,
                Content: &StageCompletedEventContent{
                    TaskId:  content.TaskId,
                    StageId: content.StageId,
                },
            })
        }

        // 阶段失败判定：任一节点失败且无法重试
        if nodeStatus == Failed && !s.hasRetryChance(stageDAG, nodeId) {
            failedCount := s.countFailedNodes(stageDAG)
            s.eventBus.Publish(&Event{
                Type: StageFailedEvent,
                Content: &StageFailedEventContent{
                    TaskId:      content.TaskId,
                    StageId:     content.StageId,
                    FailedCount: failedCount,
                },
            })
        }
    }
}
```

### 3.7 aggregateNodeStatus

```go
func (s *DAGScheduler) aggregateNodeStatus(stageDAG *StageDAG, nodeId string) DAGNodeStatus {
    allFinished := true
    hasFailed := false
    for _, subTaskId := range stageDAG.nodes[nodeId].SubTaskIds {
        subTask := s.subTaskStorage.GetSubTask(subTaskId)
        if subTask.Status == Failed { hasFailed = true }
        if subTask.Status != Finished && subTask.Status != Failed { allFinished = false }
    }
    if hasFailed { return Failed }
    if allFinished { return Finished }
    return Running
}
```

### 3.7 handleSubTaskException

```go
func (s *DAGScheduler) handleSubTaskException(event *Event) {
    content := event.Content.(*SubTaskExceptionEventContent)
    compositeKey := fmt.Sprintf("%s-%d", content.TaskId, content.StageId)
    stageDAG := s.stageDAGs[compositeKey]
    nodeId := s.findNodeBySubTaskId(stageDAG, content.SubTaskId)

    // 重新评估异常子任务所属节点
    if s.canRetryNode(stageDAG, nodeId) {
        stageDAG.nodes[nodeId].Status = Ready
        s.storage.UpdateDAGNodeStatus(nodeId, int(Ready))

        s.eventBus.Publish(&Event{
            Type: NodesReadyEvent,
            Content: &NodesReadyEventContent{
                TaskId:     content.TaskId,
                StageId:    content.StageId,
                SubTaskIds: stageDAG.nodes[nodeId].SubTaskIds,
            },
        })
    }
}
```

### 3.8 handleStageComplete

```go
func (s *DAGScheduler) handleStageComplete(event *Event) {
    content := event.Content.(*StageCompletedEventContent)
    compositeKey := fmt.Sprintf("%s-%d", content.TaskId, content.StageId)
    delete(s.stageDAGs, compositeKey)
}

// ClearTaskStageDAGs 清理指定任务的所有 StageDAG（用于任务取消场景）
func (s *DAGScheduler) ClearTaskStageDAGs(taskId string) {
    prefix := taskId + "-"
    for key := range s.stageDAGs {
        if strings.HasPrefix(key, prefix) {
            delete(s.stageDAGs, key)
        }
    }
}
```

---

## 四、扩展与集成

### 4.1 批量发布机制

调度器批量发布 NodesReadyEvent，包含所有就绪子任务，QuotaAllocator 可一次性处理多个子任务的资源分配。

### 4.2 子任务查找优化

`findNodeBySubTaskId` 通过反向映射 `subTaskToNode` 实现 O(1) 查询：

```go
func (s *DAGScheduler) findNodeBySubTaskId(stageDAG *StageDAG, subTaskId string) string {
    return stageDAG.subTaskToNode[subTaskId]  // O(1)
}
```

反向映射在 `LoadStageDAG` 时构建，每个子任务完成时调用一次，高频场景下性能显著提升。

### 4.3 阶段内 DAG 扩展

当前阶段内子任务并行执行，无内部依赖。为后续扩展性考虑：

- 支持定义节点间依赖关系（如部分子任务需等待特定前置条件）
- DAGNode.Dependencies 字段预留用于阶段内依赖

### 4.4 事件交互详情

#### 接收事件

| 事件类型 | 事件来源 | 触发时机 | 处理流程 |
|---------|---------|---------|---------|
| DAGBuildCompleteEvent | DAGBuilder | 阶段DAG构建完成，子任务和节点已持久化 | 1. LoadStageDAG构建双向图<br/>2. 标记根节点状态为Ready<br/>3. 收集根节点子任务ID<br/>4. 批量发布NodesReadyEvent |
| DAGNodeFinishedEvent | DAGScheduler（自身发布） | 节点完成，检查下游节点 | 1. 更新节点状态<br/>2. 查找children节点<br/>3. 检查依赖是否满足<br/>4. 满足依赖的节点→Ready→批量发布NodesReadyEvent |
| SubTaskFinishedEvent | TaskManager | 子任务状态已落库，TaskManager发布事件 | 1. 通过subTaskToNode找到所属节点<br/>2. aggregateNodeStatus聚合状态<br/>3. 若节点完成/失败→发布DAGNodeFinishedEvent<br/>4. 若阶段完成/失败→发布StageCompletedEvent/StageFailedEvent |
| SubTaskExceptionEvent | Dispatcher | 已分发子任务因执行服务失联等原因进入可恢复异常态 | 1. 通过subTaskToNode找到所属节点<br/>2. 重新评估节点是否仍可重试<br/>3. 若可重试→节点置为Ready<br/>4. 重新发布NodesReadyEvent |
| StageCompletedEvent | DAGScheduler（自身发布） | 阶段完成，任务进入下一阶段 | 清理内存中的StageDAG（delete stageDAGs[key]） |
| StageFailedEvent | DAGScheduler（自身发布） | 阶段失败，任务进入失败态 | 清理内存中的StageDAG（delete stageDAGs[key]） |

#### 发出事件

| 事件类型 | 发布时机 | 目标模块 | 触发动作 |
|---------|---------|---------|---------|
| NodesReadyEvent | 根节点释放、下游节点依赖满足或异常节点重新释放时 | QuotaAllocator | 批量配额门控判断 |
| DAGNodeFinishedEvent | 节点状态聚合为Finished或Failed | DAGScheduler（自身） | 继续释放下游节点 |
| StageCompletedEvent | 阶段内所有节点完成 | StageTransitioner | 触发下一阶段转换 |
| StageFailedEvent | 阶段内任一节点失败且无法重试 | StageTransitioner | 触发任务失败处理 |

#### 事件处理流程图

```mermaid
flowchart TB
    subgraph 接收事件处理
        DBE[DAGBuildCompleteEvent] --> LSD[LoadStageDAG<br/>构建双向图]
        LSD --> MRN[标记根节点=Ready]
        MRN --> CSN[收集根节点子任务ID]
        CSN --> PNRE1[Publish NodesReadyEvent]

        DNFE[DAGNodeFinishedEvent] --> USN[更新节点状态]
        USN --> FC[查找children节点]
        FC --> CD[检查依赖是否满足]
        CD -->|"满足"| MN2[标记节点=Ready]
        MN2 --> PNRE2[Publish NodesReadyEvent]

        STFE[SubTaskFinishedEvent] --> FNB[findNodeBySubTaskId<br/>O(1)反向映射]
        FNB --> ANS[aggregateNodeStatus<br/>聚合节点状态]
        ANS -->|"完成/失败"| PDNFE[Publish DAGNodeFinishedEvent]

        STE[SubTaskExceptionEvent] --> FNB2[findNodeBySubTaskId<br/>定位异常节点]
        FNB2 --> RNE[重新评估节点是否可重试]
        RNE -->|"可重试"| RR[节点置为Ready]
        RR --> PNRE3[Publish NodesReadyEvent]

        SCE[StageCompletedEvent<br/>StageFailedEvent] --> CLS[清理StageDAG<br/>delete stageDAGs]
    end

    subgraph 发出事件
        PNRE1 --> QA[QuotaAllocator<br/>配额门控]
        PNRE2 --> QA
        PNRE3 --> QA
        PDNFE --> DNFE
    end

    DB[DAGBuilder] --> DBE
    TM[TaskManager] --> STFE
```

---

## 五、相关文档

- [DAG构建器设计](DAG构建器设计.md) - DAGNode 存储层结构
- [事件总线设计](事件总线设计.md) - NodesReadyEvent 定义
- [数据模型设计文档](../../数据模型设计文档.md) - dag_nodes 表
- [模块设计文档](../../模块设计文档.md) - 增量式拆分架构