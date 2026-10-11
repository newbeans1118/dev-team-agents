---
description: 发布阶段：全新 worktree 构建、打包、部署验证并标记测试版/生产版
argument-hint: "版本号或范围，例如：v1.0.0"
---

<user_input>
$ARGUMENTS
</user_input>

---

委派 **release-engineer** agent 执行**产品发布**。

## 要求

1. 读取 `rules/team-charter.md`、`design/`、`test/report/`。
2. 用 `releasing` 技能：
   - **全新 worktree**：从 git 拉取分支到独立 worktree，从头构建
   - 编译 → 打包 → 归档到 `release/artifact/`
   - 部署验证（冒烟测试），记录命令与结果
3. 用 `doc-writing` 产出 `release/manual-release.md`、`release/manual-install.md`。
4. 标记发布状态：
   - **测试版**：未四方联合确认
   - **生产版**：经产品经理、测试经理、架构师、代码评审员**联合确认**后，打 tag

## 禁止

- 禁止在脏工作区直接构建。
- 未经四方联合确认，禁止标记为生产版。
