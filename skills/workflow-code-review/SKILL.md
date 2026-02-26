---
name: workflow-code-review
description: 代码评审。分析 git diff，spawn code-reviewer 子代理执行实际审查，识别代码缺陷、安全漏洞、边界问题，输出结构化评审意见。当用户请求 code review 或提供 git diff 时触发。
---

# 代码评审

> 调度入口：收集上下文 → spawn 子代理审查。本 Skill 不执行审查逻辑。

## 触发条件

- 用户请求 code review / 代码评审
- 用户提供 git diff 或要求审查某次提交/分支

## 工作流程

### 步骤 1：获取 diff

按优先级：用户提供的 diff → 用户指定的 commit/branch → 默认 diff 当前分支与 `main`/`master`。

用户可限定范围（特定文件、特定维度如"只看安全"），在 spawn 时作为关注点传入。

```bash
git diff --name-only <range>
git diff -U5 <range>
```

diff 为空则停止。

### 步骤 2：收集设计文档

根据变更文件路径，查找 `docs/design-docs/<module>/<feature>/` 下的 `spec.md` 和 `tasks.md`。

- **spec.md**：找到则读取；**未找到则询问用户**（是否有设计文档？是否继续无 spec 审查？）
- **tasks.md**：找到则读取；未找到则跳过

### 步骤 3：Spawn code-reviewer 子代理

所有上下文就绪后，使用 `task` 工具 spawn 子代理：

| 参数 | 值 |
|------|-----|
| subagent_name | `code-reviewer` |
| subagent_path | `workflow-code-review/code-reviewer.md` |

prompt 中传入：变更文件列表、diff 内容、spec.md 内容（如有）、tasks.md 内容（如有）、用户关注点（如有）。

子代理将自行加载编码规范、执行审查、输出结构化结果。

### 步骤 4：处理结果

直接呈现子代理输出。后续：
- 用户要求修复 → 调用 `workflow-code-generation`
- 用户要求重审 → 回到步骤 1

## 禁止

- 上下文未收集完毕就 spawn 子代理
- 在本 Skill 内执行审查逻辑
