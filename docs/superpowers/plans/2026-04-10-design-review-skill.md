# Design Review Skill 实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 创建 design-review skill，用于系统化评审模块设计文档的完整性和一致性。

**Architecture:** 单一 SKILL.md 文件，包含 6 个检查维度、25 个检查项、执行流程和输出格式。作为流程模板指导 Claude 或人工执行评审。

**Tech Stack:** Markdown skill 文件，YAML frontmatter

---

## 文件结构

| 文件 | 职责 |
|------|------|
| `.claude/skills/design-review/SKILL.md` | 主 skill 文件，包含完整评审流程模板 |

---

### Task 1: 创建 skill 目录结构

**Files:**
- Create: `.claude/skills/design-review/` 目录

- [ ] **Step 1: 创建目录**

```bash
mkdir -p .claude/skills/design-review
```

预期：目录创建成功

- [ ] **Step 2: 验证目录存在**

```bash
ls -la .claude/skills/design-review
```

预期：目录存在且为空

---

### Task 2: 编写 SKILL.md frontmatter 和概述

**Files:**
- Create: `.claude/skills/design-review/SKILL.md`

- [ ] **Step 1: 编写 frontmatter 和概述部分**

创建文件 `.claude/skills/design-review/SKILL.md`，写入以下内容：

```markdown
---
name: design-review
description: Use when reviewing module design documents for completeness and consistency. Triggers: "review this design doc", "check if design is complete", "/design-review", analyzing module-level design documents (scheduler, executor, etc.) for structural integrity, definition completeness, terminology consistency, logical consistency, interface consistency, and design reasonability.
---

# Design Review

## Overview

**Core principle:** Systematically check design documents against 25 criteria across 6 dimensions before implementation. Catch issues early, avoid wasted effort.

**This skill provides a structured checklist approach, not automated execution.** Claude or human follows the steps manually, producing a categorized issue list.

## When to Use

Use for ANY module design document review:
- Before implementation starts
- After major design changes
- Cross-component integration review
- Pre-commit design validation

**Use ESPECIALLY when:**
- Design document references external interfaces or data structures
- Multiple modules interact in complex workflows
- New design introduces state machines or DAGs
- Design claims dependency on external components

**Don't skip when:**
- Design seems simple (simple designs have consistency issues too)
- Already started implementation (catch issues before more code)
- Under time pressure (bad designs cost more time later)

## What Gets Checked

| Dimension | Target | Mode | Items |
|-----------|--------|------|-------|
| **结构完整性** | Chapter structure complete | 规则级 | 3 |
| **定义完整性** | Each definition complete | 规则级 | 5 |
| **术语一致性** | Terms/naming unified | 规则级 | 3 |
| **逻辑一致性** | Flows/states self-consistent | 规则级 | 4 |
| **接口一致性** | Cross-doc/component interfaces match | 规则级 | 4 |
| **设计合理性** | Architecture/responsibility reasonable | 原则级 | 6 |

**规则级:** Clear matching rules, pass/fail verdict, no subjective judgment
**原则级:** Based on design principles, pass/warning verdict, requires judgment

```

预期：文件创建成功，包含 frontmatter 和概述部分

- [ ] **Step 2: 验证文件内容**

```bash
head -50 .claude/skills/design-review/SKILL.md
```

预期：输出包含 frontmatter 和概述内容

---

### Task 3: 编写检查项定义（结构完整性 + 定义完整性）

**Files:**
- Modify: `.claude/skills/design-review/SKILL.md`（追加内容）

- [ ] **Step 1: 追加结构完整性检查项**

追加以下内容到 `.claude/skills/design-review/SKILL.md`：

