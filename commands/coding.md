---
description: 编码阶段：PM 按 DEV-TASKS 派发，开发者按模块实现前后端代码与单元测试
argument-hint: "模块名与功能范围，例如：user-auth 实现 fr-user-auth-001~005"
---

<user_input>
$ARGUMENTS
</user_input>

---

## 前置（缺一不可）

- 设计评审（含分解评审）已通过（`design/` 完备，含 `design/DEV-TASKS.md`）。
- 项目经理已**指派**（定责任角色）并**派发**（spawn 开发者，注入自包含任务输入）。

## 流程

1. **project-manager**（持 `Agent` 工具）：按 `design/DEV-TASKS.md` 的并行组并发派发开发者子代理；prompt 自包含（任务编号/模块/功能编号/设计文档路径/验收标准/输出位置/报告要求）。
2. **backend-developer / ui-developer**：按派发任务编码 + 单测；完成后**更新看板并向 PM 报告**（`TASK-DONE` / `TASK-BLOCKED`，格式见各自 agent 定义 §3.1）。
3. **project-manager**：核对报告（越界检查 + 单测证据 + 看板更新）→ 看板置 `待评审` → 组织代码评审（code-reviewer）。

## 开发者要求

1. 读取 `rules/team-charter.md` 与派发任务指定的设计文档。
2. 确认任务对应的功能编号与验收标准；**不确定先问用户**。
3. 先给出实现计划（文件清单）并与用户确认。
4. 编码，遵循**最简代码原则**；后端不做用户交互部分，前端通过 INTERFACE.md 调接口。
5. 编写并运行单元测试（覆盖正常/边界/异常）。
6. 完成后更新看板 + 向 PM 报告，提交代码评审。

## 禁止

- 禁止修改设计/需求文档。
- 禁止未通过自测就提交评审。
- 禁止开发者越权 spawn 其他代理（派发权唯一归 PM）。
- 禁止完成不报告、报告不带证据。
