---
name: harness-design
description: Design quality assurance mechanisms for any task. Use when user invokes '/harness-design', asks for "harness engineering", "Planner", "Evaluator", "Sprint Contract", "Context Reset", or mentions "task reliability", "quality control", "make task controllable" for code development, skill development, refactoring, content generation, or presentation tasks.
---

# Harness Design

为任何任务设计质量保障机制，确保可控、可靠。

---

## 核心流程

```
触发 → 5维度诊断 → 决策表定位 → 动态设计机制 → 输出选择
```

---

## 5维度诊断

快速判断当前任务特征，定位所需机制。

| 维度 | 判断依据 | 等级划分 |
|------|----------|----------|
| **复杂度** | 改动文件数、模块数、依赖关系 | 简单 / 中等 / 复杂 |
| **主观性** | 是否有二元验证标准 vs 需人为判断 | 客观 / 半主观 / 高主观 |
| **验收清晰度** | 用户是否给出可验证的交付标准 | 清晰 / 模糊 / 缺失 |
| **任务时长** | 预计操作步骤数、文件处理量 | 短 / 中 / 长 |
| **迭代预期** | 用户是否要求多方案或暗示需打磨 | 单次 / 需打磨 / 持续 |

**诊断输出**：填写上述表格，得出任务特征画像。

---

## 决策表

根据诊断结果，定位所需机制。

| 机制 | 触发条件 | 详细方法论 |
|------|----------|------------|
| **Planner** | 复杂度 ≥ 中等 | [planner.md](./references/mechanisms/planner.md) |
| **Evaluator** | 主观性 ≥ 半主观 或 任务重要 | [evaluator.md](./references/mechanisms/evaluator.md) |
| **Sprint Contract** | 验收清晰度 ≤ 模糊 | [sprint-contract.md](./references/mechanisms/sprint-contract.md) |
| **Context Reset** | 任务时长 ≥ 长 | [context-reset.md](./references/mechanisms/context-reset.md) |
| **G-E 循环** | 迭代预期 ≥ 需打磨 | 见 evaluator.md |

**重要任务强制规则**：生产环境、安全、资金相关 → 必须启用 Evaluator

---

## 动态设计机制

定位所需机制后，**根据当前具体任务**设计各机制参数：

| 机制 | 设计内容 | 参考 |
|------|----------|------|
| **Planner** | 分解策略、子任务边界、验收标准 | planner.md 方法论 |
| **Evaluator** | 评估维度、评分标准、校准方式 | evaluator.md 方法论 |
| **Contract** | 协商重点、验收项、边界定义 | sprint-contract.md 方法论 |
| **Context Reset** | 触发时机、Artifact 内容 | context-reset.md 方法论 |

**关键**：不预置场景模板，而是在执行时**动态分析**当前任务来设计。

---

## 输出选择

完成设计后，询问用户：

| 形式 | 产出 |
|------|------|
| **仅参考建议** | 设计建议文档 |
| **立即执行** | Claude 进入 Harness 循环执行 |
| **详细咨询** | 深入讨论某个机制 |

---

## 纠正模式

若任务进行中触发，先诊断问题：

| 问题表现 | 原因 | 处理 |
|----------|------|------|
| 输出质量虚高 | 自我评估偏差 → 添加/加强 Evaluator |
| 反复修正同一问题 | 理解偏差 → 添加 Contract 重新协商 |
| 提前结束/收尾倾向 | 上下文焦虑 → 检查是否需要 Context Reset |
| 任务执行混乱 | 缺少分解 → 添加 Planner 重新规划 |

---

## 诊断示例

见 [diagnosis-examples.md](./references/examples/diagnosis-examples.md)