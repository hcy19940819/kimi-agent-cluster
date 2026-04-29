# AI协作规范仓库 — 极简架构设计方案 v1.0

> 设计哲学：规则越少越好，执行越简单越好
> 核心指标：核心约束 ≤ 15条 | 总文件数 ≤ 8 | 单一可信来源

---

## 一、旧架构问题诊断

### 1.1 失控根因分析

| 问题 | 根因 | 影响 |
|------|------|------|
| MEMORY.md 15KB | 每条经验教训变成永久规则，只增不减 | 阅读成本高，关键信息淹没 |
| 8条经验→8条新规则 | 缺乏"规则生命周期"管理 | 规则堆砌，互相矛盾 |
| AGENTS.md vs MEMORY.md重复 | 按文件类型分，不按场景分 | 同一规则多处维护 |
| P0文件 proliferation | 每次事故产生一个永久文件 | 文件膨胀，无收敛机制 |
| 协议嵌套5+层 | 试图用"更多步骤"防止错误 | 遗漏率反而上升 |
| 规则写了≠执行了 | 无验证机制，无自动化 | 全靠人记，必然遗忘 |
| TODO.md 20KB | 无清理机制，任务与文档混杂 | 无法快速获取状态 |

### 1.2 核心教训

> **复杂性是规则的头号敌人。规则越多，遵守率越低。**
>
> 旧架构试图用"更多规则"来防止违反规则，形成恶性循环。新架构用"更少但更强的约束 + 自动化"来打破这个循环。

---

## 二、新架构总览

### 2.1 文件结构（8个文件）

```
AI_Hcy/
├── README.md              # 入口导航（仅链接，无规则）
├── META.md                # 元规则（5条，永远不变）★ 唯一可信来源
├── CONTEXT.md             # 项目快照（<1KB，自动维护）
├── SKILLS.md              # 技能注册表（<2KB，索引文件）
├── playbook/              # 场景手册（按场景分）
│   ├── start.md           # 会话启动（5步检查清单）
│   ├── develop.md         # 开发工作流（核心约束清单）
│   ├── submit.md          # 提交与验证（发布检查清单）
│   └── incident.md        # 事故处理（P0收敛处）
└── scripts/               # 自动化脚本
    ├── pre-flight.sh      # 启动自检
    ├── verify.sh          # 提交前验证
    └── guard-rail.py      # 规则守护
```

### 2.2 旧文件处置

| 旧文件 | 处置 | 说明 |
|--------|------|------|
| `AGENTS.md` | **删除** | 规则内容合并到 META.md + playbook/ |
| `TODO.md` | **删除** | 任务追踪回归外部工具（GitHub Issues/Projects），本仓库只保留上下文 |
| `.agents/skills/...` | **重构** | 合并为 SKILLS.md 索引表 |
| `MEMORY.md` | **拆分** | 项目信息→CONTEXT.md，规则→META.md，流程→playbook/ |
| `DECISIONS.md` | **删除** | 决策记录融入上下文提交信息 |
| `P0_*.md` | **全部合并** | 合并到 playbook/incident.md，历史归档 |
| `shared/coding-standards.md` | **保留** | 编码规范独立文件 |

### 2.3 核心指标

```
约束目标：
- 核心规则（META.md）：≤ 5条元规则 + ≤ 10条执行规则 = ≤ 15条
- 场景手册（playbook/）：每条手册 ≤ 30行
- 项目快照（CONTEXT.md）：≤ 1KB
- 总文件数：≤ 8个（不含脚本和历史归档）
- 启动协议：≤ 3步（旧架构5+步）
- P0事故：收敛到单一文件，非每次新文件
```

---

## 三、三层治理架构

### 3.1 Layer 1 — 元规则层（META.md）

**作用**：定义"规则如何管理规则"。这是唯一不能修改的文件（修改需特殊审批）。

**核心洞察**：旧架构的问题是规则没有自我管理规则。元规则层就是这个"规则的规则"。

**5条元规则**：

```
MR1. 单一来源：一个规则只存在于一个地方，发现重复立即删除
MR2. 场景驱动：规则按使用场景组织，不按文件类型组织  
MR3. 自动化优先：能脚本化的规则绝不靠人记，脚本未覆盖的规则无效
MR4. 约束有界：核心执行规则≤15条，新增必须淘汰旧规则
MR5. 经验≠规则：经验教训只能在incident.md中记录，不能自动升级为规则
```

