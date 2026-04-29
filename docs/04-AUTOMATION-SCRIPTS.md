# 自动化脚本完整参考

> 本文件提供3个核心脚本的完整实现和部署指南。
> 设计原则：能脚本化的规则绝不靠人记。

---

## 脚本总览

| 脚本 | 触发时机 | 覆盖规则 | 功能 |
|------|---------|---------|------|
| `pre-flight.sh` | 每次会话启动 | R1, R11, R12 | 3步自检，确认环境就绪 |
| `verify.sh` | 每次代码提交前 | R2, R6, R9, R10, R13 | 提交前验证清单 |
| `guard-rail.py` | 运行时守护 | R3, R7, R8 | 实时阻断违规行为 |

---

## pre-flight.sh — 会话启动自检

### 功能
- 验证核心文件存在（R11: 单一来源）
- 验证 CONTEXT.md 可读（R12: 上下文）
- 检查文件大小合规
- 输出当前上下文摘要

### 完整代码

```bash
#!/bin/bash
# pre-flight.sh — 会话启动强制检查
# 规则覆盖: R1, R11, R12
# 用法: ./scripts/pre-flight.sh

set -euo pipefail

META_FILE="META.md"
CONTEXT_FILE="CONTEXT.md"
PLAYBOOK_DIR="playbook"
SCRIPTS_DIR="scripts"
MAX_CONTEXT_SIZE=1024    # 1KB
MAX_META_SIZE=3072       # 3KB
MAX_PLAYBOOK_FILES=6
ERRORS=0

echo "=========================================="
echo "  Pre-flight Check (R1: 会话启动自检)"
echo "=========================================="

# ---- R11: 单一来源检查 ----
echo ""
echo "[R11] 单一来源检查..."

if [ ! -f "$META_FILE" ]; then
    echo "  ❌ FATAL: META.md 不存在（R11: 规则唯一来源）"
    ERRORS=$((ERRORS+1))
else
    META_SIZE=$(wc -c < "$META_FILE")
    echo "  ✅ META.md 存在 (${META_SIZE}B)"
    
    if [ "$META_SIZE" -gt "$MAX_META_SIZE" ]; then
        echo "  ⚠️  WARN: META.md ${META_SIZE}B > ${MAX_META_SIZE}B 上限"
    fi
fi

if [ ! -f "$CONTEXT_FILE" ]; then
    echo "  ⚠️  WARN: CONTEXT.md 不存在，使用模板创建"
    cp templates/CONTEXT.md "$CONTEXT_FILE" 2>/dev/null || touch "$CONTEXT_FILE"
else
    CONTEXT_SIZE=$(wc -c < "$CONTEXT_FILE")
    echo "  ✅ CONTEXT.md 存在 (${CONTEXT_SIZE}B)"
    
    if [ "$CONTEXT_SIZE" -gt "$MAX_CONTEXT_SIZE" ]; then
        echo "  ⚠️  WARN: CONTEXT.md ${CONTEXT_SIZE}B > ${MAX_CONTEXT_SIZE}B 上限（需清理）"
    fi
fi

# 检查 playbook 目录
if [ -d "$PLAYBOOK_DIR" ]; then
    PB_COUNT=$(find "$PLAYBOOK_DIR" -name "*.md" | wc -l)
    echo "  ✅ playbook/ 存在 (${PB_COUNT} 个手册)"
    if [ "$PB_COUNT" -gt "$MAX_PLAYBOOK_FILES" ]; then
        echo "  ⚠️  WARN: playbook 文件数 ${PB_COUNT} > ${MAX_PLAYBOOK_FILES}"
    fi
else
    echo "  ❌ playbook/ 目录不存在"
    ERRORS=$((ERRORS+1))
fi

# 检查 scripts 目录
if [ -d "$SCRIPTS_DIR" ]; then
    echo "  ✅ scripts/ 存在"
else
    echo "  ⚠️  WARN: scripts/ 目录不存在"
fi

# ---- 文件数量检查 ----
echo ""
echo "[架构] 文件数量检查..."
FILE_COUNT=$(find . -maxdepth 2 -name "*.md" -not -path "./archive/*" -not -path "./.git/*" -not -path "./templates/*" | wc -l)
echo "  规范文件数: ${FILE_COUNT} (目标: ≤8)"
if [ "$FILE_COUNT" -le 8 ]; then
    echo "  ✅ 文件数量合规"
else
    echo "  ⚠️  WARN: 文件数超标，建议归档"
fi

# ---- R12: 上下文输出 ----
echo ""
echo "[R12] 当前上下文..."
if [ -f "$CONTEXT_FILE" ]; then
    echo ""
    grep -E "^(#Session|目标:|来源:|时间:|关联:)" "$CONTEXT_FILE" 2>/dev/null || echo "  (无会话上下文)"
else
    echo "  (无 CONTEXT.md)"
fi

# ---- 结果 ----
echo ""
echo "=========================================="
if [ $ERRORS -eq 0 ]; then
    echo "  ✅ Pre-flight 通过"
    echo "=========================================="
    exit 0
else
    echo "  ❌ Pre-flight 失败 (${ERRORS} 个错误)"
    echo "=========================================="
    exit 1
fi
```

