# 迁移实施指南

> 从旧架构到新架构的完整迁移步骤。
> 预计时间：4-7天

---

## Phase 1: 基础搭建（第1天）

### Step 1.1: 创建目录结构

```bash
# 创建新目录
mkdir -p AI_Hcy/playbook
mkdir -p AI_Hcy/scripts
mkdir -p AI_Hcy/archive/rules
mkdir -p AI_Hcy/archive/incidents/2024-01
mkdir -p AI_Hcy/templates

# 标记旧文件（不删除，先归档）
mkdir -p AI_Hcy/archive/legacy
git mv AGENTS.md archive/legacy/AGENTS.md.old
git mv MEMORY.md archive/legacy/MEMORY.md.old
git mv TODO.md archive/legacy/TODO.md.old
```

### Step 1.2: 创建核心文件

按顺序创建以下文件（使用提供的模板）：

| 顺序 | 文件 | 模板来源 | 关键内容 |
|------|------|---------|---------|
| 1 | META.md | templates/META.md | 5条元规则+15条执行规则 |
| 2 | CONTEXT.md | templates/CONTEXT.md | 项目信息+当前会话 |
| 3 | SKILLS.md | templates/SKILLS.md | 技能索引表 |
| 4 | playbook/start.md | templates/playbook/start.md | 3步启动流程 |
| 5 | playbook/develop.md | templates/playbook/develop.md | 开发工作流 |
| 6 | playbook/submit.md | templates/playbook/submit.md | 提交验证 |
| 7 | playbook/incident.md | templates/playbook/incident.md | 事故统一记录 |

### Step 1.3: 从旧文件提取信息

```bash
# 从 MEMORY.md 提取项目信息 → CONTEXT.md #Project
# 从 MEMORY.md 提取硬限制 → CONTEXT.md #Constraints（最多5条）
# 从 AGENTS.md 提取红线 → META.md R4/R5/R6
# 从所有 P0_*.md 提取记录 → playbook/incident.md 活跃事故表

# 提取脚本（参考）
echo "=== 从旧文件提取关键信息 ==="
echo ""
echo "--- AGENTS.md 关键规则 ---"
grep -E "^(1\.|2\.|3\.)" archive/legacy/AGENTS.md.old 2>/dev/null || echo "(已处理)"
echo ""
echo "--- MEMORY.md 硬限制 ---"
grep -A20 "硬限制\|限制\|constraint" archive/legacy/MEMORY.md.old 2>/dev/null | head -30 || echo "(需手动提取)"
echo ""
echo "--- P0 文件列表 ---"
ls projects/video_management/P0_*.md 2>/dev/null || echo "(已归档)"
```

---

## Phase 2: 自动化脚本（第2天）

### Step 2.1: 编写 pre-flight.sh

```bash
#!/bin/bash
# 保存到: scripts/pre-flight.sh
# 功能: 会话启动自检（R1, R11, R12）

echo "=== Pre-flight Check ==="
ERRORS=0

# R11: 验证 META.md 存在
[ -f META.md ] || { echo "FATAL: META.md 缺失（R11:单一来源）"; exit 1; }
echo "[PASS] META.md 存在"

# R12: 验证 CONTEXT.md 存在
[ -f CONTEXT.md ] || { echo "WARN: CONTEXT.md 缺失，使用模板"; cp templates/CONTEXT.md CONTEXT.md; }
echo "[PASS] CONTEXT.md 存在"

# 检查 CONTEXT.md 大小 < 1KB
SIZE=$(wc -c < CONTEXT.md)
[ $SIZE -lt 1024 ] || echo "WARN: CONTEXT.md ${SIZE}B > 1KB 上限"

# 输出当前上下文
echo ""
echo "=== 当前上下文 ==="
grep -A10 "#Session" CONTEXT.md 2>/dev/null || echo "(无会话上下文，请在 #Session 中设置)"

echo ""
echo "=== Pre-flight 完成 ==="
```

赋予执行权限：
```bash
chmod +x scripts/pre-flight.sh
```

### Step 2.2: 编写 verify.sh