### 3.2 Layer 2 — 上下文层（CONTEXT.md + SKILLS.md）

**作用**：回答"我是谁？现在是什么状态？"

**CONTEXT.md**（<1KB，每次会话后自动更新）：
- 项目基本信息（路径/版本/技术栈）
- 当前任务上下文（本次会话目标）
- 关键约束速查（仅3-5条最高频的）

**SKILLS.md**（<2KB，索引文件）：
- 可用技能列表（名称 + 触发场景 + 文件路径）
- 不存放技能内容，只存放索引

### 3.3 Layer 3 — 场景手册层（playbook/*.md）

**作用**：回答"在这个场景下我该做什么？"

按场景分4个文件：
- **start.md** — 会话启动（3步启动流程，替代旧5步）
- **develop.md** — 开发工作流（需求→编码→自测的规则速查）
- **submit.md** — 提交验证（commit前的强制检查清单）
- **incident.md** — 事故处理（所有P0收敛于此，含历史记录）

**设计原则**：每个场景手册 = 检查清单（Checklist）+ 命令速查，不包含"为什么"。

---

## 四、关键设计决策

### 决策1：为什么删除 TODO.md？

TODO.md 20KB的教训：任务追踪和项目规范混在一起，导致规范被任务淹没。

**新方案**：
- 任务追踪 → 外部工具（GitHub Issues / Linear / Notion）
- CONTEXT.md 只记录"当前会话的任务上下文"（1-2句话）
- 每个会话开始时，从外部工具同步当前任务到 CONTEXT.md

### 决策2：如何防止经验教训变成永久规则？

旧架构：每条经验教训 → 新的"强制执行规则" → 规则膨胀

**新方案**：
- 经验教训只能写入 incident.md 的"经验库"章节
- 升级为规则必须经过"规则生命周期审批"（见第六节）
- META.md 的 MR5 作为硬约束

### 决策3：P0文件如何收敛？

旧架构：每次事故 → 新的 P0_*.md 文件 → 文件 proliferation

**新方案**：
- 单一文件 `playbook/incident.md`
- 新事故以"记录"形式追加，不是新文件
- 每条记录含：事故/根因/修复/验证状态
- 已关闭的事故归档到 `archive/incidents/YYYY-MM/` 目录

### 决策4：启动协议如何简化？

旧架构：读记忆→功能认知恢复→读TODO→搜索关键词→确认方案 = 5步

**新方案**：
1. 读取 CONTEXT.md（获取当前上下文）
2. 读取 META.md（确认约束）
3. 确认本次目标 → 执行

> 删除"读TODO"（外部化）和"搜索关键词"（自动化脚本处理）

---

## 五、新旧规则映射

### 5.1 规则精简总表

| 旧规则（60+条） | 新归属 | 处理方式 |
|---------------|--------|----------|
| AGENTS.md 3条红线 | → META.md 执行规则 | 合并为3条核心约束 |
| AGENTS.md 2个协议 | → scripts/ + playbook/ | 协议→脚本自动化 |
| 启动协议5步 | → playbook/start.md 3步 | 删除冗余步骤 |
| Skill识别协议7个skill | → SKILLS.md + 脚本 | 索引化+自动识别 |
| MEMORY.md 基本信息 | → CONTEXT.md | 精简 |
| MEMORY.md 6个安全检查脚本 | → scripts/verify.sh | 脚本化 |
| MEMORY.md 硬限制5条 | → CONTEXT.md #Constraints | 精简 |
| 经验1-规范唯一来源 | → MR1（元规则） | 升级 |
| 经验2-核心信息唯一来源 | → MR2（元规则） | 升级 |
| 经验3-会话启动5步 | → playbook/start.md 3步 | 简化 |
| 经验4-P0代码审查 | → playbook/incident.md | 收敛 |
| 经验5-AI行为红线5条 | → META.md 执行规则 | 精简为3条 |
| 经验6-验证无残留6条 | → scripts/verify.sh | 自动化 |
| 经验7-确认类操作4条 | → META.md 1条元规则 | 合并 |
| 经验8-需求来源铁律4条 | → playbook/develop.md | 精简 |
| P0_CODE_REVIEW_40 | → incident.md 记录 #40 | 收敛 |
| P0_DEADLOOP_41 | → incident.md 记录 #41 | 收敛 |
| P0_REGRESSION_GUARD | → scripts/verify.sh | 脚本化 |
| P0_ROOT_CAUSE_39 | → incident.md 记录 #39 | 收敛 |
| 其他20+条隐含规则 | → 删除 | 未使用的规则不是规则 |

