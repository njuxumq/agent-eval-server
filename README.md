# Agent Eval Server

应用评测服务 - Skill 自动化评测平台

## 项目简介

应用评测服务（agent-eval-server）是一个分布式评测系统，用于对 **Skill（技能）** 进行自动化评测。系统在隔离的沙箱环境中执行评测流程，评估 Skill 在特定场景下的表现。

### 核心特性

- **Skill 评测**：支持用户自定义 Skill 配置文件的评测
- **四阶段 Workflow**：synthesis → inference → eval → report
- **分布式架构**：调度服务 + 执行服务，支持水平扩展
- **沙箱隔离**：基于 AgentCube 的隔离执行环境
- **DAG 调度**：支持任务依赖关系的自动调度
- **数据持久化**：评测数据、结果的持久化存储

### 系统架构

```
┌─────────────┐      ┌─────────────────┐      ┌─────────────────┐
│   用户层    │ ──→  │    调度服务      │ ──→  │   执行服务集群   │
└─────────────┘      │  (主从模式)      │      │   (分布式)      │
                     └────────┬────────┘      └────────┬────────┘
                              │                        │
                              ▼                        ▼
                     ┌────────────────┐      ┌─────────────────┐
                     │    MongoDB     │      │   AgentCube     │
                     │   (数据存储)    │      │   (沙箱服务)     │
                     └────────────────┘      └─────────────────┘
                                                      │
                                              ┌───────┴───────┐
                                              │   eval-data   │
                                              │   (数据服务)   │
                                              └───────────────┘
```

## 快速开始

### 环境要求

| 组件 | 版本要求 |
|------|----------|
| Go | ≥ 1.21 |
| MongoDB | ≥ 5.0 |
| Docker | ≥ 20.10 (可选，用于本地开发) |

### 安装依赖

```bash
# 克隆项目
git clone https://github.com/example/agent-eval-server.git
cd agent-eval-server

# 安装 Go 依赖
go mod download
```

### 配置

```bash
# 复制配置文件模板
cp configs/scheduler.yaml.example configs/scheduler.yaml
cp configs/executor.yaml.example configs/executor.yaml

# 修改配置
vim configs/scheduler.yaml
vim configs/executor.yaml
```

### 启动服务

```bash
# 启动调度服务
go run cmd/scheduler/main.go

# 启动执行服务（新终端）
go run cmd/executor/main.go
```

### 提交评测任务

```bash
curl -X POST http://localhost:8080/v1/tasks \
  -H "Content-Type: application/json" \
  -d '{
    "app_id": "test-app",
    "description": "测试评测任务",
    "config": {
      "workflow": ["synthesis", "inference", "eval", "report"],
      "models": [
        {
          "id": "ID_JUDGE_001",
          "type": "api-openai",
          "api_key": "your-api-key",
          "api_url": "https://api.example.com",
          "model": "deepseek-r1",
          "concurrency": 10
        }
      ]
    },
    "resources": {
      "skills": "BASE64_ENCODED_SKILLS",
      "files": {}
    }
  }'
```

### 查询任务状态

```bash
curl http://localhost:8080/v1/tasks/{task_id}?app_id=test-app
```

## 项目结构

```
agent-eval-server/
├── cmd/                        # 服务入口
│   ├── scheduler/              # 调度服务
│   │   └── main.go
│   └── executor/               # 执行服务
│       └── main.go
│
├── internal/                   # 内部代码
│   ├── scheduler/              # 调度服务代码
│   │   ├── api/                # API 层
│   │   ├── manager/            # 业务逻辑层
│   │   ├── storage/            # 数据存储层
│   │   └── config/             # 配置管理
│   │
│   ├── executor/               # 执行服务代码
│   │   ├── api/                # API 层
│   │   ├── manager/            # 业务逻辑层
│   │   ├── client/             # 外部服务客户端
│   │   └── config/             # 配置管理
│   │
│   └── pkg/                    # 共享代码
│       ├── errors/             # 错误定义
│       ├── models/             # 数据模型
│       ├── utils/              # 工具函数
│       └── logger/             # 日志工具
│
├── pkg/                        # 对外暴露的代码
│   └── api/                    # API 客户端
│
├── configs/                    # 配置文件
│   ├── scheduler.yaml
│   └── executor.yaml
│
├── docs/                       # 文档
│   ├── designs/                # 设计文档
│   ├── standards/              # 规范文档
│   ├── references/             # 参考文档
│   └── dependencies/           # 依赖文档
│
├── scripts/                    # 脚本
│   ├── build.sh
│   └── test.sh
│
├── test/                       # 测试代码
│   ├── integration/
│   └── e2e/
│
├── go.mod
├── go.sum
├── Makefile
├── README.md
└── CLAUDE.md
```

## 核心概念

### Skill（技能）

Skill 是评测的核心对象，是用户定义的技能配置文件。一个 Skill 描述了 Agent 应具备的能力和行为规范。

### Workflow

评测执行流程包含四个阶段：

| 阶段 | 说明 | 输入 | 输出 |
|------|------|------|------|
| synthesis | 评测集生成 | Skill、场景描述 | 评测集 (evalset.jsonl) |
| inference | Agent 推理执行 | Skill、评测集 | 交互记录 (transcript.jsonl) |
| eval | 评测指标计算 | 交互记录、评测集 | 评测结果 (result.json) |
| report | 报告生成 | 评测结果 | 评测报告 (conclusion.json/html) |

