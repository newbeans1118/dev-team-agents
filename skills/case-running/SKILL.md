---
name: case-running
description: 生成执行计划、执行测试、记录执行明细、提交并跟踪缺陷时使用。触发场景：需要按轮次组织测试执行、生成用例与数据配对的执行计划、执行加载脚本并跑用例、记录 EXEC 执行明细、提交 BUG 并跟踪回归、按数据集切片统计结果、提到执行测试、跑用例、回归测试、缺陷提交。
---

# 测试用例执行（case-running）

## 目的

按轮次（RUN）组织执行：**先配对（PLAN）→ 再执行（EXEC）→ 失败成单（BUG）→ 回归闭环**。执行明细只追加不改写，支撑度量复算与历史回溯。

## 执行者

`test-engineer`（PLAN 由 `test-manager` 审定后方可执行）

## 输入

- `test/plan.md`、本模块 `test/trace/case-$MOD.json`、本模块注册表分片 `test/data/trace/TD-REGISTRY-$MOD.json`（SQLite 轨：查库）
- `test/automation/`（可自动化的 script_ref）
- `release/manual-install.md`、被测软件

## 输出（按存储轨道 × 分片）

**分片规则**：执行**按 `模块 × 环境` 分片**——一个执行分片 = 一个 test-engineer 实例在一个 env 上执行一个模块的计划行；红线：**同一 env 同一时刻只允许一个执行分片**（env 是共享写资源），分片内计划行按序执行。

- **JSON 轨**（默认）：
  - `test/runs/RUN-xxx.json`（执行轮定义 + `envs` 声明 + 各分片汇总）
  - `test/runs/RUN-xxx/EXEC-PLAN-$MOD.json`（本模块执行计划，**TM 审定后生效**）
  - `test/runs/RUN-xxx/EXEC-$MOD.jsonl`（本模块执行明细，**追加式**——分片文件互不交叉，多实例并行零冲突）
  - `test/bugs/BUG-xxx.json`（缺陷单）
  - 更新本模块 `test/trace/case-$MOD.json` 的 `latest_result_ref`
- **SQLite 轨**（STORE-CONFIG.store=sqlite）：写 `test/trace/metrics.db`
  - 同构入库表：`runs` / `exec_plans` / `execs` / `bugs`（DDL 见 `templates/SQLITE-SCHEMA.sql`）；`runs.envs` 列声明环境分配
  - 并行写入由行级 INSERT 天然化解（无文件级冲突）；`execs` 表**只追加**由触发器强制（UPDATE/DELETE 直接 ABORT——库级固化"EXEC 只追加不改写"红线）
  - `exec_plans.approved` 列承载 TM 审定标记（未审定行禁止执行）
  - 更新 `cases.latest_result_ref` 列

## 存储选择（开工第一步）

读 `test/STORE-CONFIG.json`——**项目立项时已一次性选定，全程跟随；仅允许一次 json→sqlite 单向迁移**（迁移后不可逆）。
- `store=json` → 按主体流程写文件（RUN / EXEC-PLAN-$MOD / EXEC-$MOD.jsonl / BUG）。
- `store=sqlite` → 同构入库：`runs` / `exec_plans` / `execs` / `bugs`（execs 触发器强制只追加；EXEC-PLAN.approved 承载审定）。
- 配置缺失 → **BLOCKED 报项目经理**（立项时未完成存储决策，不得代为默认选轨）。**数据契约（实体/字段/关联）两轨完全一致，仅读写方式不同。**

## 执行轮类型（trigger）

| 类型 | 范围 | 触发时机 |
|------|------|---------|
| `full` | 全部 active 用例 | 首轮 / 交付前 |
| `regression` | 修复相关 + 核心路径 | 缺陷修复后 / 里程碑 |
| `smoke` | 核心冒烟集 | 每次部署后 |
| `defect-verify` | 指定 BUG 的验证用例 | 修复完成后（`defect_refs` 必填） |

## 流程（六步，顺序固定）

### 1. 建 RUN

按 trigger 确定 scope，写 `RUN-xxx.json`（含环境 build/commit——**证据**）；**声明 `envs` 数组**（本轮可用的被测环境列表）——执行分片按"模块 → env"分配，**分片数 ≤ env 数**（不足则排队，禁止两分片共用同一 env）。

### 2. 生成执行计划（PLAN，按模块分片）

