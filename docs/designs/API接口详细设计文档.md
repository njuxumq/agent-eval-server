# API接口详细设计文档

## 一、概述

### 1.1 文档目的

本文档详细定义应用评测服务（agent-eval-server）的所有API接口，包括：
- 接口定义与请求/响应格式
- 参数校验规则
- 错误码定义
- 数据结构定义
- 调用示例

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
    "config": {
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
        }
    },
    "resources": {
        "skills": "base64编码的skills目录tar.gz内容",
        "files": {
            "a.pdf": "base64编码的PDF文件内容",
            "b.xlsx": "base64编码的Excel文件内容"
        }
    }
}
```

**字段说明**：

| 字段 | 类型 | 必填 | 说明 | 校验规则 |
|------|------|------|------|----------|
| `app_id` | string | 是 | 应用标识 | 非空，最大64字符，匹配 `[a-zA-Z0-9_-]+` |
| `description` | string | 否 | 任务描述 | 最大500字符 |
| `config` | object | 是 | 任务配置 | 见配置结构定义 |
| `resources` | object | 是 | 评测资源 | 见资源结构定义 |

#### 响应示例

**成功响应**：

```json
{
    "code": 0,
    "message": "ok",
    "data": {
        "task_id": "agent-eval-abc123",
        "status": "waiting",
        "created_at": 1761983527
    }
}
```

**错误响应**：

```json
{
    "code": 10204,
    "message": "request param invalid: workflow must contain at least one stage",
    "data": null
}
```

#### 业务逻辑

1. 参数校验：检查必填字段、配置格式、资源编码
2. 生成TaskId：UUID格式，前缀 `agent-eval-`
3. 存储资源：skills和files存储到数据库或对象存储
4. 构建Task记录：初始状态为 `Waiting`
5. 触发任务拆分：后台异步执行任务拆分逻辑
6. 返回TaskId：不等待执行完成，立即返回

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
GET /v1/tasks/agent-eval-abc123?app_id=agent-eval-app
```

#### 响应示例

**成功响应（运行中）**：

```json
{
    "code": 0,
    "message": "ok",
    "data": {
        "task_id": "agent-eval-abc123",
        "app_id": "agent-eval-app",
        "description": "Skill评测任务",
        "status": "running",
        "progress": 50,
        "message": "执行inference阶段",
        "config": {
            "workflow": ["synthesis", "inference", "eval", "report"],
            "models": [
                {
                    "id": "ID_JUDGE_001",
                    "model": "deepseek-r1"
                }
            ]
        },
        "result": null,
        "created_at": 1761983527,
        "updated_at": 1761983627,
        "expired_at": 1762583527
    }
}
```

**成功响应（已完成）**：

```json
{
    "code": 0,
    "message": "ok",
    "data": {
        "task_id": "agent-eval-abc123",
        "app_id": "agent-eval-app",
        "description": "Skill评测任务",
        "status": "finished",
        "progress": 100,
        "message": "评测完成",
        "config": {
            "workflow": ["synthesis", "inference", "eval", "report"],
            "models": [
                {
                    "id": "ID_JUDGE_001",
                    "model": "deepseek-r1"
                }
            ]
        },
        "result": {
            "report_url": "https://storage.example.com/reports/agent-eval-abc123/report.html",
            "usages": [
                {
                    "model_id": "ID_JUDGE_001",
                    "input_tokens": 50000,
                    "output_tokens": 10000
                }
            ],
            "conclusion": {
                "score": 85.5,
                "summary": "整体表现良好",
                "details": [
                    {
                        "dimension": "accuracy",
                        "score": 90.0,
                        "reason": "回答准确率高"
                    }
                ]
            }
        },
        "created_at": 1761983527,
        "updated_at": 1761984000,
        "expired_at": 1762583527
    }
}
```

**错误响应（任务不存在）**：

```json
{
    "code": 10306,
    "message": "task not found: agent-eval-abc123",
    "data": null
}
```

**错误响应（权限校验失败）**：

```json
{
    "code": 10201,
    "message": "request authorized failed: app_id mismatch",
    "data": null
}
```

#### 业务逻辑

