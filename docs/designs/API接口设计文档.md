# API接口设计文档

## 一、概述

### 1.1 文档目的

本文档定义应用评测服务（agent-eval-server）的所有API接口，包括：
- 接口定义与请求/响应格式
- 参数校验规则
- 错误码定义

### 1.2 设计原则

| 原则 | 说明 |
|------|------|
| **统一格式** | 所有接口使用统一的响应格式 `{code, message, data}` |
| **语义化路径** | 路径命名遵循 RESTful 规范 |
| **明确校验** | 所有参数有明确的校验规则和错误提示 |
| **版本管理** | 对外接口使用 `/v1/` 路径前缀，便于版本迭代 |
| **内部隔离** | 内部接口使用 `/internal/` 路径前缀，仅供服务间调用 |

### 1.3 接口分类

| 类别 | 路径前缀 | 说明 | 调用方 |
|------|----------|------|--------|
| **对外API** | `/v1/` | 用户调用的业务接口 | API调用方、前端 |
| **内部API** | `/internal/` | 服务间调用的管理接口 | 调度服务、执行服务 |

### 1.4 基础信息

| 项目 | 说明 |
|------|------|
| 协议 | HTTP/1.1 |
| 数据格式 | JSON |
| 字符编码 | UTF-8 |
| 响应格式 | `{code: int, message: string, data: object}` |
| 时间格式 | Unix时间戳（秒级） |

---

## 二、统一响应格式

### 2.1 成功响应

```json
{
    "code": 0,
    "message": "ok",
    "data": {
        // 业务数据
    }
}
```

### 2.2 错误响应

```json
{
    "code": 10201,
    "message": "request authorized failed: app_id mismatch",
    "data": null
}
```

### 2.3 响应码规范

| 响应码范围 | 说明 |
|-----------|------|
| 0 | 成功 |
| 102xx | API接口错误（请求解析、参数校验） |
| 103xx | 任务调度错误（任务状态、资源分配） |
| 104xx | 子任务执行错误（沙箱、评测执行） |

---

## 三、对外API（用户接口）

### 3.1 任务提交接口

#### 接口信息

| 项目 | 说明 |
|------|------|
| 路径 | `POST /v1/tasks` |
| 说明 | 提交评测任务，异步返回任务ID |
| 调用方 | API调用方 |

#### 请求参数

**请求体（JSON）**：

```json
{
    "app_id": "agent-eval-app",
    "description": "Skill评测任务",
    "type": "agent_eval",
    "config": {
        // AgentEvalConfig 或 ModelEvalConfig，根据 type 字段解析
    }
}
```

**字段说明**：

| 字段 | 类型 | 必填 | 说明 | 校验规则 |
|------|------|------|------|----------|
| `app_id` | string | 是 | 应用标识 | 非空，最大64字符，匹配 `[a-zA-Z0-9_-]+` |
| `description` | string | 否 | 任务描述 | 最大200字符 |
| `type` | string | 是 | 任务类型 | 枚举：`model_eval` / `agent_eval` |
| `config` | object | 是 | 任务配置 | 动态结构，按 `type` 字段解析为对应配置类型 |

**配置类型映射**：