```markdown
---

## Dimension 1: 结构完整性（规则级）

### S-001: 必要章节完整

**检查规则:** 文档是否包含概述、详细设计章节

**判定标准:** 缺少任意必要章节 → ❌ 失败

**必要章节定义:**
- 概述章节：包含文档目的、模块架构图、职责边界表
- 详细设计章节：包含数据结构定义、API接口定义

### S-002: 模块简介完整

**检查规则:** 每个内部模块是否有职责说明和方法签名表

**判定标准:** 模块缺少职责说明或方法签名表 → ❌ 失败

### S-003: 流程图完整

**检查规则:** 关键流程是否有时序图/流程图

**判定标准:** 涉及多模块协作的流程缺少图示 → ❌ 失败

**关键流程定义:**
- 涉及3个及以上模块协作的流程
- 核心业务流程（如任务创建、状态上报）

---

## Dimension 2: 定义完整性（规则级）

### D-001: 数据结构字段完整

**检查规则:** 数据结构表格中每行是否有类型、说明列

**判定标准:** 存在字段缺少类型或说明 → ❌ 失败

**表格格式:** 字段名 | 类型 | 必填 | 说明 | 约束（可选）

### D-002: 方法签名完整

**检查规则:** 方法是否有参数类型、返回值类型

**判定标准:** 方法缺少参数类型或返回值 → ❌ 失败

### D-003: 状态枚举完整

**检查规则:** 状态枚举是否有值和说明列

**判定标准:** 状态枚举缺少值或说明 → ❌ 失败

### D-004: 接口定义完整

**检查规则:** 接口是否有路径、方法、请求/响应结构

**判定标准:** 接口缺少路径或响应结构 → ❌ 失败

### D-005: 配置参数完整

**检查规则:** 配置表是否有类型、默认值、说明

**判定标准:** 配置参数缺少默认值或说明 → ❌ 失败

**表格格式:** 参数名 | 类型 | 默认值 | 说明

```

预期：文件追加成功

- [ ] **Step 2: 验证追加内容**

```bash
grep -A 3 "Dimension 1:" .claude/skills/design-review/SKILL.md
grep -A 3 "Dimension 2:" .claude/skills/design-review/SKILL.md
```

预期：两个维度都存在

---

### Task 4: 编写检查项定义（术语一致性 + 逻辑一致性）

**Files:**
- Modify: `.claude/skills/design-review/SKILL.md`（追加内容）

- [ ] **Step 1: 追加术语一致性和逻辑一致性检查项**

追加以下内容到 `.claude/skills/design-review/SKILL.md`：

```markdown
---

## Dimension 3: 术语一致性（规则级）

### T-001: 术语命名统一

**检查规则:** 同一概念在不同位置是否使用相同名称

**判定标准:** 同一概念使用不同名称 → ❌ 失败

**常见问题:**
- task_id 与 TaskId 混用
- SubTask 与 sub_task 混用
- status 与 Status 混用

### T-002: 命名风格统一

**检查规则:** 是否混用 snake_case 和 camelCase

**判定标准:** 同类元素命名风格不一致 → ❌ 失败

### T-003: 状态名称统一

**检查规则:** 状态名称与枚举值是否匹配

**判定标准:** 状态枚举值与文字描述不一致 → ❌ 失败

---

## Dimension 4: 逻辑一致性（规则级）

### L-001: 状态流转完整

**检查规则:** 流程图/时序图中的状态是否都在枚举中定义

**判定标准:** 流程图出现未定义状态 → ❌ 失败

**提取规则:** 从 mermaid 流程图提取状态节点

### L-002: 流程节点匹配

**检查规则:** 时序图参与者是否与模块定义对应

**判定标准:** 出现未定义的模块参与者 → ❌ 失败

**提取规则:** 从 mermaid 时序图提取参与者名称

### L-003: 依赖方向正确

**检查规则:** 依赖关系图是否与职责边界表匹配

**判定标准:** 依赖关系与职责表矛盾 → ❌ 失败

### L-004: 方法调用匹配

**检查规则:** 流程中调用的方法是否在方法签名表中定义

**判定标准:** 流程调用未定义的方法 → ❌ 失败

**提取规则:** 从流程描述提取方法调用

```

预期：文件追加成功

- [ ] **Step 2: 验证追加内容**

```bash
grep -A 3 "Dimension 3:" .claude/skills/design-review/SKILL.md
grep -A 3 "Dimension 4:" .claude/skills/design-review/SKILL.md
```

预期：两个维度都存在

---

### Task 5: 编写检查项定义（接口一致性）

**Files:**
- Modify: `.claude/skills/design-review/SKILL.md`（追加内容）

- [ ] **Step 1: 追加接口一致性检查项**

