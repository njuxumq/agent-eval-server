---
name: context-reset-methodology
description: Use when task is expected to run long, to learn how to design context reset and state transfer
---

# Context Reset 设计方法论

## 核心理念

清空上下文窗口，通过 Artifact 传递状态，防止上下文焦虑导致过早收尾。

---

## 为什么需要 Context Reset

| 问题 | 说明 |
|------|------|
| **上下文填充** | 长任务中 context window 填满，失去连贯性 |
| **上下文焦虑** | 接近上下文限制时过早收尾（"差不多了"） |
| **Compaction 局限** | 原地压缩不提供"干净起点" |

---

## Compaction vs Context Reset

| 方式 | 操作 | 效果 |
|------|------|------|
| **Compaction** | 原地压缩历史为摘要 | 保持同一 agent，焦虑可能残留 |
| **Context Reset** | 清空 context，启动新 agent | 干净起点，需 Artifact 传递状态 |

**选择依据**：超长任务（几小时）仍需 Context Reset

---

## 如何判断是否需要 Context Reset

| 条件 | 说明 |
|------|------|
| 任务时长 = 长 | 预计 > 50 步操作或几小时 |
| 上下文填充度 > 70% | Claude 出现收尾倾向 |
| Generator 提前结束 | "差不多了，剩下的你自己处理" |
| 跨 session 执行 | 用户中断后恢复 |

---

## Context Reset 设计要点

### 1. 触发时机

| 时机判断 | 说明 |
|----------|------|
| 填充度检测 | 观察是否接近上下文限制 |
| 收尾倾向 | Claude 开始说"差不多完成了" |
| 预估时长 | 任务开始前预估是否需要 Reset |

---

### 2. Artifact 内容设计

Artifact 必须让新 Agent 无历史 context 也能理解：

| 必需内容 | 说明 |
|----------|------|
| **任务背景** | 为什么做（目标） |
| **当前状态** | 做到哪里（进度） |
| **下一步指引** | 怎么做（具体操作） |
| **关键决策** | 已做决策（避免重复决策） |
| **注意事项** | 执行过程中的注意点 |

---

### 3. 验证衔接点

新 Agent 启动后必须验证：

| 验证内容 | 说明 |
|----------|------|
| 已完成产出存在 | 文件确实存在 |
| 内容符合预期 | 产出质量可用 |
| 无断裂遗漏 | 可正常继续 |

---

## Artifact 输出结构

根据当前任务设计 Artifact：

```markdown
# Harness State Artifact

**生成时间**：{timestamp}
**任务目标**：{当前任务的目标}

## 执行进度

**当前阶段**：{阶段名}

**已完成**：
| 序号 | 任务 | 产出 | 状态 |

**待完成**：
| 序号 | 任务 | 依赖 | 预计产出 |

## 关键决策

| 决策点 | 决策内容 | 原因 |

## 下一步

**正在处理**：{当前任务摘要}
**下一步操作**：{具体指引}

## 注意事项

- {执行过程中的特殊注意点}
```

---

## 设计原则

| 原则 | 说明 | 反例 |
|------|------|------|
| Artifact 自描述 | 新 Agent 无历史 context 也能理解 | 信息不足无法接续 |
| 验证衔接点 | 新 Agent 验证已完成产出 | 假设产出存在 |
| Harness 连续性 | Reset 后继续相同机制 | Reset 后跳过 Evaluator |
| 决策已记录 | 关键决策记录在 Artifact | 新 Agent 可能重做决策 |

---

## Common Pitfalls

| Pitfall | 症状 | 原因 | 解决方案 |
|---------|------|------|----------|
| **Artifact 信息不足** | 新 Agent 无法理解上下文 | 缺少背景/状态/指引 | Artifact 必须包含：任务背景、当前状态、下一步指引 |
| **未验证衔接** | 假设已完成产出存在 | 跳过验证步骤 | 新 Agent 启动后必须验证文件存在和内容 |
| **Harness 中断** | Reset 后跳过 Evaluator/Contract | 未在 Artifact 中要求 | 明确记录需要继续的 Harness 机制 |
| **决策未记录** | 新 Agent 重复做决策 | 关键决策未写入 Artifact | 所有设计决策必须记录 |
| **Reset 时机错误** | 过早或过晚 Reset | 未监控上下文状态 | 填充度 > 70% 或出现收尾倾向时触发 |
| **Artifact 过大** | Token 消耗过多 | 记录过多细节 | 保持精简，只记录关键信息 |

---

## 与其他机制配合

| 配合机制 | Artifact 来源 |
|----------|---------------|
| **Planner** | Planner 分解结果 → 任务列表 |
| **Contract** | Contract → 验收标准 |
| **Evaluator** | Artifact 明确要求继续 Evaluator |