### DAG 调度

任务拆分后形成有向无环图（DAG），系统自动管理节点依赖关系和执行顺序。

```
synthesis → [case_1, case_2, ...] → report
```

## API 接口

### 对外接口

| 接口 | 方法 | 说明 |
|------|------|------|
| `/v1/tasks` | POST | 提交评测任务 |
| `/v1/tasks/:task_id` | GET | 查询任务状态 |
| `/v1/tasks/:task_id` | PATCH | 取消任务 |
| `/v1/tasks` | GET | 查询任务列表 |

### 内部接口

| 接口 | 方法 | 说明 |
|------|------|------|
| `/internal/subtasks/dispatch` | POST | 子任务分发 |
| `/internal/subtasks/status` | POST | 状态上报 |
| `/internal/health` | GET | 健康检查 |

详细接口文档见 [API接口设计文档](docs/designs/API接口设计文档.md)。

## 开发指南

### 本地开发

```bash
# 安装开发工具
make install-tools

# 运行测试
make test

# 代码检查
make lint

# 构建服务
make build
```

### 代码规范

项目遵循 [代码开发规范](docs/standards/代码开发规范.md)，主要包含：

- Go 代码风格规范
- 项目结构规范
- 错误处理规范
- 日志规范
- 测试规范
- Git 规范

### 分支管理

| 分支 | 说明 |
|------|------|
| `main` | 主分支，稳定版本 |
| `feature/*` | 功能分支 |
| `fix/*` | 修复分支 |
| `release/*` | 发布分支 |

## 文档索引

### 设计文档

| 文档 | 说明 |
|------|------|
| [架构设计文档](docs/designs/架构设计文档.md) | 系统架构总览 |
| [数据模型设计文档](docs/designs/数据模型设计文档.md) | MongoDB 表结构设计 |
| [API接口设计文档](docs/designs/API接口设计文档.md) | 接口定义与示例 |
| [模块设计文档](docs/designs/模块设计文档.md) | 调度服务与执行服务模块设计 |

#### 详细设计文档

| 文档 | 说明 |
|------|------|
| [DAG构建器设计](docs/designs/details/DAG构建器设计.md) | 任务 DAG 构建逻辑 |
| [DAG调度器设计](docs/designs/details/DAG调度器设计.md) | DAG 节点调度执行 |
| [事件总线设计](docs/designs/details/事件总线设计.md) | 任务状态事件管理 |
| [任务拆分器设计](docs/designs/details/任务拆分器设计.md) | 任务拆分逻辑 |
| [子任务处理器设计](docs/designs/details/子任务处理器设计.md) | 子任务执行处理 |

### 规范文档

| 文档 | 说明 |
|------|------|
| [代码开发规范](docs/standards/代码开发规范.md) | 代码风格、项目结构、错误处理等规范 |

### 依赖文档

| 文档 | 说明 |
|------|------|
| [AgentCube使用说明](docs/dependencies/AgentCube使用说明.md) | AgentCube 沙箱服务 SDK |
| [数据管理服务接口文档](docs/dependencies/数据管理服务接口文档.md) | eval-data 服务接口 |

### 参考文档

| 文档 | 说明 |
|------|------|
| [模型评测服务设计文档](docs/references/模型评测服务设计文档.md) | 架构参考 |

## 依赖服务

### AgentCube（沙箱服务）

提供隔离的沙箱执行环境，沙箱内预装：
- Claude Code / Open Claw（Agent 执行环境）
- 评测工具（astron-eval）

### eval-data（数据管理服务）

评测数据的持久化存储：
- 评测集（Evalset）的存储与查询
- 评测记录（Record）的存储与查询

## 部署

### Docker 部署

```bash
# 构建镜像
docker build -t agent-eval-scheduler -f Dockerfile.scheduler .
docker build -t agent-eval-executor -f Dockerfile.executor .

# 启动服务
docker-compose up -d
```

### Kubernetes 部署

```bash
# 应用配置
kubectl apply -f deploy/k8s/
```

## 监控

### Prometheus 指标

服务暴露以下 Prometheus 指标：

- `task_total`: 任务总数
- `subtask_running`: 运行中子任务数
- `executor_active`: 活跃执行服务数
- `sandbox_active`: 活跃沙箱数

### 健康检查

```bash
# 调度服务健康检查
curl http://localhost:8080/health

# 执行服务健康检查
curl http://localhost:8081/internal/health
```

## 常见问题

### Q: 如何跳过 synthesis 阶段？

在任务配置中，`workflow` 字段只包含需要的阶段即可：

```json
{
  "workflow": ["inference", "eval", "report"]
}
```

同时需要在 `config.inference.evalset_ref` 中指定已有的评测集。

### Q: 如何增加执行服务并发能力？

1. 增加单个执行服务的 `max_concurrent` 配置
2. 部署多个执行服务实例

### Q: 任务失败如何排查？

1. 查询任务状态获取错误信息
2. 查看服务日志
3. 检查沙箱执行日志

## 贡献指南

欢迎提交 Issue 和 Pull Request。

提交 PR 前请确保：
1. 代码通过 `make lint` 检查
2. 测试通过 `make test`
3. 遵循 [代码开发规范](docs/standards/代码开发规范.md)

## 许可证

MIT License

## 联系方式

- Issue: https://github.com/example/agent-eval-server/issues
- Email: team@example.com