追加以下内容到 `.claude/skills/design-review/SKILL.md`：

```markdown
---

## Dimension 5: 接口一致性（规则级）

**此维度需要穿透验证:** 检查跨文档引用和依赖组件源码的真实性

### I-001: 数据结构字段匹配

**检查规则:** 引用的数据结构与定义文档一致

**判定标准:** 字段名称或类型不一致 → ❌ 失败

**穿透范围:**
- docs/designs/ 目录下的数据模型文档
- docs/dependencies/ 目录下的接口文档

### I-002: 外部接口存在

**检查规则:** 声称调用的外部接口在依赖文档/源码中存在

**判定标准:** 接口不存在 → ❌ 失败

**穿透范围:**
- AgentCube SDK: `/home/xmq/projects/go/AgentCube/go-sdk`
- eval-data: `/home/xmq/projects/go/eval-data`

**验证方法:** 使用 Grep 搜索方法定义，确认存在

### I-003: 请求响应结构匹配

**检查规则:** 跨文档引用的请求响应结构一致

**判定标准:** 字段缺失或类型不同 → ❌ 失败

### I-004: 枚举值匹配

**检查规则:** 跨文档引用的状态枚举一致

**判定标准:** 枚举值或名称不同 → ❌ 失败

```

预期：文件追加成功

- [ ] **Step 2: 验证追加内容**

```bash
grep -A 10 "Dimension 5:" .claude/skills/design-review/SKILL.md
```

预期：维度 5 存在且包含穿透验证说明

---

### Task 6: 编写检查项定义（设计合理性）

**Files:**
- Modify: `.claude/skills/design-review/SKILL.md`（追加内容）

- [ ] **Step 1: 追加设计合理性检查项**

追加以下内容到 `.claude/skills/design-review/SKILL.md`：

```markdown
---

## Dimension 6: 设计合理性（原则级）

**此维度需要主观判断:** 基于设计原则评估，结果为警告而非强制失败

### R-001: 职责划分清晰

**检查原则:** 每个模块是否有单一明确的职责

**典型问题:** 一个模块承担过多职责 → ⚠️ 警告

**评估依据:** 模块职责边界表是否清晰定义「负责」和「不负责」

### R-002: 依赖方向合理

**检查原则:** 是否存在下层依赖上层、循环依赖

**典型问题:** Storage 层依赖 TaskManager → ⚠️ 警告

**评估依据:** 依赖关系图是否遵循单向依赖原则

### R-003: 错误处理覆盖

**检查原则:** 关键流程是否有错误处理说明

**典型问题:** 核心流程缺少失败处理分支 → ⚠️ 警告

**评估依据:** 关键流程是否有异常分支说明

### R-004: 并发安全设计

**检查原则:** 并发场景是否有同步机制说明

**典型问题:** 共享资源无同步机制 → ⚠️ 警告

**评估依据:** 并发相关代码是否有同步说明

### R-005: 失败恢复策略

**检查原则:** 是否有失败重试和恢复机制

**典型问题:** 失败后无重试或恢复说明 → ⚠️ 警告

**评估依据:** 是否有重试次数、重试间隔、恢复策略定义

### R-006: 可观测性设计

**检查原则:** 是否有日志、监控指标定义

**典型问题:** 缺少关键操作的可观测性说明 → ⚠️ 警告

**评估依据:** 是否有日志级别、监控指标、告警规则定义

```

预期：文件追加成功

- [ ] **Step 2: 验证追加内容**

```bash
grep -A 10 "Dimension 6:" .claude/skills/design-review/SKILL.md
```

预期：维度 6 存在且包含 6 个原则级检查项

---

### Task 7: 编写执行流程和输出格式

**Files:**
- Modify: `.claude/skills/design-review/SKILL.md`（追加内容）

- [ ] **Step 1: 追加执行流程**

追加以下内容到 `.claude/skills/design-review/SKILL.md`：