---

## verify.sh — 提交前验证

### 功能
- 检测违规新建 P0 文件（R13）
- 检查文档同步（R6）
- 检测遗留标记 TODO/FIXME（R9）
- 检查需求来源标记（R10）
- 验证核心文件完整性

### 完整代码

```bash
#!/bin/bash
# verify.sh — 提交前强制验证
# 规则覆盖: R2, R6, R9, R10, R13
# 用法: ./scripts/verify.sh [commit-msg-file]

COMMIT_MSG_FILE="${1:-}"
ERRORS=0
WARNINGS=0

echo "=========================================="
echo "  Submit Verification (R2: 提交前验证)"
echo "=========================================="

# ---- R13: 禁止新建 P0 文件 ----
echo ""
echo "[R13] 检查新建 P0 文件..."
NEW_P0=$(git diff --cached --name-only --diff-filter=A 2>/dev/null | grep "^P0_" | wc -l)
if [ "$NEW_P0" -gt 0 ]; then
    echo "  ❌ FAIL: 发现 ${NEW_P0} 个新建 P0_*.md 文件"
    echo "     规则: 事故记录写入 playbook/incident.md，禁止另建新文件"
    ERRORS=$((ERRORS+1))
else
    echo "  ✅ 无新建 P0 文件"
fi

# ---- R6: 文档同步检查 ----
echo ""
echo "[R6] 文档同步检查..."
CODE_CHANGES=$(git diff --cached --name-only 2>/dev/null | grep -cE "\.(py|js|ts|java|go|rs|cpp|c|h|vue|jsx|tsx)$" || echo "0")
DOC_CHANGES=$(git diff --cached --name-only 2>/dev/null | grep -cE "\.(md|rst|txt)$" || echo "0")
if [ "$CODE_CHANGES" -gt 0 ] && [ "$DOC_CHANGES" -eq 0 ]; then
    echo "  ⚠️  WARN: 代码变更(${CODE_CHANGES} files)未同步文档变更"
    echo "     建议: 更新 CHANGELOG.md 或相关文档"
    WARNINGS=$((WARNINGS+1))
else
    echo "  ✅ 文档同步检查通过 (代码:${CODE_CHANGES}, 文档:${DOC_CHANGES})"
fi

# ---- R9: 遗留标记检查 ----
echo ""
echo "[R9] 遗留标记检查..."
TODO_FILES=$(git diff --cached -G "TODO|FIXME|HACK|XXX" --name-only 2>/dev/null || true)
if [ -n "$TODO_FILES" ]; then
    TODO_COUNT=$(echo "$TODO_FILES" | grep -c . || echo "0")
    echo "  ⚠️  WARN: 发现 ${TODO_COUNT} 个文件含 TODO/FIXME 标记:"
    echo "$TODO_FILES" | sed 's/^/     - /'
    echo "     规则: 遗留问题必须显式标记，禁止静默跳过"
    WARNINGS=$((WARNINGS+1))
else
    echo "  ✅ 无新增遗留标记"
fi

# ---- R10: 需求来源标记 ----
echo ""
echo "[R10] 需求来源标记检查..."
if [ -n "$COMMIT_MSG_FILE" ] && [ -f "$COMMIT_MSG_FILE" ]; then
    if grep -qE '\[(user|tech-debt|incident|assumed)\]' "$COMMIT_MSG_FILE" 2>/dev/null; then
        echo "  ✅ Commit message 已标记需求来源"
    else
        echo "  ⚠️  WARN: Commit message 缺少需求来源标记"
        echo "     格式: [user] 描述 或 [tech-debt] 描述 或 [incident#N] 描述"
        WARNINGS=$((WARNINGS+1))
    fi
else
    echo "  ⏭️  SKIP: 无 commit message 文件供检查"
fi

# ---- 核心文件完整性检查 ----
echo ""
echo "[架构] 核心文件完整性..."
for f in META.md CONTEXT.md; do
    if [ -f "$f" ]; then
        echo "  ✅ $f 存在"
    else
        echo "  ❌ $f 缺失"
        ERRORS=$((ERRORS+1))
    fi
done

# ---- 结果汇总 ----
echo ""
echo "=========================================="
if [ $ERRORS -eq 0 ]; then
    echo "  ✅ 验证通过 (WARNINGS: ${WARNINGS})"
    echo "=========================================="
    if [ $WARNINGS -gt 0 ]; then
        echo "  ⚠️ 有 ${WARNINGS} 个警告，建议处理但不阻塞"
    fi
    exit 0
else
    echo "  ❌ 验证失败 (ERRORS: ${ERRORS}, WARNINGS: ${WARNINGS})"
    echo "=========================================="
    exit 1
fi
```