1. 参数校验：检查task_id格式、app_id非空
2. 查询任务：从MongoDB查询Task记录
3. 权限校验：检查请求app_id与任务app_id是否一致
4. 计算进度：根据已完成子任务比例计算progress
5. 构建响应：包含任务详情、状态、结果

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

#### 业务逻辑

1. 参数校验：检查task_id格式、action为cancel
2. 查询任务：从MongoDB查询Task记录
3. 权限校验：检查请求app_id与任务app_id是否一致
4. 状态检查：任务状态必须为 `Waiting`、`Running` 或 `Exception`
5. 更新状态：Task状态改为 `Canceled`
6. 联动更新：更新所有活跃SubTask为 `Canceled`
7. 通知执行服务：调用执行服务取消接口

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
| `status` | string | 否 | 任务状态筛选 | 枚举：waiting/running/finished/failed/canceled |
| `offset` | int | 否 | 偏移量 | 默认0，最小0 |
| `limit` | int | 否 | 返回数量 | 默认20，最大100 |

**请求示例**：

```
GET /v1/tasks?app_id=agent-eval-app&status=running&offset=0&limit=20
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
                "task_id": "agent-eval-abc123",
                "app_id": "agent-eval-app",
                "description": "Skill评测任务",
                "status": "running",
                "progress": 50,
                "message": "执行inference阶段",
                "created_at": 1761983527,
                "updated_at": 1761983627
            },
            {
                "task_id": "agent-eval-def456",
                "app_id": "agent-eval-app",
                "description": "另一个评测任务",
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

#### 业务逻辑

1. 参数校验：检查app_id非空、status枚举值、分页参数范围
2. 构建查询：根据app_id和status筛选条件
3. 执行查询：按created_at倒序，应用分页参数
4. 构建响应：包含总数、分页信息、任务列表

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
    "task_id": "agent-eval-abc123",
    "stage": "inference",
    "case_id": "case-001",
    "merge_mode": false,
    "config": {
        "model_config": {
            "id": "ID_JUDGE_001",
            "type": "api-openai",
            "api_key": "sk-xxx",
            "api_url": "https://api.example.com",
            "model": "deepseek-r1",
            "concurrency": 40
        },
        "stage_config": {
            "type": "claude-code",
            "model": "ID_JUDGE_001",
            "merge_eval": false
        },
        "evalset_ref": {
            "evalset_id": "evalset-001",
            "app_id": "agent-eval-app"
        }
    },
    "resources_ref": {
        "task_id": "agent-eval-abc123"
    }
}
```

**字段说明**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `sub_task_id` | string | 是 | 子任务唯一标识 |
| `task_id` | string | 是 | 所属任务ID |
| `stage` | string | 是 | 阶段类型：synthesis/inference/eval/report |
| `case_id` | string | 否 | case标识（inference/eval阶段必填） |
| `merge_mode` | bool | 否 | 是否合并模式（默认false） |
| `config` | object | 是 | 子任务执行配置 |
| `resources_ref` | object | 是 | 资源引用（执行服务按task_id获取资源） |

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

#### 业务逻辑