```markdown
---

## Execution Process

Follow these steps in order for each design document review:

### Step 1: Parse Document

Extract definition elements from the document:
- **数据结构:** Match tables with pattern `字段名 | 类型`
- **方法签名:** Match code blocks with `func` or method signature tables
- **状态枚举:** Match tables with pattern `值 | 名称 | 说明`
- **接口定义:** Match tables with pattern `路径 | 方法`
- **流程图:** Parse mermaid code blocks for participants and states

### Step 2: Check Structure Integrity (S-001, S-002, S-003)

Execute rule-level checks, record pass/fail for each.

### Step 3: Check Definition Completeness (D-001 ~ D-005)

Execute rule-level checks, record pass/fail for each.

### Step 4: Check Terminology Consistency (T-001, T-002, T-003)

Execute rule-level checks, record pass/fail for each.

### Step 5: Check Logical Consistency (L-001 ~ L-004)

Execute rule-level checks, record pass/fail for each.

### Step 6: Check Interface Consistency (I-001 ~ I-004)

**Penetration verification required:**
- Cross-doc: Read referenced design documents, compare field names/types
- Source-code: Use Grep tool to search for method definitions in dependency projects

### Step 7: Check Design Reasonability (R-001 ~ R-006)

Apply principles with judgment, record pass/warning for each.

### Step 8: Output Issue List

Generate categorized issue list with location references.

---

## Output Format

After completing all checks, output a report in this format:

```markdown
# 设计文档评审报告

## 评审概况
- 文档: [文档路径]
- 检查维度: [已执行的维度列表]
- 问题总数: [数量]

## 结构完整性问题 (X项)
| 检查项 | 问题 | 位置 | 说明 |
|--------|------|------|------|
| S-001 | [问题描述] | [章节位置] | [具体缺失内容] |

## 定义完整性问题 (X项)
| 检查项 | 问题 | 位置 | 说明 |
|--------|------|------|------|

## 术语一致性问题 (X项)
| 检查项 | 问题 | 位置 | 说明 |
|--------|------|------|------|

## 逻辑一致性问题 (X项)
| 检查项 | 问题 | 位置 | 说明 |
|--------|------|------|------|

## 接口一致性问题 (X项)
| 检查项 | 问题 | 位置 | 说明 |
|--------|------|------|------|

## 设计合理性警告 (X项)
| 检查项 | 警告 | 位置 | 建议 |
|--------|------|------|------|

## 通过检查项 (X项)
- [列出通过的检查项，格式: S-002 模块简介完整]
```

```

预期：文件追加成功

- [ ] **Step 2: 验证执行流程部分**

```bash
grep -A 20 "Execution Process" .claude/skills/design-review/SKILL.md
```

预期：包含 8 个步骤和输出格式

---

### Task 8: 编写 Quick Reference 和 Common Mistakes

**Files:**
- Modify: `.claude/skills/design-review/SKILL.md`（追加内容）

- [ ] **Step 1: 追加 Quick Reference 和结尾部分**

追加以下内容到 `.claude/skills/design-review/SKILL.md`：