---

## guard-rail.py — 运行时守护

### 功能
- 实时监测命令重复执行（R3/R7）
- 确认对话轮次限制（R8）
- 违规文件创建拦截（R13）
- 操作日志记录

### 完整代码

```python
#!/usr/bin/env python3
"""
guard-rail.py — AI运行时规则守护
规则覆盖: R3, R7, R8, R13

设计为守护进程模式运行，拦截和阻断违规操作。
在实际集成中，需接入AI执行流的拦截点。

简化命令行用法:
  python guard-rail.py check-command "ls -la"    # 检查命令
  python guard-rail.py check-confirm             # 检查确认轮次
  python guard-rail.py check-file "P0_test.md"   # 检查文件创建
  python guard-rail.py reset                     # 重置计数器
"""

import sys
import json
import os
from collections import Counter
from datetime import datetime

# === 配置 ===
STATE_FILE = ".guard-rail-state.json"
MAX_REPEAT = 3       # R3/R7: 同一操作最多3次
MAX_CONFIRM = 2      # R8: 确认对话最多2轮


class GuardRail:
    """规则守护引擎"""
    
    def __init__(self, state_file=STATE_FILE):
        self.state_file = state_file
        self.command_history = Counter()
        self.confirm_count = 0
        self.violations = []
        self._load_state()
    
    def _load_state(self):
        """从文件恢复状态（跨会话持久化）"""
        if os.path.exists(self.state_file):
            try:
                with open(self.state_file) as f:
                    data = json.load(f)
                self.command_history = Counter(data.get("commands", {}))
                self.confirm_count = data.get("confirm_count", 0)
                self.violations = data.get("violations", [])
            except (json.JSONDecodeError, IOError):
                pass
    
    def _save_state(self):
        """保存状态到文件"""
        data = {
            "commands": dict(self.command_history),
            "confirm_count": self.confirm_count,
            "violations": self.violations[-50:],  # 保留最近50条
            "last_update": datetime.now().isoformat()
        }
        try:
            with open(self.state_file, "w") as f:
                json.dump(data, f, indent=2)
        except IOError:
            pass
    
    # === R3/R7: 命令重复检测 ===
    
    def check_command(self, cmd: str) -> dict:
        """
        检查命令是否触发重复限制
        返回: {"allowed": bool, "count": int, "reason": str}
        """
        # 提取命令关键词（简化处理）
        key = cmd.strip().split()[0] if cmd else "unknown"
        
        # 忽略无害命令
        safe_commands = {"cd", "pwd", "echo", "cat", "ls", "grep", "find"}
        if key in safe_commands:
            return {"allowed": True, "count": 0, "reason": "safe command"}
        
        self.command_history[key] += 1
        count = self.command_history[key]
        
        result = {
            "allowed": count <= MAX_REPEAT,
            "count": count,
            "reason": ""
        }
        
        if count > MAX_REPEAT:
            result["reason"] = f"命令 '{key}' 已执行 {count} 次，超过上限 {MAX_REPEAT}（R3/R7）"
            self._record_violation("R3/R7", f"命令重复: {key} ({count}次)")
        elif count == MAX_REPEAT:
            result["reason"] = f"警告: 命令 '{key}' 即将达到上限（{count}/{MAX_REPEAT}）"
        
        self._save_state()
        return result
    
    # === R8: 确认轮次检测 ===
    
    def check_confirm(self) -> dict:
        """
        检查确认对话是否超限
        返回: {"allowed": bool, "count": int, "reason": str}
        """
        self.confirm_count += 1
        
        result = {
            "allowed": self.confirm_count <= MAX_CONFIRM,
            "count": self.confirm_count,
            "reason": ""
        }
        
        if self.confirm_count > MAX_CONFIRM:
            result["reason"] = f"确认已 {self.confirm_count} 轮，超过上限 {MAX_CONFIRM}，建议上报（R8）"
            self._record_violation("R8", f"确认超限: {self.confirm_count}轮")
        elif self.confirm_count == MAX_CONFIRM:
            result["reason"] = f"警告: 确认即将达到上限（{self.confirm_count}/{MAX_CONFIRM}）"
        
        self._save_state()
        return result
    
    def reset_confirm(self):
        """重置确认计数器（用户明确确认后调用）"""
        self.confirm_count = 0
        self._save_state()
    
    # === R13: 文件创建拦截 ===
    
    def check_file_creation(self, filepath: str) -> dict:
        """
        检查是否违规创建 P0 文件
        返回: {"allowed": bool, "reason": str}
        """
        basename = os.path.basename(filepath)
        
        if basename.startswith("P0_"):
            reason = f"禁止创建 '{basename}'，事故记录请写入 playbook/incident.md（R13）"
            self._record_violation("R13", f"违规创建: {basename}")
            self._save_state()
            return {"allowed": False, "reason": reason}
        
        return {"allowed": True, "reason": ""}
    
    # === 违规记录 ===
    
    def _record_violation(self, rule: str, detail: str):
        """记录违规事件"""
        violation = {
            "timestamp": datetime.now().isoformat(),
            "rule": rule,
            "detail": detail
        }
        self.violations.append(violation)
    
    def get_violations(self, limit=10) -> list:
        """获取最近违规记录"""
        return self.violations[-limit:]
    
    def reset(self):
        """完全重置所有计数器"""
        self.command_history.clear()
        self.confirm_count = 0
        self.violations = []
        if os.path.exists(self.state_file):
            os.remove(self.state_file)
    
    def status(self) -> dict:
        """获取当前状态摘要"""
        return {
            "command_history": dict(self.command_history),
            "confirm_count": self.confirm_count,
            "total_violations": len(self.violations),
            "recent_violations": self.violations[-5:]
        }


# === CLI 接口 ===

def main():
    if len(sys.argv) < 2:
        print("Usage: guard-rail.py <command> [args...]")
        print("")
        print("Commands:")
        print("  check-command <cmd>   检查命令是否合规")
        print("  check-confirm         检查确认轮次")
        print("  check-file <path>     检查文件创建")
        print("  reset-confirm         重置确认计数")
        print("  reset                 完全重置")
        print("  status                显示状态")
        sys.exit(1)
    
    guard = GuardRail()
    cmd = sys.argv[1]
    
    if cmd == "check-command" and len(sys.argv) >= 3:
        result = guard.check_command(sys.argv[2])
        status = "✅ ALLOWED" if result["allowed"] else "❌ BLOCKED"
        print(f"{status}: {result['reason']}" if result['reason'] else status)
        sys.exit(0 if result["allowed"] else 1)
    
    elif cmd == "check-confirm":
        result = guard.check_confirm()
        status = "✅ ALLOWED" if result["allowed"] else "❌ BLOCKED"
        print(f"{status}: {result['reason']}" if result['reason'] else status)
        sys.exit(0 if result["allowed"] else 1)
    
    elif cmd == "check-file" and len(sys.argv) >= 3:
        result = guard.check_file_creation(sys.argv[2])
        status = "✅ ALLOWED" if result["allowed"] else "❌ BLOCKED"
        print(f"{status}: {result['reason']}" if result['reason'] else status)
        sys.exit(0 if result["allowed"] else 1)
    
    elif cmd == "reset-confirm":
        guard.reset_confirm()
        print("确认计数器已重置")
    
    elif cmd == "reset":
        guard.reset()
        print("所有计数器已重置")
    
    elif cmd == "status":
        import pprint
        pprint.pprint(guard.status())
    
    else:
        print(f"未知命令: {cmd}")
        sys.exit(1)


if __name__ == "__main__":
    main()
```