1. 参数校验：检查必填字段、stage枚举值
2. 存储子任务：创建SubTask记录，状态为 `Allocated`
3. 加入执行队列：将子任务加入本地执行队列
4. 返回确认：立即返回，不等待执行完成

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
    "task_id": "agent-eval-abc123",
    "sub_task_id": "subtask-001",
    "status": "finished",
    "result": {
        "success": 1,
        "message": "",
        "output_files": {
            "transcript.jsonl": "/root/workspace/transcript.jsonl",
            "evalset.jsonl": "/root/workspace/evalset.jsonl"
        },
        "usage": {
            "model_id": "ID_JUDGE_001",
            "input_tokens": 1000,
            "output_tokens": 500
        }
    }
}
```

**字段说明**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `task_id` | string | 是 | 所属任务ID |
| `sub_task_id` | string | 是 | 子任务唯一标识 |
| `status` | string | 是 | 子任务状态：running/finished/failed/exception |
| `result` | object | 否 | 执行结果（完成状态必填） |

#### 响应示例

**成功响应**：

```json
{
    "code": 0,
    "message": "ok"
}
```

#### 业务逻辑

1. 参数校验：检查必填字段、status枚举值
2. 更新SubTask：更新子任务状态和结果
3. 检查DAGNode：检查所属DAGNode是否全部完成
4. 触发节点释放：若DAGNode完成，触发后续节点释放
5. 更新Task状态：若所有DAGNode完成，更新Task状态

---

### 4.3 执行服务健康检查接口

#### 接口信息

| 项目 | 说明 |
|------|------|
| 路径 | `GET /internal/health` |
| 说明 | 调度服务检查执行服务健康状态 |
| 调用方 | 调度服务 |
| 被调方 | 执行服务 |

#### 请求参数

无参数。

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

#### 业务逻辑

1. 检查本地状态：查询活跃子任务数量
2. 计算负载：活跃子任务数 / 最大并发数
3. 返回状态：包含健康状态、负载信息

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

**错误响应（子任务不存在）**：

```json
{
    "code": 10400,
    "message": "subtask not found: subtask-001",
    "data": null
}
```

#### 业务逻辑

1. 查询子任务：检查子任务是否存在
2. 状态检查：子任务状态必须为 `Allocated` 或 `Running`
3. 取消执行：若正在执行，关闭沙箱实例
4. 更新状态：SubTask状态改为 `Canceled`

---

## 五、数据结构定义

### 5.1 TaskConfig（任务配置）

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
    }
}
```

**字段说明**：

| 字段 | 类型 | 必填 | 说明 | 校验规则 |
|------|------|------|------|----------|
| `version` | string | 否 | 配置版本 | 默认 `1.0.0` |
| `workflow` | array | 是 | 执行阶段列表 | 非空数组，元素为synthesis/inference/eval/report |
| `models` | array | 是 | 模型配置列表 | 非空数组，每个元素见ModelConfig |
| `synthesis` | object | 否 | synthesis阶段配置 | workflow包含synthesis时必填 |
| `inference` | object | 否 | inference阶段配置 | workflow包含inference时必填 |
| `eval` | object | 否 | eval阶段配置 | workflow包含eval时必填 |
| `report` | object | 否 | report阶段配置 | workflow包含report时必填 |

### 5.2 ModelConfig（模型配置）

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
| `type` | string | 是 | 模型类型 | 枚举：api-openai/api-anthropic/local |
| `api_key` | string | 否 | API密钥 | type为api-*时必填 |
| `api_url` | string | 否 | API地址 | type为api-*时必填 |
| `model` | string | 是 | 模型名称 | 非空 |
| `concurrency` | int | 否 | 并发数 | 默认10，最小1，最大100 |

### 5.3 TaskResources（评测资源）

```json
{
    "skills": "base64编码的skills目录tar.gz内容",
    "files": {
        "a.pdf": "base64编码的PDF文件内容",
        "b.xlsx": "base64编码的Excel文件内容"
    }
}
```

**字段说明**：

| 字段 | 类型 | 必填 | 说明 | 校验规则 |
|------|------|------|------|----------|
| `skills` | string | 是 | skills目录打包内容 | Base64编码，解压后为tar.gz格式 |
| `files` | object | 否 | resource文件映射 | 文件名 → Base64编码内容 |

### 5.4 TaskResult（评测结果）

```json
{
    "report_url": "https://storage.example.com/reports/agent-eval-abc123/report.html",
    "usages": [
        {
            "model_id": "ID_JUDGE_001",
            "input_tokens": 50000,
            "output_tokens": 10000
        }
    ],
    "conclusion": {
        "score": 85.5,
        "summary": "整体表现良好",
        "details": [
            {
                "dimension": "accuracy",
                "score": 90.0,
                "reason": "回答准确率高"
            }
        ]
    }
}
```

**字段说明**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `report_url` | string | 否 | 评测报告URL（完成状态填充） |
| `usages` | array | 否 | Token消耗统计列表 |
| `conclusion` | object | 否 | 评测结论（完成状态填充） |

### 5.5 SubTaskResult（子任务执行结果）

```json
{
    "success": 1,
    "message": "",
    "output_files": {
        "transcript.jsonl": "/root/workspace/transcript.jsonl",
        "evalset.jsonl": "/root/workspace/evalset.jsonl"
    },
    "usage": {
        "model_id": "ID_JUDGE_001",
        "input_tokens": 1000,
        "output_tokens": 500
    }
}
```