### 5.2 15条核心执行规则

```
【自动执行层 — 脚本覆盖，无需记忆】
R1. 会话启动必须运行 pre-flight.sh（3步自检）
R2. 代码提交前必须运行 verify.sh（检查清单通过）
R3. 发现死循环（同一操作>3次）自动终止并报告

【决策约束层 — 3条核心红线】
R4. 不改需求外内容（不自主改UI/加功能/重构）
R5. 执行前确认方案变更（汇报→同意→执行）
R6. 文档与代码同步提交（不允许文档滞后）

【质量约束层 — 防止已知问题】
R7. 同一Shell命令最多重试3次
R8. 确认对话最多2轮，未决立即上报
R9. 遗留问题必须标记 TODO/KNOWN-ISSUE，禁止静默跳过
R10. 需求必须标明来源（user/tech-debt/incident#N），禁止臆测

【信息管理层 — 保持上下文一致】
R11. 一个规则只存在于一个地方（META.md是唯一规则来源）
R12. 当前任务上下文必须写在 CONTEXT.md #Session 中
R13. 事故记录必须写入 incident.md，禁止另建新文件

【元规则约束层 — 规则自我管理】
R14. 新增规则必须淘汰旧规则（总数≤15）
R15. 经验教训≠规则，升级需审批（见META.md MR5）
```

---

## 六、防呆机制设计

### 6.1 核心理念

> **最好的规则是让人不知道规则的存在就能遵守。**

### 6.2 防呆设计矩阵

| 防呆级别 | 机制 | 应用规则 |
|---------|------|---------|
| **强制层** | 脚本拦截，不通过无法继续 | R1, R2, R3 |
| **默认层** | 默认行为就是正确行为 | R4, R5, R6, R12 |
| **检测层** | 违规自动检测并告警 | R7, R8, R9, R10, R11, R13 |
| **流程层** | 流程设计使违规更困难 | R14, R15 |

### 6.3 自动化脚本设计

#### scripts/pre-flight.sh — 会话启动自检

```bash
#!/bin/bash
# pre-flight.sh — 会话启动强制检查
# 用法：每次AI会话启动时自动运行
# 规则覆盖：R1, R11, R12

echo "=== Pre-flight Check ==="

# 1. 验证 META.md 存在（R11: 单一来源）
[ -f META.md ] || { echo "FATAL: META.md 缺失"; exit 1; }

# 2. 验证 CONTEXT.md 存在并可读（R12: 上下文）
[ -f CONTEXT.md ] || { echo "WARN: CONTEXT.md 缺失，创建模板"; cp templates/context.md CONTEXT.md; }

# 3. 验证核心文件总数≤8（防止文件增殖）
FILE_COUNT=$(find . -maxdepth 2 -name "*.md" -not -path "./archive/*" -not -path "./.git/*" | wc -l)
[ $FILE_COUNT -le 10 ] || echo "WARN: 文件数($FILE_COUNT)超过阈值"

# 4. 输出当前上下文摘要
echo "=== 当前上下文 ==="
grep -A5 "# Current" CONTEXT.md 2>/dev/null || echo "(无上下文，请在CONTEXT.md中设置)"

echo "=== Pre-flight 通过 ==="
```

#### scripts/verify.sh — 提交前验证

