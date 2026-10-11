# 测试度量报告：$RUN_DESC

> 模版：TEST-METRICS.md ｜ 产出角色：test-engineer（test-rebuild 派生）｜ 存放：test/report/test-metrics-$RUN.md
> 本文件由 test-rebuild 从原始记录派生，**禁止手工编辑**；所有数字必须可从原始记录复算。

## 1. 校验结论

| 校验项 | 结果 | 说明 |
|--------|------|------|
| ID 完整性 | | 孤立用例/孤立数据/悬空引用 |
| 覆盖完整性 | | 无用例功能点 / 无数据用例 |
| 源同步 | | spec ↔ req ↔ case hash 是否一致（stale 标记） |
| 状态合法 | | 枚举/状态机/双向关联 |

## 2. 本轮概览

| 项 | 值 |
|----|-----|
| 执行轮 | RUN-xxx（trigger） |
| 环境 | env / build / commit |
| 计划配对数 | |
| 已执行 / 通过 / 失败 / 阻塞 | |

## 3. 核心指标

| 指标 | 公式 | 值 | 出处 |
|------|------|-----|------|
| 需求覆盖率 | 有≥1 active 用例的功能点 ÷ 全部 active 功能点 | | req×case |
| 用例执行率 | 已执行 PLAN 行 ÷ 计划 PLAN 行 | | PLAN vs EXEC |
| 通过率 | pass ÷ executed | | EXEC |
| 缺陷密度 | bug ÷ executed | | EXEC×BUG |
| 修复率 | verify_pass ÷ 已提交修复 | | BUG.verification |
| 重开率 | reopen ÷ closed | | BUG 状态历史 |
| 回归通过率 | regression 轮 pass ÷ executed | | RUN.trigger |
| 自动化率 | automatable 用例 ÷ 全部用例 | | CASE.automation |

## 4. 切片分析（可选维度：模块 / 优先级 / suitable_for / TD）

| 切片 | 通过率 | 失败exec | 关联缺陷 |
|------|--------|---------|---------|

## 5. 遗留风险与结论

- 遗留 P0/P1：
- 建议处置：
