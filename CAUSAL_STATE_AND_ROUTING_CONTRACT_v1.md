# CAUSAL_STATE_AND_ROUTING_CONTRACT_v1

## 因果状态与路由优先级维护契约 v1

---

## 1. 目标

本契约用于约束模型在长线任务会话中的状态维护行为。模型不得仅依赖会话正文历史承载全部任务状态，而应在会话运行期间，于容器内持续维护一份结构化状态，用于保存：

1. 已确认结论
2. 结论成立的最小因果结构
3. 结论之间的依赖、覆盖与失效关系
4. 当前问题下的信息路由优先级与展开优先级
5. 状态变更历史

本契约的目标不是记录完整思维链，而是维护一份足以支撑后续推理持续对齐的最小结构化状态。

---

## 2. 适用范围

本契约仅约束**单会话 / 单容器生命周期内**的状态维护行为。
跨会话状态延续不由本契约直接保证，但本契约要求状态必须具备可导出、可导入、可增量更新的形式，以支持后续跨会话同步。

---

## 3. 基本原则

### 3.1 不保存完整思维链

模型不得将完整思维过程原样落盘。
仅允许保存结论可被后续推理继续复用所需的最小因果结构。

### 3.2 结论必须带最小因果

任何进入状态库的结论，不得只保存 `statement`。
至少必须同时保存：

- `why`
- `evidence`
- `scope`
- `assumptions`

### 3.3 路由与展开分离

优先级不得混成单一“权重”。
至少应拆分为：

- **route priority**：是否优先进入当前推理
- **expand priority**：进入后展开到什么层级

### 3.4 动态问题，动态路由

路由与展开优先级不是长期固定真值。
它们必须围绕“当前问题”动态计算或动态分级。

### 3.5 状态更新优先于正文复述

当模型在新一轮推理中得到新的有效结论、发现旧结论失效、或识别出新的依赖关系时，应先更新状态，再组织正文输出。
正文不得简单重复旧结论块，而不反映状态变更。

---

## 4. 容器内状态目录结构

建议维护以下目录：

```text
/state/
  index.yaml
  facts/
    F0001.yaml
    F0002.yaml
    ...
  routing/
    current_query.yaml
    route_plan.yaml
  history/
    changelog.md
    snapshots/
  exports/
```

---

## 5. 状态文件定义

### 5.1 index.yaml

用于维护全局索引。

建议字段：

```yaml
project: <string>
session_id: <string>
version: v1
active_topics:
  - <topic_1>
  - <topic_2>
current_focus: <string>
fact_count: <int>
last_update: <timestamp>
```

### 5.2 单条因果结构文件 facts/Fxxxx.yaml

每个结论一个文件。最小 schema 如下：

```yaml
id: F0001
statement: <结论本身>
why: <最小原因说明>
evidence:
  - type: log|code|experiment|summary|observation
    ref: <引用或定位信息>
scope: <适用范围>
assumptions:
  - <前提1>
  - <前提2>
depends_on:
  - F0003
  - F0007
invalidates:
  - F0002
supersedes:
  - F0004
confidence: high|medium|low
status: active|superseded|invalidated|tentative
tags:
  - <topic>
  - <module>
created_at: <timestamp>
updated_at: <timestamp>
```

---

## 6. 路由优先级机制

第一版不要求连续分数。
第一版采用**离散分级制**。

### 6.1 路由等级 route_grade

```text
A = 当前问题核心，遗漏会直接影响结论
B = 高相关，应进入主推理
C = 相关，但可延后
D = 弱相关，仅作背景
E = 当前基本无关，但保留索引
F = 当前不进入本轮推理
```

### 6.2 展开等级 expand_grade

```text
A = 展开到 statement + why + evidence
B = 展开到 statement + why
C = 仅 statement
D = 仅索引，不展开
```

---

## 7. 当前问题上下文文件

`/state/routing/current_query.yaml`

```yaml
query: <当前问题>
task_type: analysis|design|debug|verification|summary
goal: <当前轮目标>
constraints:
  - <约束1>
  - <约束2>
policy_profile:
  read_order: conclusion_first|causal_first
  evidence_threshold: low|medium|high
  allow_empirical_assumption: true|false
  exploration_mode: conservative|balanced|exploratory
```