```bash
#!/bin/bash
# verify.sh — 提交前强制验证
# 用法：git commit 前自动运行（建议配置为 git pre-commit hook）
# 规则覆盖：R2, R6, R9, R10

echo "=== Submit Verification ==="
ERRORS=0

# 1. 检查文档同步（R6）
if git diff --cached --name-only | grep -qE "\.(py|js|ts|java|go|rs|cpp|c|h)$"; then
    # 有代码变更，检查是否有文档同步
    DOC_SYNC=$(git diff --cached --name-only | grep -cE "\.(md|txt|rst)$")
    [ $DOC_SYNC -eq 0 ] && { echo "FAIL R6: 代码变更需同步文档"; ERRORS=$((ERRORS+1)); }
fi

# 2. 检查遗留问题标记（R9）
TODO_MARKERS=$(git diff --cached -G "TODO|FIXME|HACK|XXX" --name-only | wc -l)
[ $TODO_MARKERS -gt 0 ] && echo "WARN: 发现 $TODO_MARKERS 个文件含 TODO/FIXME 标记"

# 3. 检查需求来源标记（R10）
# （适用于commit message检查）
COMMIT_MSG=$(cat "$1" 2>/dev/null)
echo "$COMMIT_MSG" | grep -qE "\[(user|tech-debt|incident|assumed)\]" || {
    echo "WARN: Commit建议标记需求来源 [user/tech-debt/incident#N/assumed]"
}

# 4. 检查新增md文件是否符合架构（R13: 防止新建P0文件）
NEW_MD=$(git diff --cached --name-only --diff-filter=A | grep "^P0_" | wc -l)
[ $NEW_MD -gt 0 ] && { echo "FAIL R13: 禁止新建P0_文件，请写入incident.md"; ERRORS=$((ERRORS+1)); }

[ $ERRORS -eq 0 ] && { echo "=== 验证通过 ==="; exit 0; } || { echo "=== 验证失败($ERRORS) ==="; exit 1; }
```

#### scripts/guard-rail.py — 运行中守护

```python
#!/usr/bin/env python3
"""
guard-rail.py — 运行时规则守护
规则覆盖：R3, R7, R8, R13
用法：在AI执行环境中以守护模式运行，监控操作日志
"""

import sys
from collections import Counter

class GuardRail:
    # R3: 死循环检测
    MAX_REPEAT = 3      # 同一操作最多3次
    # R8: 确认上限
    MAX_CONFIRM = 2     # 确认对话最多2轮
    
    def __init__(self):
        self.command_history = Counter()
        self.confirm_count = 0
    
    def check_command(self, cmd: str) -> bool:
        """检查命令是否触发R3/R7"""
        key = cmd.strip().split()[0] if cmd else ""
        self.command_history[key] += 1
        if self.command_history[key] > self.MAX_REPEAT:
            print(f"[GUARD] 阻断: 命令'{key}'已执行{self.command_history[key]}次 (R3/R7)")
            return False
        return True
    
    def check_confirm(self) -> bool:
        """检查确认轮次R8"""
        self.confirm_count += 1
        if self.confirm_count > self.MAX_CONFIRM:
            print(f"[GUARD] 阻断: 确认已{self.confirm_count}轮，建议上报 (R8)")
            return False
        return True
    
    def check_file_creation(self, filepath: str) -> bool:
        """检查是否违规创建P0文件 R13"""
        if filepath.startswith("P0_"):
            print(f"[GUARD] 阻断: 禁止创建'{filepath}'，请写入incident.md (R13)")
            return False
        return True

if __name__ == "__main__":
    # 简化版命令行接口
    guard = GuardRail()
    # 实际集成时需接入AI操作流
    pass
```

### 6.4 默认层设计

**让正确行为成为默认行为**：

| 规则 | 默认行为设计 |
|------|-------------|
| R4（不改需求外内容） | CONTEXT.md 的 #Session 只写当前任务，不写允许做的事 |
| R5（执行前确认） | playbook 所有步骤都以"确认目标"开头，形成惯性 |
| R6（文档同步） | verify.sh 作为 git pre-commit hook 自动运行 |
| R12（上下文记录） | pre-flight.sh 自动读取并显示当前上下文 |

---

## 七、规则生命周期管理

### 7.1 规则状态机

