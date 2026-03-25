# MESIS-Bench 设计规范（草案）

## 1. 背景与目标

### 1.1 背景

MESIS-Bench 用于评测大模型在以下能力上的表现：

- Interpreter 构建能力（`eval.scm`）
- Meta-circular evaluation
- 多层执行（multi-layer eval）
- 跨语言执行（Scheme / SQL / VM / DSL）
- 系统能力（I/O routing / error handling）
- 语义鲁棒性（adversarial cases）

现有 benchmark 的主要不足：

- 仅关注函数级代码生成
- 缺乏执行语义测试
- 缺乏 multi-layer / self-hosting
- 缺乏跨语言能力评测
- 缺乏统一自动验证机制

### 1.2 核心设计原则

MESIS-Bench 采用 **Test-Case-Centric Design**。

关键原则：

> Execution semantics are encoded directly in test cases, not constructed dynamically by the pipeline.

含义：

- pipeline 不构造执行链
- test case 完整描述执行流程
- `eval.scm` 仅负责解释输入描述的执行流程

---

## 2. 系统角色定义（重要）

### 2.1 `eval.scm`（被测对象）

由模型生成，必须实现：

- 解析输入中的执行指令
- 根据 `execution_type` 调用对应解释逻辑
- 支持：
  - Scheme eval
  - SQL execution
  - VM execution
  - DSL execution
- 支持 multi-layer execution
- 支持输入转发（I/O routing）

### 2.2 `interp.py`（Ground Truth）

由 pipeline 自动生成，仅用于：

- 执行 test case
- 生成标准输出（ground truth）
- 作为 validator 的执行引擎

重要约束：

- `interp.py` 必须支持与 `eval.scm` 相同的执行语义
- `interp.py` 必须覆盖所有 Feature Spec 定义能力

### 2.3 Validator

唯一评测组件。

输入：

- `eval.scm`（模型生成）
- `test_cases`

输出：

- pass / fail

---

## 3. Benchmark 任务分类（11类）

- A: Core Evaluation
- B: Functional Semantics
- C: Control Flow & Recursion
- D: Meta-Evaluation
- E: Self-hosting
- F: IO & Runtime
- G: Error Handling
- H: Adversarial
- I: SQL Interpreter
- J: VM / Bytecode
- K: DSL Definition

---

## 4. Task Schema

```json
{
  "task_id": "mesis_xxxxxx",
  "category": "A-K",
  "difficulty": 1-8,
  "description": "...",
  "constraints": [
    "必须生成 eval.scm",
    "必须通过所有 test_cases",
    "输出必须与 ground truth 完全一致"
  ],
  "files": {
    "interp.py": "...",
    "resources": "optional (csv / bytecode / dsl spec)"
  },
  "test_cases": [
    {
      "input": "...",
      "output": "..."
    }
  ],
  "metadata": {
    "execution_model": "scheme/sql/vm/dsl",
    "eval_layers": 2,
    "cross_language": true,
    "features": [],
    "adversarial": [],
    "io_mode": "batch|stream",
    "error_mode": "strict|tolerant"
  }
}
```

---

## 5. Test Case 语义定义（核心）

### 5.1 输入格式

统一格式：

```text
<execution_type> <program_or_resource>
<optional next layer>
...
<program input>
```

`execution_type ∈ {scheme, sql, vm, dsl}`。

### 5.2 执行规则（必须明确）

`eval.scm` 必须按如下规则执行：

1. 逐行读取输入。
2. 第一行确定执行类型（`execution_type`）。
3. 若下一行仍为 `execution_type`，则递归执行（multi-layer）。
4. 否则，将剩余输入作为 program input。
5. 最终执行 program 并输出结果。

### 5.3 示例

单层：

```text
scheme program.scm
(+ 7 8)
```

多层：

```text
scheme eval.scm
scheme program.scm
(+ 7 8)
```

SQL：

```text
sql program.sql
table.csv
```

Cross-language：

```text
scheme eval.scm
sql program.sql
table.csv
```

---

## 6. Pipeline 架构

```text
pipeline/
  generators/
    feature_spec.py
    scheme_generator.py
    sql_generator.py
    vm_generator.py
    dsl_generator.py
    testcase_generator.py
  executor/
    interp_builder.py
  validator/
    validate.py
  dataset/
    build.py
  outputs/
    mesis_1000.jsonl
```

---

## 7. 数据生成流程（逻辑闭环）

### Step 1: 采样任务类别 + difficulty

- 从 A–K 中采样
- 确定 difficulty（L1–L8）

### Step 2: 生成 Feature Spec（能力规格）

定义：

```json
{
  "execution_model": "...",
  "features": [],
  "eval_layers": 1-3,
  "cross_language": true,
  "io_mode": "batch|stream",
  "error_mode": "strict|tolerant"
}
```

作用：

- 约束 `interp.py`
- 约束 test case 生成

### Step 3: 构建 `interp.py`（Ground Truth Interpreter）

必须：

- 完整实现 Feature Spec
- 支持所有 `execution_type`
- 支持 multi-layer execution
- 支持 deterministic 输出

保证：

- `interp.py` 的能力 ≥ test cases 使用的能力

### Step 4: 生成 Test Cases（核心步骤）

#### Step 4.1: 生成 program / resource

- scheme → `program.scm`
- sql → `program.sql`
- vm → `program.vm`
- dsl → `program.dsl`
- sql → `table.csv`

#### Step 4.2: 构造输入结构

必须编码：

- `execution_type`
- multi-layer 结构
- cross-language 调用

#### Step 4.3: 注入复杂性（受控）

仅允许：

- 不超出 Feature Spec
- 增加组合复杂度

#### Step 4.4: 生成 ground truth

使用 `interp.py` 执行：

- input → output

#### Step 4.5: 构造 test cases

- 每个 task 必须包含多个 test cases

### Step 5: Validator

验证逻辑：

```python
for case in test_cases:
    run eval.scm
    compare output
```

### Step 6: 写入 dataset

输出：

- `train.jsonl`
- `test.jsonl`
- `metadata.json`

---

## 8. Difficulty 定义

- L1: 基础 eval
- L2: 函数 / 查询
- L3: closure / aggregation
- L4: recursion
- L5: multi-layer
- L6: self-hosting
- L7: cross-language
- L8: system + adversarial

---

## 9. 关键一致性约束（非常重要）

- `interp.py` 必须覆盖所有 test case 能力
- test case 不允许超出 Feature Spec
- `eval.scm` 必须能解析 `execution_type`
- 所有执行路径必须 deterministic
- multi-layer 语义必须一致

---

## 10. Benchmark 本质

MESIS-Bench 本质是：

> 通过 test cases 描述执行语义，并评测模型是否能构建正确 interpreter。

## 11. 一句话总结

MESIS-Bench 是一个：

> 以 test case 为中心、支持多语言与多层执行语义的解释器构建 benchmark。