**字段说明**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `success` | int | 是 | 执行结果：-1失败 / 0执行中 / 1完成 |
| `message` | string | 否 | 执行消息（失败时填充错误信息） |
| `output_files` | object | 否 | 输出文件路径映射 |
| `usage` | object | 否 | Token消耗统计 |

---

## 六、状态枚举定义

### 6.1 TaskStatus（任务状态）

| 值 | 名称 | 说明 |
|----|------|------|
| 0 | waiting | 等待中（DAG节点未释放） |
| 1 | running | 执行中（有子任务正在执行） |
| 2 | finished | 执行完成 |
| 3 | failed | 任务失败 |
| 4 | canceled | 任务取消 |
| 5 | exception | 任务异常（中间态） |

### 6.2 SubTaskStatus（子任务状态）

| 值 | 名称 | 说明 |
|----|------|------|
| 0 | waiting | 等待调度 |
| 1 | allocated | 已分配执行服务 |
| 2 | running | 运行中 |
| 3 | finished | 完成 |
| 4 | failed | 失败 |
| 5 | exception | 异常（可重试） |
| 6 | canceled | 取消 |

### 6.3 DAGNodeStatus（DAG节点状态）

| 值 | 名称 | 说明 |
|----|------|------|
| 0 | blocked | 阻塞（依赖节点未完成） |
| 1 | ready | 就绪（依赖节点已完成） |
| 2 | running | 运行中 |
| 3 | finished | 完成 |
| 4 | failed | 失败 |

---

## 七、错误码详细定义

### 7.1 API接口错误码 (102xx)

| 错误码 | 错误类型 | 说明 | 典型场景 |
|-------|---------|------|----------|
| 10201 | request_authorized_failed | 权限校验失败 | app_id与任务不匹配 |
| 10202 | request_body_read_error | 读取请求体错误 | 请求体过大、格式错误 |
| 10203 | request_body_decode_error | 解析请求体错误 | JSON格式错误、字段类型错误 |
| 10204 | request_param_invalid_error | 请求参数校验失败 | 必填字段缺失、参数格式错误 |
| 10205 | config_parse_error | 配置解析错误 | workflow格式错误、models缺失 |
| 10206 | resource_decode_error | 资源解码错误 | Base64解码失败、tar.gz解压失败 |

**错误响应示例**：

```json
// 10201 权限校验失败
{
    "code": 10201,
    "message": "request authorized failed: app_id mismatch, expected 'agent-eval-app', got 'other-app'",
    "data": null
}

// 10203 解析错误
{
    "code": 10203,
    "message": "request body decode error: invalid JSON format, unexpected character at position 42",
    "data": null
}

// 10204 参数校验失败
{
    "code": 10204,
    "message": "request param invalid: field 'workflow' is required but missing",
    "data": null
}
```

### 7.2 任务调度错误码 (103xx)

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

**错误响应示例**：

```json
// 10300 无可用执行服务
{
    "code": 10300,
    "message": "no available executor: all executors are offline or overloaded",
    "data": null
}

// 10306 任务不存在
{
    "code": 10306,
    "message": "task not found: task_id 'agent-eval-abc123' does not exist",
    "data": null
}

// 10307 任务已完成
{
    "code": 10307,
    "message": "task already finished: cannot cancel task in 'finished' status",
    "data": null
}
```

### 7.3 子任务执行错误码 (104xx)

| 错误码 | 错误类型 | 说明 | 典型场景 |
|-------|---------|------|----------|
| 10400 | subtask_not_found_error | 子任务不存在 | sub_task_id不存在 |
| 10401 | sandbox_create_error | 创建沙箱失败 | AgentCube服务异常 |
| 10402 | sandbox_upload_error | 上传资源失败 | 文件上传失败、网络异常 |
| 10403 | sandbox_execute_error | 执行命令失败 | 评测工具执行异常 |
| 10404 | sandbox_download_error | 下载结果失败 | 结果文件不存在 |
| 10405 | sandbox_timeout_error | 执行超时 | 命令执行超过时间限制 |
| 10406 | sandbox_close_error | 关闭沙箱失败 | 沙箱实例关闭异常 |
| 10407 | evaldata_fetch_error | 获取评测集失败 | eval-data服务异常 |
| 10408 | evaldata_upload_error | 上传结果失败 | eval-data服务异常 |
| 10409 | output_parse_error | 输出解析错误 | 结果文件格式错误 |