```
                    ┌─────────────┐
                    │   经验库     │  (incident.md 经验章节)
                    │ (非规则)     │
                    └──────┬──────┘
                           │ "规则化提案"
                           ▼
                    ┌─────────────┐
         ┌─────────│   提案中     │
         │         │ (讨论/评估)  │
         │         └──────┬──────┘
         │  "拒绝"        │ "审批通过"
         │                ▼
         │         ┌─────────────┐
         │         │   待生效     │
         │         │ (淘汰旧规则) │
         └─────────┤             │
                   └──────┬──────┘
                          │ "淘汰完成"
                          ▼
                   ┌─────────────┐
            ┌─────│   生效中     │
            │     │ (META.md)    │
            │     └──────┬──────┘
            │            │ "定期review"
            │            ▼
            │     ┌─────────────┐
            │     │   待审查     │
            └─────┤ (30天周期)  │
             "废止│             │
              或  └──────┬──────┘
             归档        │ "仍有效"
                        └────────┘
```

### 7.2 新增规则审批流程（防止随意添加）

```
触发条件：有人提议新增规则

Step 1: 提案
  - 在 incident.md 的"规则提案"章节添加条目
  - 格式：| 编号 | 提议规则 | 解决的重复问题 | 提议人 | 日期 |

Step 2: 三问检查（必须全部回答"是"）
  Q1: 这条规则解决的是重复出现的问题吗？（一次性的问题不需要规则）
  Q2: 能否用脚本自动化？（能→写脚本，不新增规则）
  Q3: 淘汰哪条旧规则？（不淘汰旧规则→不能新增）

Step 3: 试运行
  - 作为"建议"写入 playbook（非强制）
  - 试运行2周

Step 4: 转正或废弃
  - 有效→移入 META.md，淘汰旧规则
  - 无效→移入 incident.md 经验库，标注"已废弃"
```

### 7.3 定期Review机制

```
频率：每30天
执行者：AI自动提醒 + 人工确认

Review清单：
[ ] 当前执行规则数量 ≤ 15？（超出需淘汰）
[ ] 每条规则在过去30天内是否被触发过？（未触发→考虑删除）
[ ] 是否有规则被脚本覆盖？（被覆盖→删除，改为自动化）
[ ] incident.md 中是否有可升级为规则的经验？（最多1条）
[ ] 是否有互相矛盾的规则？（有→修正）
```

### 7.4 规则淘汰机制

```
自动淘汰条件（满足任一即触发）：
1. 连续30天未被引用
2. 已被脚本自动化覆盖
3. 与其他规则矛盾
4. 新增规则导致总数>15

淘汰流程：
1. 在规则前标注 [DEPRECATED-YYYY-MM-DD]
2. 迁移到 archive/rules/ 目录
3. 在 incident.md 记录淘汰原因
4. 更新 META.md 规则编号
```

---

## 八、文件模板

详见配套模板文件：
- `templates/META.md` — 元规则文件
- `templates/CONTEXT.md` — 项目快照文件
- `templates/SKILLS.md` — 技能注册表
- `templates/playbook/*.md` — 4个场景手册

---

## 九、迁移路线图

### Phase 1: 立即（第1天）
- [ ] 创建新文件结构
- [ ] 编写 META.md（5条元规则）
- [ ] 将 MEMORY.md 拆分到 CONTEXT.md + playbook/
- [ ] 删除 AGENTS.md（内容已合并）

### Phase 2: 第2-3天
- [ ] 编写 4个场景手册
- [ ] 合并所有 P0 文件到 incident.md
- [ ] 配置 git pre-commit hook（verify.sh）
- [ ] 编写 pre-flight.sh

### Phase 3: 第4-7天
- [ ] 试运行新架构
- [ ] 根据实际使用调整 playbook
- [ ] 归档旧文件到 archive/

### Phase 4: 持续
- [ ] 30天规则review
- [ ] 持续优化脚本
- [ ] 经验库积累（但不自动升级）

---

## 十、成功指标

| 指标 | 旧架构 | 新架构目标 |
|------|--------|-----------|
| 核心规则数量 | 60+条 | ≤ 15条 |
| 规范文件总数 | 10+ | ≤ 8个 |
| MEMORY.md 大小 | 15KB | < 1KB（CONTEXT.md） |
| TODO.md 大小 | 20KB | 删除（外部化） |
| 启动协议步骤 | 5步 | 3步 |
| P0事故文件数 | 持续增长 | 收敛到1个 |
| 规则遵守率 | ~50%（规则太多记不住） | > 80%（脚本+防呆） |

---

*设计完成。下一步：查看 templates/ 目录下的完整模板文件。*
