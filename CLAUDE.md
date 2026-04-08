# 应用评测服务 (agent-eval-server)

## 项目概述

### 背景

应用评测服务旨在实现对AI应用(Agent)的自动化评测能力。与模型评测服务不同，应用评测关注Agent在特定场景下的端到端表现，包括:
- 应用场景下的评测集合成
- Agent推理执行与交互过程记录
- 多维度评测指标计算
- 评测报告生成与汇总

### 目标

构建一个分布式评测服务系统，支持:
- 通过API接口提交应用评测任务
- 任务自动拆分与调度
- 在隔离的沙箱环境中执行评测
- 评测数据的持久化存储与管理
- 评测结果的汇总与报告生成

### 核心功能定位

| 功能 | 说明 |
|-----|------|
| 任务管理 | 接收任务请求，管理任务生命周期 |
| 任务调度 | 任务拆分、子任务分发、资源分配 |
| 子任务执行 | 在沙箱环境中执行评测工具 |
| 数据管理 | 评测集、评测记录的存储与查询 |
| 结果汇总 | 评测结果聚合、报告生成 |

## 需求描述

### 用户交互流程概览

```
用户 → 调度服务(API) → 任务创建 → 任务拆分 → 子任务调度 → 执行服务 → 沙箱执行 → 结果汇总 → 用户
```

用户通过调度服务提供的API接口提交应用评测任务。任务包含:
- 评测配置 (workflow定义、模型配置、评测维度等)
- 评测资源 (skills、resource files等)

### 任务执行流程概览

任务执行包含以下workflow阶段(可配置执行部分或全部):

| 阶段 | 说明 | 输入 | 输出 |
|-----|------|------|------|
| synthesis | 评测集合成 | 场景描述、skills | 评测集(dataset.jsonl, evalset.jsonl) |
| inference | Agent推理执行 | 评测集、skills | 推理结果、交互记录(transcript.jsonl) |
| eval | 评测执行 | 推理结果、评测集 | 评测结果(result.json) |
| report | 报告生成 | 评测结果 | 评测报告(conclusion.json, conclusion.html) |

### 关键功能需求

1. **任务提交**: 接收用户提交的评测任务配置
2. **任务拆分**: 根据workflow配置拆分为子任务
3. **资源准备**: 将评测所需资源(skills、数据)准备至沙箱环境
4. **沙箱执行**: 在隔离环境中运行评测工具
5. **结果收集**: 收集评测输出并上传至数据管理服务
6. **状态追踪**: 实时更新任务状态，支持用户查询
7. **失败重试**: 子任务执行失败时支持重试

## 系统组成

### 待开发组件

#### 调度服务

职责:
- 提供API接口接收任务提交、状态查询、任务取消
- 任务拆分逻辑(根据workflow拆分子任务)
- 子任务调度与分发(将子任务分发至执行服务)
- 子任务状态追踪与汇总
- 任务生命周期管理

参考: 模型评测服务(eval-server)的调度服务设计

#### 执行服务

职责:
- 接收调度服务分发的子任务
- 管理沙箱实例的创建与销毁
- 将评测资源上传至沙箱
- 在沙箱中执行评测工具命令
- 从沙箱下载评测结果
- 将结果上传至数据管理服务
- 子任务状态上报

参考: 模型评测服务(eval-server)的执行服务设计，但需适配远程沙箱模式

#### 评测工具

职责:
- 在沙箱环境中执行具体评测逻辑
- 支持各workflow阶段的命令执行
- 输出结构化的评测结果

形式: Python编写的命令行工具，安装至沙箱环境

### 依赖项目

#### AgentCube 沙箱服务

项目路径: `/home/xmq/projects/go/AgentCube`

定位: 提供隔离的沙箱执行环境

核心能力:
- 沙箱创建与会话管理
- 命令执行 (ExecuteCommand, RunCode)
- 文件操作 (Upload, Download, Write, Read)
- 运行时管理 (Shell, Python环境)

接口方式: Go SDK (`github.com/volcano-sh/agentcube/go-sdk`)

详细文档: [docs/dependencies/agentcube-sdk.md](docs/dependencies/agentcube-sdk.md)

#### 数据管理服务

项目路径: `/home/xmq/projects/go/eval-data`

定位: 评测数据持久化存储

核心能力:
- 评测集(Evalset)的存储与查询
- 评测记录(Record)的存储与查询

接口方式: HTTP API

详细文档: [docs/dependencies/eval-data-api.md](docs/dependencies/eval-data-api.md)

## 参考项目

### 模型评测服务 (eval-server)

项目路径: `/home/xmq/projects/go/eval-server`

定位: 架构设计参考

可借鉴内容:
- 调度服务架构: 任务管理、子任务拆分、状态流转
- 执行服务架构: 槽位管理、异步执行、状态轮询
- API设计: 任务提交/查询/取消接口
- 数据模型: Task/SubTask数据结构

关键差异:
- 模型评测的执行服务本身是沙箱环境
- 应用评测的执行服务通过SDK管理远程沙箱

详细文档:
- [docs/references/eval-server-architecture.md](docs/references/eval-server-architecture.md)
- [docs/references/eval-server-api.md](docs/references/eval-server-api.md)

## 术语定义

| 术语 | 定义 |
|-----|------|
| 任务(Task) | 用户提交的完整评测请求，包含评测配置和workflow定义 |
| 子任务(SubTask) | 任务拆分后的执行单元，对应一个workflow阶段或一个case的执行 |
| Workflow | 任务执行的阶段序列，包含synthesis→inference→eval→report |
| 沙箱(Sandbox) | AgentCube提供的隔离执行环境，用于运行评测工具 |
| 评测集(Evalset) | 待评测的数据集合，包含question、answer等字段 |
| 评测记录(Record) | 单条评测结果，包含维度、结果、理由等字段 |
| Skills | Agent评测所需的技能配置文件 |
| Resource | 评测所需的资源文件(pdf、txt、xlsx、docx等) |

## 开发阶段规划

### 当前阶段: 文档完善

目标: 建立完整的参考文档和需求文档

产出:
- CLAUDE.md (项目基础介绍)
- docs/references/ (参考项目文档)
- docs/dependencies/ (依赖项目文档)

### 后续阶段: 方案设计

目标: 设计应用评测服务的详细架构

产出:
- 架构设计文档
- API接口定义
- 数据模型定义
- 任务拆分与调度逻辑设计

### 后续阶段: 代码开发

目标: 实现调度服务、执行服务、评测工具

产出:
- 调度服务代码
- 执行服务代码
- 评测工具代码
- 部署配置

## 文档索引

### 参考文档
- [模型评测服务设计文档](docs/references/模型评测服务设计文档.md)

### 设计文档
- [应用评测任务设计文档](docs/designs/应用评测任务设计文档.md)

### 依赖文档
- [AgentCube SDK使用说明](docs/dependencies/AgentCube-SDK使用说明.md)
- [数据管理服务接口文档](docs/dependencies/数据管理服务接口文档.md)