```bash
#!/bin/bash
# 保存到: scripts/verify.sh
# 功能: 提交前验证（R2, R6, R9, R10, R13）

COMMIT_MSG_FILE="${1:-}"
echo "=== Submit Verification ==="
ERRORS=0
WARNINGS=0

# R13: 检查是否新建了 P0_ 文件
NEW_P0=$(git diff --cached --name-only --diff-filter=A 2>/dev/null | grep "^P0_" | wc -l)
if [ "$NEW_P0" -gt 0 ]; then
    echo "[FAIL R13] 禁止新建 P0_*.md 文件，请写入 playbook/incident.md"
    ERRORS=$((ERRORS+1))
else
    echo "[PASS R13] 无新建 P0_ 文件"
fi

# R6: 检查文档同步
CODE_CHANGES=$(git diff --cached --name-only 2>/dev/null | grep -cE "\.(py|js|ts|java|go|rs|cpp|c|h)$" || true)
DOC_CHANGES=$(git diff --cached --name-only 2>/dev/null | grep -cE "\.(md|rst|txt)$" || true)
if [ "$CODE_CHANGES" -gt 0 ] && [ "$DOC_CHANGES" -eq 0 ]; then
    echo "[WARN R6] 代码变更($CODE_CHANGES files)未同步文档"
    WARNINGS=$((WARNINGS+1))
else
    echo "[PASS R6] 文档同步检查通过"
fi

# R9: 检查遗留标记
if git diff --cached -G "TODO|FIXME|HACK" --name-only 2>/dev/null | grep -q .; then
    TODO_FILES=$(git diff --cached -G "TODO|FIXME|HACK" --name-only 2>/dev/null | tr '\n' ' ')
    echo "[WARN R9] 遗留标记待处理: $TODO_FILES"
    WARNINGS=$((WARNINGS+1))
else
    echo "[PASS R9] 无新增遗留标记"
fi

# R10: 检查 commit message 标记
if [ -n "$COMMIT_MSG_FILE" ] && [ -f "$COMMIT_MSG_FILE" ]; then
    if ! grep -qE '\[(user|tech-debt|incident|assumed)\]' "$COMMIT_MSG_FILE" 2>/dev/null; then
        echo "[WARN R10] Commit message 建议标记需求来源 [user/tech-debt/incident#N/assumed]"
        WARNINGS=$((WARNINGS+1))
    else
        echo "[PASS R10] 需求来源已标记"
    fi
fi

# R4: 检查文件数是否超标
FILE_COUNT=$(find . -maxdepth 2 -name "*.md" -not -path "./archive/*" -not -path "./.git/*" -not -path "./templates/*" | wc -l)
[ $FILE_COUNT -le 10 ] || echo "[WARN] 规范文件数($FILE_COUNT)建议≤10"

echo ""
if [ $ERRORS -eq 0 ]; then
    echo "=== 验证通过 (WARNINGS: $WARNINGS) ==="
    exit 0
else
    echo "=== 验证失败 (ERRORS: $ERRORS, WARNINGS: $WARNINGS) ==="
    exit 1
fi
```

赋予执行权限：
```bash
chmod +x scripts/verify.sh
```

### Step 2.3: 配置 Git Hook（可选但推荐）

```bash
# 配置 verify.sh 为 pre-commit hook
cat > .git/hooks/pre-commit << 'EOF'
#!/bin/bash
# Git pre-commit hook: 自动运行验证
./scripts/verify.sh "$1" || exit 1
EOF
chmod +x .git/hooks/pre-commit
```

---

## Phase 3: 数据迁移（第3天）

### Step 3.1: 合并 P0 文件

```bash
# 将所有 P0_*.md 内容迁移到 playbook/incident.md
# 格式：每条事故一个表格行

echo "=== 迁移 P0 文件到 incident.md ==="

for f in projects/video_management/P0_*.md; do
    [ -f "$f" ] || continue
    BASENAME=$(basename "$f" .md)
    echo "处理: $BASENAME"
    
    # 提取标题和关键内容（简化处理）
    TITLE=$(head -1 "$f" | sed 's/#//g' | xargs)
    
    # 追加到 incident.md 历史归档
    cat >> playbook/incident.md << EOF

<!-- 从 $BASENAME.md 迁移 -->
- $BASENAME: $TITLE — 已归档 — 迁移于 $(date +%Y-%m-%d)
EOF
    
    # 移动旧文件到归档
    git mv "$f" "archive/incidents/2024-01/"
done

echo "P0 文件迁移完成"
```

### Step 3.2: 归档旧文件