| type | 配置类型 | 说明 |
|------|----------|------|
| `model_eval` | ModelEvalConfig | 模型评测配置，详见 [5.1.1 ModelEvalConfig](#511-model_eval_config模型评测配置) |
| `agent_eval` | AgentEvalConfig | 应用评测配置，详见 [5.1.2 AgentEvalConfig](#512-agent_eval_config应用评测配置) |

#### 响应示例

**成功响应**：

```json
{
    "code": 0,
    "message": "ok",
    "data": {
        "task_id": "task-abc123",
        "status": "waiting",
        "created_at": 1761983527
    }
}
```

**错误响应**：

```json
{
    "code": 10204,
    "message": "request param invalid: type must be one of [model_eval, agent_eval]",
    "data": null
}
```

---

### 3.2 任务查询接口

#### 接口信息

| 项目 | 说明 |
|------|------|
| 路径 | `GET /v1/tasks/:task_id` |
| 说明 | 查询任务详情和执行状态 |
| 调用方 | API调用方 |

#### 请求参数

**路径参数**：

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `task_id` | string | 是 | 任务ID |

**查询参数**：

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `app_id` | string | 是 | 应用ID（用于权限校验） |

**请求示例**：

```
GET /v1/tasks/task-abc123?app_id=agent-eval-app
```

#### 响应示例

**成功响应（应用评测任务-运行中）**：

```json
{
    "code": 0,
    "message": "ok",
    "data": {
        "task_id": "task-abc123",
        "app_id": "agent-eval-app",
        "type": "agent_eval",
        "description": "Skill评测任务",
        "status": "running",
        "progress": 50,
        "message": "执行inference阶段",
        "config": {
            "workflow": ["synthesis", "inference", "eval", "report"],
            "models": [{"id": "ID_JUDGE_001", "model": "deepseek-r1"}]
        },
        "result": null,
        "created_at": 1761983527,
        "updated_at": 1761983627,
        "expired_at": 1762583527
    }
}
```

**成功响应（应用评测任务-已完成）**：

```json
{
    "code": 0,
    "message": "ok",
    "data": {
        "task_id": "task-abc123",
        "app_id": "agent-eval-app",
        "type": "agent_eval",
        "description": "Skill评测任务",
        "status": "finished",
        "progress": 100,
        "message": "评测完成",
        "config": {
            "workflow": ["synthesis", "inference", "eval", "report"],
            "models": [{"id": "ID_JUDGE_001", "model": "deepseek-r1"}]
        },
        "result": {
            "success": 1,
            "message": "",
            "progress": 100,
            "report_url": "https://storage.example.com/reports/task-abc123/report.html",
            "usages": [
                {
                    "model_id": "ID_JUDGE_001",
                    "input_tokens": 50000,
                    "output_tokens": 10000
                }
            ]
        },
        "created_at": 1761983527,
        "updated_at": 1761984000,
        "expired_at": 1762583527
    }
}
```

**成功响应（模型评测任务-已完成）**：

```json
{
    "code": 0,
    "message": "ok",
    "data": {
        "task_id": "task-def456",
        "app_id": "maas",
        "type": "model_eval",
        "description": "模型评测任务",
        "status": "finished",
        "progress": 100,
        "message": "评测完成",
        "config": {
            "dataset": "http://storage.example.com/datasets/dataset.xlsx",
            "agents": [{"id": "agent-001", "name": "TestAgent"}],
            "models": [{"id": "judge-001", "model": "deepseek-r1"}]
        },
        "result": {
            "success": 1,
            "message": "",
            "progress": 100,
            "report_url": "https://storage.example.com/reports/task-def456/report.xlsx",
            "usages": [
                {
                    "model_id": "judge-001",
                    "input_tokens": 30000,
                    "output_tokens": 5000
                }
            ]
        },
        "created_at": 1761983000,
        "updated_at": 1761983500,
        "expired_at": 1762583000
    }
}
```

---

### 3.3 任务取消接口

#### 接口信息

| 项目 | 说明 |
|------|------|
| 路径 | `PATCH /v1/tasks/:task_id` |
| 说明 | 取消正在执行的任务 |
| 调用方 | API调用方 |

#### 请求参数

**路径参数**：

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `task_id` | string | 是 | 任务ID |

**请求体（JSON）**：

```json
{
    "app_id": "agent-eval-app",
    "action": "cancel"
}
```

**字段说明**：

| 字段 | 类型 | 必填 | 说明 | 校验规则 |
|------|------|------|------|----------|
| `app_id` | string | 是 | 应用ID | 非空，与任务app_id一致 |
| `action` | string | 是 | 操作类型 | 固定值 `cancel` |

#### 响应示例

**成功响应**：

```json
{
    "code": 0,
    "message": "ok",
    "data": {
        "task_id": "agent-eval-abc123",
        "status": "canceled",
        "updated_at": 1761983700
    }
}
```

**错误响应（任务已完成）**：

```json
{
    "code": 10307,
    "message": "task already finished: cannot cancel",
    "data": null
}
```

---

### 3.4 任务列表接口

#### 接口信息

| 项目 | 说明 |
|------|------|
| 路径 | `GET /v1/tasks` |
| 说明 | 查询应用下的任务列表 |
| 调用方 | API调用方 |

#### 请求参数

**查询参数**：

| 参数 | 类型 | 必填 | 说明 | 校验规则 |
|------|------|------|------|----------|
| `app_id` | string | 是 | 应用ID | 非空 |
| `type` | string | 否 | 任务类型筛选 | 枚举：`model_eval`/`agent_eval` |
| `status` | string | 否 | 任务状态筛选 | 枚举：waiting/running/finished/failed/canceled |
| `offset` | int | 否 | 偏移量 | 默认0，最小0 |
| `limit` | int | 否 | 返回数量 | 默认20，最大100 |

**请求示例**：

```
GET /v1/tasks?app_id=agent-eval-app&type=agent_eval&status=running&offset=0&limit=20
```

#### 响应示例

**成功响应**：

```json
{
    "code": 0,
    "message": "ok",
    "data": {
        "total": 100,
        "offset": 0,
        "limit": 20,
        "items": [
            {
                "task_id": "task-abc123",
                "app_id": "agent-eval-app",
                "type": "agent_eval",
                "description": "Skill评测任务",
                "status": "running",
                "progress": 50,
                "message": "执行inference阶段",
                "created_at": 1761983527,
                "updated_at": 1761983627
            },
            {
                "task_id": "task-def456",
                "app_id": "agent-eval-app",
                "type": "model_eval",
                "description": "模型评测任务",
                "status": "finished",
                "progress": 100,
                "message": "评测完成",
                "created_at": 1761983000,
                "updated_at": 1761983500
            }
        ]
    }
}
```

---

## 四、内部API（调度↔执行）

### 4.1 子任务分发接口

#### 接口信息

| 项目 | 说明 |
|------|------|
| 路径 | `POST /internal/subtasks/dispatch` |
| 说明 | 调度服务向执行服务分发子任务 |
| 调用方 | 调度服务 |
| 被调方 | 执行服务 |

#### 请求参数

**请求体（JSON）**：

```json
{
    "sub_task_id": "subtask-001",
    "task_id": "task-abc123",
    "type": "agent_eval",
    "sub_type": "inference",
    "input": {
        "command": "astron-eval",
        "args": ["--config", "./task.yaml", "--stage", "inference"],
        "envs": {
            "API_KEY": "sk-xxx",
            "API_URL": "https://api.example.com"
        },
        "input_files": [
            {"name": "task.yaml", "content": "..."},
            {"name": "setting.yaml", "content": "..."}
        ],
        "output_specs": [
            {"name": "transcript.jsonl"},
            {"name": "evalset.jsonl"}
        ],
        "timeout_sec": 3600
    }
}
```

**字段说明**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `sub_task_id` | string | 是 | 子任务唯一标识 |
| `task_id` | string | 是 | 所属任务ID |
| `type` | string | 是 | 任务类型（agent_eval/model_eval），用于 Registry 选择 Handler |
| `sub_type` | string | 是 | 子任务阶段类型（synthesis/inference/eval/report） |
| `input` | object | 是 | 执行参数（命令、环境变量、输入文件等） |

#### 响应示例

**成功响应**：

```json
{
    "code": 0,
    "message": "ok",
    "data": {
        "sub_task_id": "subtask-001",
        "status": "allocated"
    }
}
```

**错误响应（执行服务负载过高）**：

```json
{
    "code": 10400,
    "message": "executor overloaded: max concurrent subtasks reached",
    "data": null
}
```

---

### 4.2 子任务状态上报接口

#### 接口信息

| 项目 | 说明 |
|------|------|
| 路径 | `POST /internal/subtasks/status` |
| 说明 | 执行服务向调度服务上报子任务状态 |
| 调用方 | 执行服务 |
| 被调方 | 调度服务 |

#### 请求参数

**请求体（JSON）**：

```json
{
    "task_id": "task-abc123",
    "sub_task_id": "subtask-001",
    "status": 3,
    "result": {
        "success": 1,
        "message": "",
        "progress": 100,
        "report_url": "",
        "usages": [
            {
                "model_id": "ID_JUDGE_001",
                "input_tokens": 1000,
                "output_tokens": 500
            }
        ],
        "output_files": [
            "http://storage.example.com/outputs/task-abc123/transcript.jsonl",
            "http://storage.example.com/outputs/task-abc123/evalset.jsonl"
        ]
    }
}
```

**字段说明**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `task_id` | string | 是 | 所属任务ID |
| `sub_task_id` | string | 是 | 子任务唯一标识 |
| `status` | int | 是 | 子任务状态码：0-6（见数据模型文档） |
| `result` | object | 否 | 执行结果（完成状态必填） |

#### 响应示例

**成功响应**：

```json
{
    "code": 0,
    "message": "ok"
}
```

---

### 4.3 执行服务健康检查接口

#### 接口信息

| 项目 | 说明 |
|------|------|
| 路径 | `GET /internal/health` |
| 说明 | 调度服务检查执行服务健康状态 |
| 调用方 | 调度服务 |
| 被调方 | 执行服务 |

#### 响应示例

**成功响应（健康）**：

```json
{
    "code": 0,
    "message": "ok",
    "data": {
        "healthy": true,
        "active_subtasks": 5,
        "max_subtasks": 20,
        "load_percent": 25
    }
}
```

**成功响应（不健康）**：

```json
{
    "code": 0,
    "message": "ok",
    "data": {
        "healthy": false,
        "active_subtasks": 20,
        "max_subtasks": 20,
        "load_percent": 100
    }
}
```

---

### 4.4 子任务取消接口

#### 接口信息

| 项目 | 说明 |
|------|------|
| 路径 | `POST /internal/subtasks/:sub_task_id/cancel` |
| 说明 | 调度服务通知执行服务取消子任务 |
| 调用方 | 调度服务 |
| 被调方 | 执行服务 |

#### 请求参数

**路径参数**：

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `sub_task_id` | string | 是 | 子任务唯一标识 |

**请求示例**：

```
POST /internal/subtasks/subtask-001/cancel
```

#### 响应示例

**成功响应**：

```json
{
    "code": 0,
    "message": "ok",
    "data": {
        "sub_task_id": "subtask-001",
        "status": "canceled"
    }
}
```

---

## 五、数据结构定义

### 5.1 任务配置（TaskConfig）

任务配置根据 `type` 字段解析为不同类型：

#### 5.1.1 ModelEvalConfig（模型评测配置）

**适用类型**：`type = model_eval`

```json
{
    "version": "0.2.0",
    "dataset": "http://storage.example.com/datasets/dataset.xlsx",
    "agents": [
        {
            "id": "agent-001",
            "name": "TestAgent",
            "config": {
                "model": "deepseek-r1",
                "prompt": "..."
            }
        }
    ],
    "models": [
        {
            "id": "judge-001",
            "type": "api-openai",
            "api_key": "sk-xxx",
            "api_url": "https://api.example.com",
            "model": "deepseek-r1",
            "concurrency": 40
        }
    ],
    "eval": [
        {
            "dimension": "accuracy",
            "type": "subj_assessment",
            "judge": "judge-001"
        }
    ]
}
```

**字段说明**：

| 字段 | 类型 | 必填 | 说明 | 校验规则 |
|------|------|------|------|----------|
| `version` | string | 否 | 配置版本 | 默认 `0.2.0` |
| `dataset` | string | 是 | 评测数据集URL | 非空，有效的URL格式 |
| `agents` | array | 是 | 待评测Agent配置列表 | 非空数组 |
| `models` | array | 是 | 评委模型配置列表 | 非空数组，元素见 ModelConfig |
| `eval` | array | 是 | 评测维度配置列表 | 非空数组 |

**AgentConfig 字段说明**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `id` | string | 是 | Agent唯一标识 |
| `name` | string | 是 | Agent名称 |
| `config` | object | 是 | Agent运行配置（动态结构） |

**EvalDimension 字段说明**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `dimension` | string | 是 | 评测维度名称 |
| `type` | string | 是 | 评测类型：`subj_assessment`（主观）/ `obj_assessment`（客观） |
| `judge` | string | 否 | 评委模型ID（主观评测必填） |

---

#### 5.1.2 AgentEvalConfig（应用评测配置）

**适用类型**：`type = agent_eval`

```json
{
    "version": "1.0.0",
    "workflow": ["synthesis", "inference", "eval", "report"],
    "models": [
        {
            "id": "ID_JUDGE_001",
            "type": "api-openai",
            "api_key": "sk-xxx",
            "api_url": "https://api.example.com",
            "model": "deepseek-r1",
            "concurrency": 40
        }
    ],
    "synthesis": {
        "type": "agent",
        "count": 10,
        "model": "ID_JUDGE_001"
    },
    "inference": {
        "type": "claude-code",
        "model": "ID_JUDGE_001",
        "merge_eval": false
    },
    "eval": {
        "model": "ID_JUDGE_001"
    },
    "report": {
        "type": "agent",
        "model": "ID_JUDGE_001"
    },
    "resources": "http://storage.example.com/resources/task-abc123.zip"
}
```

**字段说明**：

| 字段 | 类型 | 必填 | 说明 | 校验规则 |
|------|------|------|------|----------|
| `version` | string | 否 | 配置版本 | 默认 `1.0.0` |
| `workflow` | array | 是 | 执行阶段列表 | 非空数组，元素为 `synthesis`/`inference`/`eval`/`report` |
| `models` | array | 是 | 模型配置列表 | 非空数组，元素见 ModelConfig |
| `synthesis` | object | 否 | synthesis阶段配置 | workflow包含synthesis时必填 |
| `inference` | object | 否 | inference阶段配置 | workflow包含inference时必填 |
| `eval` | object | 否 | eval阶段配置 | workflow包含eval时必填 |
| `report` | object | 否 | report阶段配置 | workflow包含report时必填 |
| `resources` | string | 否 | 资源文件包URL | 有效URL格式 |

**阶段配置说明**：

| 阶段 | 配置字段 | 说明 |
|------|----------|------|
| synthesis | `synthesis` | 评测集合成配置，包含type、count、model等 |
| inference | `inference` | Agent推理执行配置，包含type、model、merge_eval等 |
| eval | `eval` | 评测执行配置，包含model等 |
| report | `report` | 报告生成配置，包含type、model等 |

---

### 5.2 ModelConfig（模型配置）

适用于 ModelEvalConfig 和 AgentEvalConfig 中的 models 字段。

```json
{
    "id": "ID_JUDGE_001",
    "type": "api-openai",
    "api_key": "sk-xxx",
    "api_url": "https://api.example.com",
    "model": "deepseek-r1",
    "concurrency": 40
}
```

**字段说明**：

| 字段 | 类型 | 必填 | 说明 | 校验规则 |
|------|------|------|------|----------|
| `id` | string | 是 | 模型唯一标识 | 非空，其他配置通过id引用 |
| `type` | string | 是 | 模型类型 | 枚举：`api-openai`/`api-anthropic`/`local` |
| `api_key` | string | 否 | API密钥 | type为api-*时必填 |
| `api_url` | string | 否 | API地址 | type为api-*时必填 |
| `model` | string | 是 | 模型名称 | 非空 |
| `concurrency` | int | 否 | 并发数 | 默认10，最小1，最大100 |

---

### 5.3 任务类型扩展机制

为支持后续新增评测任务类型，采用分层 Registry + 策略模式的扩展机制：

#### 设计原则

- **调度服务侧**：
  - ConfigParserRegistry：按 `Task.type` 注册 ConfigParser（供 API 层校验）
  - TaskSplitter：内部注册 SplitStrategy（拆分策略）
  - DAGBuilder：内部注册 BuildStrategy（构建策略）
- **执行服务侧**：SubTaskHandlerRegistry 按 `(type, sub_type)` 注册 Handler
- **新增类型只需实现策略接口并注册，无需修改现有代码**

#### 扩展步骤

1. **定义任务类型枚举值**：在 `type` 字段的枚举列表中新增类型
2. **定义配置结构**：创建对应的 Config 结构定义（如 `XXXEvalConfig`）
3. **实现 ConfigParser**：解析和校验配置
4. **实现 SplitStrategy**：定义拆分为子任务的逻辑
5. **实现 BuildStrategy**：定义 DAG 依赖关系
6. **实现 SubTaskHandler**：各阶段的执行逻辑
7. **注册到对应组件**：服务启动时调用 Register 方法

#### 调度服务组件

| 组件 | 注册方法 | 策略接口 |
|------|---------|---------|
| ConfigParserRegistry | `Register(taskType, parser)` | `ConfigParser` |
| TaskSplitter | `RegisterStrategy(taskType, strategy)` | `SplitStrategy` |
| DAGBuilder | `RegisterStrategy(taskType, strategy)` | `BuildStrategy` |

#### 执行服务组件

| 组件 | 注册方法 | 策略接口 |
|------|---------|---------|
| SubTaskHandlerRegistry | `Register(type, subType, handler)` | `SubTaskHandler` |

#### SubTask 数据结构

执行服务根据 `type` 和 `sub_type` 组合键选择 Handler：

```javascript
{
    "sub_task_id": "subtask-001",
    "task_id": "task-abc123",
    "type": "agent_eval",       // 任务类型
    "sub_type": "inference",    // 子任务阶段类型
    // ...
}
```

#### 扩展示例

新增 `safety_eval` 任务类型：

| 步骤 | 组件 | 实现 |
|-----|------|------|
| 1 | ConfigParser | `SafetyEvalConfigParser` |
| 2 | SplitStrategy | `SafetyEvalSplitStrategy` |
| 3 | BuildStrategy | `SafetyEvalBuildStrategy` |
| 4 | SubTaskHandler | `SafetyEvalXxxHandler`（各阶段） |

**无需修改** TaskManager、SubTaskManager、DAGScheduler 等现有代码。

---

### 5.4 TaskResult（评测结果）

```json
{
    "success": 1,
    "message": "",
    "progress": 100,
    "report_url": "https://storage.example.com/reports/task-abc123/report.html",
    "usages": [
        {
            "model_id": "ID_JUDGE_001",
            "input_tokens": 50000,
            "output_tokens": 10000
        }
    ]
}
```

**字段说明**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `success` | int | 是 | 执行结果：1成功 / -1失败 |
| `message` | string | 否 | 失败信息 |
| `progress` | int | 是 | 进度 0-100 |
| `report_url` | string | 否 | 评测报告URL（完成状态填充） |
| `usages` | array | 否 | Token消耗统计列表 |

### 5.5 SubTaskResult（子任务执行结果）

```json
{
    "success": 1,
    "message": "",
    "progress": 100,
    "report_url": "",
    "usages": [
        {
            "model_id": "ID_JUDGE_001",
            "input_tokens": 1000,
            "output_tokens": 500
        }
    ],
    "output_files": [
        "http://storage.example.com/outputs/task-abc123/transcript.jsonl",
        "http://storage.example.com/outputs/task-abc123/evalset.jsonl"
    ]
}
```

**字段说明**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `success` | int | 是 | 执行结果：1成功 / -1失败 |
| `message` | string | 否 | 执行消息（失败时填充错误信息） |
| `progress` | int | 是 | 进度 0-100 |
| `report_url` | string | 否 | 报告URL |
| `usages` | array | 否 | Token消耗统计列表 |
| `output_files` | array | 否 | 输出文件URL列表 |

---

## 六、错误码详细定义

### 6.1 API接口错误码 (102xx)

| 错误码 | 错误类型 | 说明 | 典型场景 |
|-------|---------|------|----------|
| 10201 | request_authorized_failed | 权限校验失败 | app_id与任务不匹配 |
| 10202 | request_body_read_error | 读取请求体错误 | 请求体过大、格式错误 |
| 10203 | request_body_decode_error | 解析请求体错误 | JSON格式错误、字段类型错误 |
| 10204 | request_param_invalid_error | 请求参数校验失败 | 必填字段缺失、参数格式错误 |
| 10205 | config_parse_error | 配置解析错误 | workflow格式错误、models缺失 |

### 6.2 任务调度错误码 (103xx)

| 错误码 | 错误类型 | 说明 | 典型场景 |
|-------|---------|------|----------|
| 10300 | no_available_executor_error | 无可用执行服务 | 所有执行服务失联或负载过高 |
| 10301 | executor_http_request_error | HTTP请求错误 | 网络超时、连接失败 |
| 10302 | executor_response_code_error | 响应码错误 | 执行服务返回非0响应码 |
| 10303 | executor_not_alive_error | 执行服务失联 | 健康检查连续失败 |
| 10304 | model_not_found | 模型配置不存在 | 引用的model_id在models列表中不存在 |
| 10305 | dag_node_blocked_error | DAG节点阻塞 | 前置依赖节点未完成 |
| 10306 | task_not_found_error | 任务不存在 | task_id不存在 |
| 10307 | task_already_finished_error | 任务已完成 | 无法取消已完成的任务 |
| 10308 | task_retry_count_over_limit_error | 重试次数超限 | 子任务重试次数超过最大限制 |

### 6.3 子任务执行错误码 (104xx)

| 错误码 | 错误类型 | 说明 | 典型场景 |
|-------|---------|------|----------|
| 10400 | subtask_not_found_error | 子任务不存在 | sub_task_id不存在 |
| 10401 | sandbox_create_error | 创建沙箱失败 | AgentCube服务异常 |
| 10402 | sandbox_upload_error | 上传资源失败 | 文件上传失败、网络异常 |
| 10403 | sandbox_execute_error | 执行命令失败 | 评测工具执行异常 |
| 10404 | sandbox_download_error | 下载结果失败 | 结果文件不存在 |
| 10405 | sandbox_timeout_error | 执行超时 | 命令执行超过时间限制 |
| 10406 | sandbox_close_error | 关闭沙箱失败 | 沙箱实例关闭异常 |
| 10407 | evaldata_fetch_error | 获取评测集失败 | 数据管理服务异常 |
| 10408 | evaldata_upload_error | 上传结果失败 | 数据管理服务异常 |
| 10409 | output_parse_error | 输出解析错误 | 结果文件格式错误 |

---

## 七、接口路径汇总

| 接口 | 路径 | 方法 | 类型 |
|------|------|------|------|
| 任务提交 | `/v1/tasks` | POST | 对外 |
| 任务查询 | `/v1/tasks/:task_id` | GET | 对外 |
| 任务取消 | `/v1/tasks/:task_id` | PATCH | 对外 |
| 任务列表 | `/v1/tasks` | GET | 对外 |
| 子任务分发 | `/internal/subtasks/dispatch` | POST | 内部 |
| 子任务状态上报 | `/internal/subtasks/status` | POST | 内部 |
| 执行服务健康检查 | `/internal/health` | GET | 内部 |
| 子任务取消 | `/internal/subtasks/:sub_task_id/cancel` | POST | 内部 |

---

## 八、相关文档

- [架构设计文档](架构设计文档.md) - 系统架构、核心概念
- [数据模型设计文档](数据模型设计文档.md) - 数据结构、状态枚举定义
- [模块设计文档](模块设计文档.md) - 服务模块详细设计