**错误响应示例**：

```json
// 10401 创建沙箱失败
{
    "code": 10401,
    "message": "sandbox create error: AgentCube service returned error: quota exceeded",
    "data": null
}

// 10405 执行超时
{
    "code": 10405,
    "message": "sandbox timeout error: command execution exceeded 10 minutes timeout",
    "data": null
}
```

---

## 八、参数校验规则

### 8.1 通用校验规则

| 字段类型 | 校验规则 |
|----------|----------|
| **string** | 非空检查、长度限制、正则匹配 |
| **int** | 范围检查、最小值、最大值 |
| **array** | 非空检查、长度限制、元素类型检查 |
| **object** | 必填字段检查、嵌套校验 |
| **base64** | Base64格式校验、解码后内容校验 |

### 8.2 各接口参数校验

#### 任务提交接口

| 字段 | 校验规则 |
|------|----------|
| `app_id` | 非空，最大64字符，匹配 `[a-zA-Z0-9_-]+` |
| `description` | 最大500字符 |
| `config.workflow` | 非空数组，元素为synthesis/inference/eval/report |
| `config.models` | 非空数组，至少1个元素 |
| `config.models[].id` | 非空，唯一性检查 |
| `config.models[].type` | 枚举：api-openai/api-anthropic/local |
| `config.models[].model` | 非空 |
| `resources.skills` | Base64编码，解码后为有效的tar.gz |

#### 任务查询接口

| 字段 | 校验规则 |
|------|----------|
| `task_id` | 非空，格式 `agent-eval-{uuid}` |
| `app_id` | 非空，最大64字符 |

#### 任务取消接口

| 字段 | 校验规则 |
|------|----------|
| `task_id` | 非空，格式 `agent-eval-{uuid}` |
| `app_id` | 非空，与任务app_id一致 |
| `action` | 固定值 `cancel` |

#### 任务列表接口

| 字段 | 校验规则 |
|------|----------|
| `app_id` | 非空，最大64字符 |
| `status` | 枚举：waiting/running/finished/failed/canceled |
| `offset` | 最小0 |
| `limit` | 最小1，最大100 |

---

## 九、调用示例

### 9.1 任务提交示例

**curl命令**：

```bash
curl -X POST "http://localhost:8080/v1/tasks" \
  -H "Content-Type: application/json" \
  -d '{
    "app_id": "agent-eval-app",
    "description": "Skill评测任务",
    "config": {
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
      }
    },
    "resources": {
      "skills": "BASE64_ENCODED_CONTENT",
      "files": {}
    }
  }'
```

### 9.2 任务查询示例

**curl命令**：

```bash
curl -X GET "http://localhost:8080/v1/tasks/agent-eval-abc123?app_id=agent-eval-app"
```

### 9.3 任务取消示例

**curl命令**：

```bash
curl -X PATCH "http://localhost:8080/v1/tasks/agent-eval-abc123" \
  -H "Content-Type: application/json" \
  -d '{
    "app_id": "agent-eval-app",
    "action": "cancel"
  }'
```

### 9.4 任务列表查询示例

**curl命令**：

```bash
curl -X GET "http://localhost:8080/v1/tasks?app_id=agent-eval-app&status=running&offset=0&limit=20"
```

---

## 十、附录

### A. 接口路径汇总

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

### B. HTTP状态码使用规范

| HTTP状态码 | 使用场景 |
|-----------|----------|
| 200 | 请求成功（业务响应码在body中） |
| 400 | 请求格式错误（无法解析请求体） |
| 404 | 资源不存在（路径错误） |
| 500 | 服务内部错误（未预期的异常） |

### C. 接口版本管理

| 版本 | 路径前缀 | 说明 |
|------|----------|------|
| v1 | `/v1/` | 当前版本 |
| v2 | `/v2/` | 未来版本（预留） |

**版本升级策略**：
- 新增接口：可直接在当前版本添加
- 兼容修改：在当前版本扩展，保持向后兼容
- 不兼容修改：发布新版本，旧版本标记deprecated