```markdown
---

## Quick Reference

### Check Items Summary

**规则级 (19项):**
| 维度 | 检查项 | ID |
|------|--------|-----|
| 结构完整性 | 必要章节完整 | S-001 |
| 结构完整性 | 模块简介完整 | S-002 |
| 结构完整性 | 流程图完整 | S-003 |
| 定义完整性 | 数据结构字段完整 | D-001 |
| 定义完整性 | 方法签名完整 | D-002 |
| 定义完整性 | 状态枚举完整 | D-003 |
| 定义完整性 | 接口定义完整 | D-004 |
| 定义完整性 | 配置参数完整 | D-005 |
| 术语一致性 | 术语命名统一 | T-001 |
| 术语一致性 | 命名风格统一 | T-002 |
| 术语一致性 | 状态名称统一 | T-003 |
| 逻辑一致性 | 状态流转完整 | L-001 |
| 逻辑一致性 | 流程节点匹配 | L-002 |
| 逻辑一致性 | 依赖方向正确 | L-003 |
| 逻辑一致性 | 方法调用匹配 | L-004 |
| 接口一致性 | 数据结构字段匹配 | I-001 |
| 接口一致性 | 外部接口存在 | I-002 |
| 接口一致性 | 请求响应结构匹配 | I-003 |
| 接口一致性 | 枚举值匹配 | I-004 |

**原则级 (6项):**
| 维度 | 检查项 | ID |
|------|--------|-----|
| 设计合理性 | 职责划分清晰 | R-001 |
| 设计合理性 | 依赖方向合理 | R-002 |
| 设计合理性 | 错误处理覆盖 | R-003 |
| 设计合理性 | 并发安全设计 | R-004 |
| 设计合理性 | 失败恢复策略 | R-005 |
| 设计合理性 | 可观测性设计 | R-006 |

---

## Common Mistakes

### Mistake 1: Skipping penetration verification

**Wrong:** Only checking current document, not verifying external references

**Right:** Read referenced design docs, search dependency source code

### Mistake 2: Subjective judgment on rule-level items

**Wrong:** Giving warnings for terminology inconsistency

**Right:** Rule-level items are pass/fail, no interpretation needed

### Mistake 3: Missing location references

**Wrong:** Reporting "术语不一致" without specific location

**Right:** Include section/table location: "§2.1 数据结构表格"

### Mistake 4: Reviewing without reading the spec

**Wrong:** Improvising checks based on general knowledge

**Right:** Read the spec document first to understand design intent

---

## Dependency Projects

| 项目 | 路径 | 验证内容 |
|------|------|----------|
| AgentCube SDK | `/home/xmq/projects/go/AgentCube/go-sdk` | ExecutorClient 接口定义 |
| eval-data | `/home/xmq/projects/go/eval-data` | 数据管理服务接口 |
| 设计文档集 | `docs/designs/` | 跨文档数据结构引用 |
```

预期：文件追加成功

- [ ] **Step 2: 验证完整文件**

```bash
wc -l .claude/skills/design-review/SKILL.md
```

预期：文件约 200+ 行

---

### Task 9: 验证 skill 可用性

**Files:**
- Verify: `.claude/skills/design-review/SKILL.md`

- [ ] **Step 1: 检查 frontmatter 格式**

```bash
head -10 .claude/skills/design-review/SKILL.md
```

预期：包含正确的 YAML frontmatter（name, description）

- [ ] **Step 2: 检查所有维度存在**

```bash
grep "Dimension" .claude/skills/design-review/SKILL.md
```

预期：输出包含 Dimension 1-6

- [ ] **Step 3: 检查检查项数量**

```bash
grep -c "### [A-Z]-" .claude/skills/design-review/SKILL.md
```

预期：输出 25（19规则级 + 6原则级）

---

### Task 10: 提交 skill 文件

**Files:**
- Commit: skill 创建记录

- [ ] **Step 1: 验证文件完整性**

阅读完整文件确认无占位符或缺失内容：

```bash
cat .claude/skills/design-review/SKILL.md | grep -E "(TBD|TODO|待定|placeholder)"
```

预期：无匹配（无占位符）

- [ ] **Step 2: 在项目目录记录 skill 创建（可选）**

如果需要记录到项目的 git：

```bash
echo "Created .claude/skills/design-review/SKILL.md for design document review" >> ~/.claude/skills/design-review/CHANGELOG.md
```

预期：记录创建日志

---

## Self-Review Checklist

计划编写完成后，对照 spec 检查：

| Spec 章节 | 对应任务 | 状态 |
|-----------|----------|------|
| 概述 | Task 2 | ✓ |
| 检查维度 (§2) | Task 2 | ✓ |
| 结构完整性 (§3.1) | Task 3 | ✓ |
| 定义完整性 (§3.2) | Task 3 | ✓ |
| 术语一致性 (§3.3) | Task 4 | ✓ |
| 逻辑一致性 (§3.4) | Task 4 | ✓ |
| 接口一致性 (§3.5) | Task 5 | ✓ |
| 设计合理性 (§3.6) | Task 6 | ✓ |
| 执行流程 (§4) | Task 7 | ✓ |
| 输出格式 (§5.3) | Task 7 | ✓ |
| Quick Reference (附录§6.1) | Task 8 | ✓ |
| 依赖项目路径 (附录§6.2) | Task 8 | ✓ |

**占位符检查:** 无 TBD/TODO/placeholder

**类型一致性检查:** 所有检查项 ID 格式统一（S/D/T/L/I/R-XXX）