- 从 RUN 范围内**本模块**的用例出发，按本模块 TD 注册表分片 `suitable_for` 推荐"用例 × 数据"配对。
- 每行 `{plan_id, case, td, expected, source}`——**expected 以文本落定在配对行**（从 TD description 语义提炼或项目补充；同一 TD 配不同用例预期不同）。
- 产出**本模块计划分片** `EXEC-PLAN-$MOD.json`；全部模块分片交 `test-manager` **审定**（跨片查覆盖率与配对合理性后合并放行）；未经审定不得执行。

### 3. 执行（本分片 = 本模块 × 分到的 env）

逐计划行执行：
1. 调用该行 TD 的 `load_entry`（加载数据；不解释内容，照单调用）。
2. 执行用例步骤。
3. **追加**一行 EXEC 到本分片 `RUN-xxx/EXEC-$MOD.jsonl`：`{exec_id, plan_ref, run, case, td, result, defect_ref, started_at, duration_ms, evidence}`。
4. 无论成败，按数据单元的清理逻辑恢复环境。

> 单行失败**继续下一行**（除非冒烟轮核心失败才中止）——保持轮次完整性。
> **env 独占**：本分片执行期间独占分到的 env；结束（含清理完成）后释放。

### 4. 提交缺陷

失败 EXEC → 建 `BUG-xxx.json`（模板 `templates/BUG.json`）：**必须回指** `{exec, run, case, td}`（度量分母的依据）；状态机 `open → in_fix → fixed → verify_pass → closed`，`verify_fail` 重开并记 `reopen_count`。

### 5. 回归验证

修复后开 `trigger=defect-verify` 的新 RUN，只跑相关配对行；**BUG 组分片**——验证范围天然隔离，可多实例并行（仍受同 env 单写者约束）；`verify_pass` 才能关闭缺陷。**验证由测试侧做，开发者不得自行关闭**。

### 6. 汇总与移交

- 汇总 `RUN.summary`（total/pass/fail/blocked，**聚合全部模块分片**）。
- 更新用例侧 `latest_result_ref` 指针（各分片实例更新自己模块的）。
- 触发 `test-rebuild` 生成度量报告（消费全部分片），交 `test-manager` 评估。

## 分片粒度决策规则

- **默认**：一模块一执行分片。
- **模块过大**（计划行 > 200）：可拆"模块 × 子域"，分片键 `$MOD-subdomain`。
- **模块过小**（计划行 < 10）：并入相邻模块分片，避免实例数爆炸（并发上限 20 同样适用）。
- **上限**：分片数 ≤ `RUN.envs` 数。

## 质量要求

- EXEC-PLAN 未经 TM 审定不得执行；EXEC **只追加不改写**。
- 每条 EXEC 必须回指计划行（plan_ref）——度量可复算的前提。
- 数据加载必须经 `load_entry`，**禁止绕过直接改环境**（保证配对与执行的对应）。
- BUG 必须回指 exec（否则修复率无分母）。
- **env 独占**：同一 env 同一时刻只允许一个执行分片；分片实例**只写本分片产物**（EXEC-$MOD.jsonl / 本模块 case 分片 / 本模块 TD）。

## 检查清单

- [ ] RUN 已建立（含环境 build/commit 与 `envs` 声明）
- [ ] 各模块 PLAN 分片已生成并经 TM 审定
- [ ] 分片与 env 分配满足"同 env 单写者"
- [ ] EXEC 只追加、逐行回指 plan_ref、落在各自分片文件
- [ ] 失败项全部建 BUG 且回指 exec
- [ ] 回归轮已跑、verify 结果已回写 BUG
- [ ] RUN.summary 已聚合全部分片、latest_result_ref 已更新
- [ ] test-rebuild 已触发

## 触发与边界

**该用**：任何一次测试执行（全量/回归/冒烟/缺陷验证）。

**不该用**：
- 设计用例 → `case-design`；生成数据 → `test-data-gen`
- **审定 PLAN** → `test-manager`
- 度量统计 → `test-rebuild`
- 修改缺陷状态为 closed → 须 verify_pass 证据，且由流程驱动，不跳步

## 常见错误

- PLAN 未审定就开跑（配对错误 = 整轮作废）。
- EXEC 改写历史行（度量失去复算基础）。
- 绕过 load_entry 手工造数据（EXEC 与 TD 的对应断裂）。
- 失败不建 BUG 或 BUG 不回指 exec（修复率分母丢失）。
- 开发者自行关闭缺陷（验证权在测试侧）。