---

## 8. 路由计划文件

`/state/routing/route_plan.yaml`

对本轮候选 facts 做分级：

```yaml
query: <当前问题>
selected_facts:
  - id: F0001
    route_grade: A
    expand_grade: B
    reason: 当前问题直接依赖该结论，且需展开 why 判断后续是否仍成立
  - id: F0007
    route_grade: B
    expand_grade: C
    reason: 与当前问题高相关，但当前只需结论，不需证据
  - id: F0012
    route_grade: D
    expand_grade: D
    reason: 仅作背景，不参与主推理
generated_at: <timestamp>
```

---

## 9. 状态更新触发条件

满足以下任一条件时，模型必须更新因果状态：

1. 产生新的有效结论
2. 旧结论被新证据推翻
3. 发现新的 `depends_on` / `invalidates` / `supersedes` 关系
4. 当前问题切换，导致 `route_grade` / `expand_grade` 发生显著变化
5. 发现先前结论的 `scope` 或 `assumptions` 需要修正

---

## 10. 每轮工作流程

每一轮长线任务，模型应遵循以下流程：

### Step 1：读取当前问题

写入或更新 `current_query.yaml`

### Step 2：选择候选因果结构

从 `/state/facts/` 中选择与当前问题可能相关的 facts。

### Step 3：生成路由计划

为候选 facts 赋予：

- `route_grade`
- `expand_grade`
- `reason`

写入 `route_plan.yaml`

### Step 4：执行本轮推理

仅使用 `route_grade` 为 A/B 的核心项，以及必要的 C 级背景项；
并按 `expand_grade` 决定展开深度。

### Step 5：写回新增状态

若产生新结论或关系变化，更新对应 fact 文件；
必要时新增新的 fact 文件。

### Step 6：记录变更日志

在 `history/changelog.md` 中记录：

- 新增了什么
- 推翻了什么
- 修正了什么
- 当前问题使用了哪些核心 fact

---

## 11. 正文输出约束

### 11.1 禁止无意义复述

若当前轮没有状态变化，且用户只要求增量分析，模型不得完整复述已知结论块。

### 11.2 增量优先

若用户问题建立在已知前提上，正文应优先输出：

- 新增判断
- 新增实验设计
- 新增冲突分析
- 状态变更点

### 11.3 必要时引用状态而非重铺正文

当某条结论已在状态中被明确维护时，正文可简短引用其存在，而不重新完整展开。

---

## 12. 导出要求

当需要跨会话同步时，模型应从 `/state/` 导出以下内容：

1. `index.yaml`
2. 全部 `active / tentative` 的 facts
3. 最近一次 `current_query.yaml`
4. 最近一次 `route_plan.yaml`
5. `changelog.md`

导出形式可为：

- 单一压缩包
- 单一合并 Markdown / YAML 文件
- 项目级 snapshot

---

## 13. 导入要求

新会话导入时，不应把导出内容视为完整正文历史。
导入内容应被视为：

- 因果状态基线
- 当前问题前置状态
- 路由参考状态

新会话应基于这些结构继续维护，而不是回退到只读结论列表。

---

## 14. 第一版落地原则

v1 版本优先满足以下要求：

1. 可维护
2. 可解释
3. 可导出
4. 可人工校正
5. 足以降低跨轮次和跨会话同步成本

v1 不要求：

- 自动完美评分
- 完整图数据库
- 全自动优先级学习
- 完全免人工干预

---

## 15. 说明

本契约讨论的不是传统意义上的 memory，而是一种面向长线任务的结构化状态维护机制。它的目标不是替代模型内部推理，而是在当前模型能力边界下，为长线协作提供一个更稳定、可解释、可导出、可继承的外部状态层。

从长期看，因果状态维护与任务相关性路由更像通用智能应具备的内在能力；但在当前工程现实下，将其外显为容器内可维护的结构化状态，仍然是更可控、可验证、可复用的方案。
