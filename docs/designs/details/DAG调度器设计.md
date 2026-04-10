# DAG调度器设计

## 一、概述

管理所有执行中任务的 DAG 图调度，根据事件驱动执行依赖检查、节点释放、状态聚合。

**职责边界**：
- 管理多个任务的 TaskDAG（内存结构）
- 只关注 DAG 调度逻辑，不关注子任务执行细节
- 通过 EventBus 接收事件，处理调度决策
- 任务完成后清理 TaskDAG

---

## 二、结构与数据模型

### 2.1 TaskDAG（内存层双向图）

TaskDAG 采用双向图设计，实现 O(1) 查询上下游关系。

```go
type TaskDAG struct {
    taskId     string

    // 双向图结构
    parents    map[string][]string          // nodeId → 父节点（依赖谁）
    children   map[string][]string          // nodeId → 子节点（谁依赖我）

    // 节点状态
    nodes      map[string]*DAGNodeState     // nodeId → 状态

    // 根节点列表
    roots      []string                     // 无依赖的节点
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

| 操作 | 存储层（单向） | 内存层（双向） |
|------|----------------|----------------|
| 查询上游依赖 | 遍历 Dependencies | `parents[nodeId]` O(1) |
| 查询下游子节点 | 扫描全表 | `children[nodeId]` O(1) |

### 2.2 DAGScheduler

```go
type DAGScheduler struct {
    storage        DAGNodeStorage
    subTaskStorage SubTaskStorage
    eventBus       EventBus

    taskDAGs       map[string]*TaskDAG      // taskId → TaskDAG
    events         chan *Event              // 事件队列（串行处理）
    logger         *zap.Logger
    stopCh         chan struct{}
}
```

**设计要点**：
- `taskDAGs` 按 taskId 组织多任务并发调度
- 事件串行处理，无需额外锁
- 任务完成后 `delete(taskDAGs, taskId)` 清理

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
    case TaskFinishedEvent, TaskFailedEvent:
        s.handleTaskComplete(event)
    }
}
```

### 3.2 LoadTaskDAG（构建 TaskDAG）

```go
func (s *DAGScheduler) LoadTaskDAG(taskId string) (*TaskDAG, error) {
    // 1. 从 Storage 加载 DAGNode
    dagNodes := s.storage.GetDAGNodesByTask(taskId)

    // 2. 构建双向图
    taskDAG := &TaskDAG{
        taskId:   taskId,
        parents:  make(map[string][]string),
        children: make(map[string][]string),
        nodes:    make(map[string]*DAGNodeState),
        roots:    make([]string, 0),
    }

    for _, node := range dagNodes {
        taskDAG.nodes[node.NodeId] = &DAGNodeState{
            NodeId: node.NodeId, NodeType: node.NodeType,
            Status: Blocked, SubTaskIds: node.SubTaskIds,
        }

        taskDAG.parents[node.NodeId] = node.Dependencies
        for _, parentId := range node.Dependencies {
            taskDAG.children[parentId] = append(taskDAG.children[parentId], node.NodeId)
        }

        if len(node.Dependencies) == 0 {
            taskDAG.roots = append(taskDAG.roots, node.NodeId)
        }
    }

    s.taskDAGs[taskId] = taskDAG
    return taskDAG, nil
}
```

### 3.3 handleDAGBuildComplete

```go
func (s *DAGScheduler) handleDAGBuildComplete(event *Event) {
    content := event.Content.(*DAGBuildCompleteEventContent)
    taskDAG := s.LoadTaskDAG(content.TaskId)
    for _, nodeId := range taskDAG.roots {
        s.releaseNode(taskDAG, nodeId)
    }
}
```

### 3.4 handleDAGNodeFinished

```go
func (s *DAGScheduler) handleDAGNodeFinished(event *Event) {
    content := event.Content.(*DAGNodeFinishedEventContent)
    taskDAG := s.taskDAGs[content.TaskId]
    nodeState := taskDAG.nodes[content.NodeId]
    nodeState.Status = Finished

    // 查找子节点（双向图 O(1)）
    for _, childNodeId := range taskDAG.children[content.NodeId] {
        if s.checkDependenciesMet(taskDAG, childNodeId) {
            s.releaseNode(taskDAG, childNodeId)
        }
    }
}
```