---

## 部署指南

### Step 1: 安装脚本

```bash
# 创建目录
mkdir -p scripts

# 保存脚本（复制上方代码）
cat > scripts/pre-flight.sh << 'EOF'
# ... pre-flight.sh 代码 ...
EOF
cat > scripts/verify.sh << 'EOF'
# ... verify.sh 代码 ...
EOF
cat > scripts/guard-rail.py << 'EOF'
# ... guard-rail.py 代码 ...
EOF

# 赋予执行权限
chmod +x scripts/pre-flight.sh scripts/verify.sh scripts/guard-rail.py
```

### Step 2: 配置 Git Hook

```bash
# 配置 pre-commit hook（可选但推荐）
cat > .git/hooks/pre-commit << 'HOOK'
#!/bin/bash
./scripts/verify.sh "$1" || exit 1
HOOK
chmod +x .git/hooks/pre-commit
```

### Step 3: 测试

```bash
# 测试 pre-flight
./scripts/pre-flight.sh

# 测试 verify
./scripts/verify.sh

# 测试 guard-rail
python scripts/guard-rail.py check-command "test"
python scripts/guard-rail.py check-command "test"
python scripts/guard-rail.py check-command "test"  # 第3次
python scripts/guard-rail.py check-command "test"  # 第4次 - 应被阻断
python scripts/guard-rail.py check-file "P0_test.md"  # 应被阻断
python scripts/guard-rail.py status
python scripts/guard-rail.py reset
```

---

## 脚本覆盖规则统计

```
脚本化覆盖的规则:
  R1:  会话启动 → pre-flight.sh
  R2:  提交验证 → verify.sh
  R3:  死循环检测 → guard-rail.py
  R7:  命令重试上限 → guard-rail.py
  R8:  确认轮次上限 → guard-rail.py
  R11: 单一来源检查 → pre-flight.sh
  R12: 上下文检查 → pre-flight.sh
  R13: P0文件拦截 → verify.sh + guard-rail.py
  ─────────────────────────────────
  脚本覆盖: 8/13 条执行规则 (62%)

需人工记忆的规则:
  R4:  不改需求外内容
  R5:  执行前确认
  R6:  文档同步（脚本辅助检测）
  R9:  遗留问题标记（脚本辅助检测）
  R10: 需求来源标记（脚本辅助检测）
  R14: 新增淘汰旧规则（流程层面）
  R15: 经验≠规则（流程层面）
  ─────────────────────────────────
  人工: 7/15 条（其中4条有脚本辅助）
```
