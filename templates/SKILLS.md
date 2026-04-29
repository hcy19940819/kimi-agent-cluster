# SKILLS — 技能注册表

> 索引文件：只列有什么技能和何时触发，不存技能内容。
> 技能内容存放于各自文件路径。

---

## 技能列表

| 技能名称 | 触发场景 | 文件路径 |
|---------|---------|---------|
| todo-manager | 任务管理/待办事项处理 | `.agents/skills/todo-manager/SKILL.md` |
| code-review | 代码审查/PR review | `.agents/skills/code-review/SKILL.md` |
| debug-tracer | 调试追踪/死循环检测 | `.agents/skills/debug-tracer/SKILL.md` |
| doc-sync | 文档同步检查 | `.agents/skills/doc-sync/SKILL.md` |
| test-runner | 测试执行/回归检查 | `.agents/skills/test-runner/SKILL.md` |
| git-assistant | Git操作/提交辅助 | `.agents/skills/git-assistant/SKILL.md` |
| security-scan | 安全检查/敏感信息扫描 | `.agents/skills/security-scan/SKILL.md` |

## 自动识别关键词

| 关键词 | 触发技能 |
|--------|---------|
| "TODO"/"待办"/"task" | todo-manager |
| "review"/"审查"/"PR" | code-review |
| "debug"/"bug"/"死循环"/"卡住" | debug-tracer |
| "文档"/"同步"/"CHANGELOG" | doc-sync |
| "test"/"测试"/"回归" | test-runner |
| "git"/"commit"/"push"/"merge" | git-assistant |
| "安全"/"密码"/"密钥"/"泄露" | security-scan |
