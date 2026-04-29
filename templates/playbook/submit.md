# PLAYBOOK — 提交与验证

> 覆盖场景：代码完成 → 提交前检查 → commit → 后续验证
> 核心工具：scripts/verify.sh（自动化大部分检查）

---

## 提交前检查清单

```
□ Step 1: 运行自动化验证（R2）
  命令: ./scripts/verify.sh
  目的: 自动检查 R6/R9/R10/R13

□ Step 2: 手动确认清单
  □ 文档是否已同步？（R6）— 检查 CHANGELOG/相关 .md 是否已更新
  □ 是否有遗留问题未标记？（R9）— grep -r "TODO\|FIXME\|HACK" src/
  □ 需求来源是否已标记？（R10）— 检查 commit message
  □ 是否误建新文件？（R13）— 确认没有新的 P0_*.md

□ Step 3: Commit
  格式: `[来源] 描述 (#关联issue)`
  示例: `[user] 修复视频上传进度条 (#42)`

□ Step 4: 提交后验证（关键！）
  □ CI是否通过？
  □ 部署后功能是否正常？
  □ 无回归？（对比基线）
```

---

## Commit Message 模板

```
[来源] 简短描述

- 变更点1
- 变更点2
- 遗留问题: TODO/KNOWN-ISSUE（如有）

关联: incident#N / issue#N（如有）
```

## 回归检查速查

```bash
# 快速回归测试
pytest tests/ -x -q --tb=short

# 检查是否有 TODO 遗留
grep -rn "TODO\|FIXME\|HACK\|XXX" src/ --include="*.py"

# 检查文档同步状态
git diff --name-only HEAD~1 | grep -E "\.(md|rst|txt)$"
```