### 3.5 checkDependenciesMet

```go
func (s *DAGScheduler) checkDependenciesMet(taskDAG *TaskDAG, nodeId string) bool {
    for _, parentId := range taskDAG.parents[nodeId] {
        if taskDAG.nodes[parentId].Status != Finished {
            return false
        }
    }
    return true
}
```

### 3.6 releaseNode

```go
func (s *DAGScheduler) releaseNode(taskDAG *TaskDAG, nodeId string) error {
    nodeState := taskDAG.nodes[nodeId]
    nodeState.Status = Ready
    s.storage.UpdateDAGNodeStatus(nodeId, int(Ready))

    // 发布 SubTaskReadyEvent，由 QuotaAllocator 处理
    for _, subTaskId := range nodeState.SubTaskIds {
        subTask := s.subTaskStorage.GetSubTask(subTaskId)
        s.eventBus.Publish(&Event{
            Type: SubTaskReadyEvent,
            Content: &SubTaskReadyEventContent{
                TaskId:    taskDAG.taskId,
                SubTaskId: subTaskId,
                ModelId:   subTask.Extra["model_id"],
            },
        })
    }

    taskDAG.nodes[nodeId].Status = Running
    s.storage.UpdateDAGNodeStatus(nodeId, int(Running))
    return nil
}
```

### 3.7 handleSubTaskFinished

```go
func (s *DAGScheduler) handleSubTaskFinished(event *Event) {
    content := event.Content.(*SubTaskFinishedEventContent)
    taskDAG := s.taskDAGs[content.TaskId]
    nodeId := s.findNodeBySubTaskId(taskDAG, content.SubTaskId)

    nodeStatus := s.aggregateNodeStatus(taskDAG, nodeId)
    if nodeStatus == Finished || nodeStatus == Failed {
        taskDAG.nodes[nodeId].Status = nodeStatus
        s.storage.UpdateDAGNodeStatus(nodeId, int(nodeStatus))
        s.eventBus.Publish(&Event{
            Type: DAGNodeFinishedEvent,
            Content: &DAGNodeFinishedEventContent{
                TaskId: content.TaskId,
                NodeId: nodeId,
                Status: int(nodeStatus),
            },
        })
    }
}
```

### 3.8 aggregateNodeStatus

```go
func (s *DAGScheduler) aggregateNodeStatus(taskDAG *TaskDAG, nodeId string) DAGNodeStatus {
    allFinished := true
    hasFailed := false
    for _, subTaskId := range taskDAG.nodes[nodeId].SubTaskIds {
        subTask := s.subTaskStorage.GetSubTask(subTaskId)
        if subTask.Status == Failed { hasFailed = true }
        if subTask.Status != Finished && subTask.Status != Failed { allFinished = false }
    }
    if hasFailed { return Failed }
    if allFinished { return Finished }
    return Running
}
```

### 3.9 handleTaskComplete

```go
func (s *DAGScheduler) handleTaskComplete(event *Event) {
    content := event.Content.(*TaskStatusEventContent)
    delete(s.taskDAGs, content.TaskId)
}
```

---

## 四、扩展与集成

### 4.1 事件处理总览

| 事件 | 处理流程 |
|------|----------|
| DAGBuildCompleteEvent | LoadTaskDAG → 释放根节点 |
| DAGNodeFinishedEvent | 更新状态 → 查找 children → 检查依赖 → releaseNode |
| SubTaskFinishedEvent | findNode → aggregate → publish DAGNodeFinishedEvent |
| TaskFinishedEvent/TaskFailedEvent | delete(taskDAGs) |

### 4.2 性能优化建议

`findNodeBySubTaskId` 当前为 O(n×m)，可增加反向映射优化为 O(1)：

```go
type TaskDAG struct {
    ...
    subTaskToNode map[string]string  // subTaskId → nodeId
}
```

---

## 五、相关文档

- [DAG构建器设计](DAG构建器设计.md) - DAGNode 存储层结构
- [事件总线设计](事件总线设计.md) - 事件类型定义
- [数据模型设计文档](../数据模型设计文档.md) - dag_nodes 表