```bash
# 移动旧文件到 archive/legacy/
# 保留至少30天后再删除

cat > archive/legacy/README.md << 'EOF'
# 旧文件归档

这些文件已被新架构替代，保留30天作为过渡参考。

| 旧文件 | 新位置 | 替代日期 |
|--------|--------|---------|
| AGENTS.md | META.md + playbook/ | [date] |
| MEMORY.md | CONTEXT.md + playbook/ + META.md | [date] |
| TODO.md | 外部工具 (GitHub Issues) | [date] |
| P0_*.md | playbook/incident.md | [date] |

计划在 [date+30天] 删除此目录。
EOF
```

### Step 3.3: 验证新文件结构

```bash
echo "=== 新文件结构验证 ==="
echo ""
echo "核心文件:"
for f in META.md CONTEXT.md SKILLS.md; do
    SIZE=$(wc -c < "$f" 2>/dev/null || echo "0")
    STATUS=$([ "$SIZE" -lt 2048 ] && echo "✅" || echo "⚠️")
    printf "  %s: %d B %s\n" "$f" "$SIZE" "$STATUS"
done

echo ""
echo "场景手册:"
for f in playbook/*.md; do
    SIZE=$(wc -c < "$f" 2>/dev/null || echo "0")
    STATUS=$([ "$SIZE" -lt 3072 ] && echo "✅" || echo "⚠️")
    printf "  %s: %d B %s\n" "$(basename "$f")" "$SIZE" "$STATUS"
done

echo ""
echo "自动化脚本:"
for f in scripts/*.sh; do
    [ -f "$f" ] && printf "  %s: ✅ 可执行\n" "$(basename "$f")" || printf "  %s: ❌ 缺失\n" "$(basename "$f")"
done

echo ""
echo "=== 文件总数 ==="
find . -maxdepth 2 -name "*.md" -not -path "./archive/*" -not -path "./.git/*" -not -path "./templates/*" | wc -l
echo "(目标: ≤8)"
```

---

## Phase 4: 试运行与调整（第4-7天）

### Step 4.1: 试运行清单

```
Day 1:
[ ] 使用新启动流程（start.md 3步）
[ ] 检查 pre-flight.sh 是否正常运行
[ ] 确认 CONTEXT.md #Session 更新正常

Day 2-3:
[ ] 使用 develop.md 进行开发
[ ] 验证 R4/R5/R6 是否自然遵守
[ ] 测试 verify.sh 在提交前拦截

Day 4-5:
[ ] 处理一个真实事故（使用 incident.md）
[ ] 验证不新建 P0_ 文件
[ ] 测试经验记录流程（不自动升级规则）

Day 6-7:
[ ] 30天规则review预演
[ ] 收集反馈
[ ] 微调 playbook 细节
```

### Step 4.2: 常见问题处理

**Q: AI 还是违反规则怎么办？**
A: 检查该规则是否已脚本化。如果规则只能靠人记→脚本化它。如果不能脚本化→可能是坏规则，考虑删除。

**Q: 需要新增规则怎么办？**
A: 
1. 先写入 incident.md 经验库
2. 经过30天冷却
3. 通过三问检查
4. 试运行2周
5. 审批后替换旧规则（MR5）

**Q: CONTEXT.md 又超过1KB了？**
A:
1. 检查是否有信息可以脚本化
2. 淘汰旧的约束
3. 检查是否有"为什么"的解释文字（应删除）

**Q: 外部TODO工具无法访问？**
A: CONTEXT.md #Session 可以临时记录当前任务（1-2句话），但不应积累。

---

## Phase 5: 清理（第30天）

```bash
# 删除旧归档（确认新架构稳定运行30天后）
rm -rf archive/legacy/

# 清理已关闭的事故记录（可选）
# 将 incident.md 中状态=closed的记录移到 archive/incidents/
```

---

## 迁移检查清单

```
□ Phase 1 完成
  □ 目录结构创建
  □ 7个核心文件创建
  □ 旧文件标记归档

□ Phase 2 完成
  □ pre-flight.sh 编写并测试
  □ verify.sh 编写并测试
  □ Git hook 配置（可选）

□ Phase 3 完成
  □ P0 文件合并到 incident.md
  □ 旧文件移入 archive/
  □ 文件结构验证通过

□ Phase 4 完成
  □ 7天试运行完成
  □ 问题记录并解决
  □ playbook 细节调整

□ Phase 5 完成
  □ 30天稳定运行确认
  □ 旧归档清理
  □ 团队培训完成
```

---

*迁移完成后，新架构应持续运行，核心规则 ≤ 15 条，规范文件 ≤ 8 个。*
