# AI_Hcy 仓库架构重构 — 渐进式迁移计划

> **版本**: v1.0
> **日期**: 2026-01-12
> **状态**: 待执行
> **迁移方式**: 双轨并行 + 渐进切换
> **核心约束**: 零停机 | 可回滚 | AI协作不中断

---

## 目录

1. [迁移总览](#一迁移总览)
2. [阶段详细计划](#二阶段详细计划)
3. [风险分析](#三风险分析)
4. [并行处理策略](#四并行处理策略)
5. [验证清单](#五验证清单)
6. [回滚计划](#六回滚计划)
7. [附录](#七附录)

---

## 一、迁移总览

### 1.1 迁移策略："桥梁并行"模式

```
时间线
─────────────────────────────────────────────────────────>

Phase 1    Phase 2      Phase 3      Phase 4      Phase 5      Phase 6
(准备)     (奠基)       (核心)       (整合)       (切换)       (清理)

旧架构     旧架构       双轨并行     双轨并行     新架构为主   新架构
    \     /  \        /  \        /    \       /    旧架构存档
     \   /    \      /    \      /      \     /
      \ /      \    /      \    /        \   /
       X        \  /        \  /          \ /
      / \        \/          \/            X
     /   \       /\          /\          / \
    /     \     /  \        /  \        /   \
   /       \   /    \      /    \      /     \
新架构     新骨架   新骨架    新内容   新内容    新架构
(只读观察) (写入)    (验证)    (切换)    (清理)
```

### 1.2 关键原则

| 原则 | 说明 |
|------|------|
| **只增不减（前4阶段）** | 不删除任何旧文件，只创建新文件和添加兼容层 |
| **影子写入** | 新架构文件由旧架构文件自动/半自动生成 |
| **双向索引** | 新旧文件之间维护映射关系，便于追踪和回滚 |
| **渐进信任** | 从"参考"到"主要"到"唯一"，逐步提升新架构权重 |
| **可逆操作** | 每阶段结束后都可快速回到上一阶段状态 |

### 1.3 迁移范围

```
旧架构元素                          新架构对应
─────────────────────────────────────────────────────────────────
AGENTS.md                    →    meta-rules.md (Layer 1)
README.md                    →    profile.md (Layer 2)
TODO.md (活跃项)              →    playbooks/startup.md + todo/active.md
TODO.md (历史项)              →    archive/todo-history-YYYY-MM.md
TODO.md (20KB)               →    拆分后 ≤50活跃项 + 归档
P0_CODE_REVIEW_40.md         →    incident-log.md 条目
P0_DEADLOOP_41.md            →    incident-log.md 条目
P0_REGRESSION_GUARD.md       →    incident-log.md 条目
P0_REGRESSION_QUICK.md       →    incident-log.md 条目
P0_ROOT_CAUSE_39.md          →    incident-log.md 条目
.skills/todo-manager/        →    playbooks/startup.md (整合)
shared/coding-standards.md   →    playbooks/develop.md (引用)
projects/_template/          →    profile.md 项目模板引用
projects/*/MEMORY.md         →    profile.md 状态快照
projects/*/DECISIONS.md      →    playbooks/develop.md 决策记录
```

---

## 二、阶段详细计划

### Phase 1: 准备与审计（预计 2-3 天）

**目标**: 完成迁移前审计，建立安全基线，创建兼容层

#### 动作清单

| 序号 | 动作 | 类型 | 文件路径 | 说明 |
|------|------|------|----------|------|
| 1.1 | 创建 | 新目录 | `.migration/` | 迁移工作目录 |
| 1.2 | 创建 | 新文件 | `.migration/MIGRATION_LOG.md` | 迁移日志，记录所有操作 |
| 1.3 | 创建 | 新文件 | `.migration/FILE_INVENTORY.md` | 全文件清单与哈希值 |
| 1.4 | 创建 | 新文件 | `.migration/NEW_OLD_MAP.md` | 新旧文件映射表 |
| 1.5 | 创建 | 脚本 | `.migration/scripts/verify_checksum.sh` | 校验脚本 |
| 1.6 | 创建 | 兼容层 | `.agents/BRIDGE.md` | 桥梁文件：旧架构入口指向新位置 |
| 1.7 | 备份 | 快照 | `.migration/backup/phase0/` | 完整仓库快照（tar.gz） |
| 1.8 | 创建 | 新文件 | `.migration/DEPRECATION_SCHEDULE.md` | 旧文件废弃时间表 |

#### 详细步骤

**Step 1: 建立安全基线**
```bash
# 创建完整备份
tar czf .migration/backup/phase0/ai_hcy-$(date +%Y%m%d-%H%M%S).tar.gz \
  --exclude=.migration/ \
  --exclude=.git/ \
  .

# 生成所有文件的 SHA256 校验和
find . -type f -not -path './.migration/*' -not -path './.git/*' | \
  sort | xargs sha256sum > .migration/FILE_INVENTORY.md
```

**Step 2: 创建桥梁文件 BRIDGE.md**
```markdown
# 架构迁移桥梁文件

> 此文件在迁移期间作为新旧架构的桥梁。
> AI助手应优先读取新架构文件，如找不到则回退到旧文件。

## 新架构文件位置速查

| 旧文件 | 新文件 | 状态 |
|--------|--------|------|
| AGENTS.md | meta-rules.md | [未创建] |
| README.md | profile.md | [未创建] |
| TODO.md | playbooks/startup.md + todo/active.md | [未创建] |
| P0_*文件 | incident-log.md | [未创建] |

## 迁移阶段: Phase 1/6
## 当前生效架构: 旧架构（完整）
## 新架构状态: 只读观察
```

**Step 3: 创建新旧映射表**
- 逐项映射每个旧文件到新位置
- 标注迁移优先级（P0=核心, P1=重要, P2=可选）
- 标注文件大小和复杂度

#### 交付物

| 交付物 | 路径 | 验收标准 |
|--------|------|----------|
| 完整备份 | `.migration/backup/phase0/*.tar.gz` | 可解压还原，校验和匹配 |
| 文件清单 | `.migration/FILE_INVENTORY.md` | 包含所有文件的 SHA256 |
| 映射表 | `.migration/NEW_OLD_MAP.md` | 所有文件都有映射条目 |
| 桥梁文件 | `.agents/BRIDGE.md` | AI能读取到新架构规划 |
| 废弃时间表 | `.migration/DEPRECATION_SCHEDULE.md` | 每旧文件都有废弃阶段标注 |

#### 验证标准

- [ ] 备份可完整还原仓库到迁移前状态
- [ ] 执行校验和脚本无差异
- [ ] BRIDGE.md 内容清晰，AI助手能理解迁移计划
- [ ] 所有团队成员已审阅并同意迁移计划

#### 回滚方案

```
触发条件: 审计发现数据丢失风险、团队未达成共识
回滚步骤:
  1. 删除 .migration/ 目录
  2. 删除 .agents/BRIDGE.md
  3. 仓库恢复至 Phase 0 状态
回滚耗时: < 5 分钟
数据影响: 零
```

---

### Phase 2: 奠基 — 创建 Layer 1 + Layer 2（预计 3-4 天）

**目标**: 创建元规则层和角色上下文层，建立新架构骨架

#### 动作清单

| 序号 | 动作 | 类型 | 文件路径 | 来源 |
|------|------|------|----------|------|
| 2.1 | 创建 | 新文件 | `meta-rules.md` | 提炼 AGENTS.md 核心规则 |
| 2.2 | 创建 | 新文件 | `profile.md` | 整合 README.md + 项目信息 |
| 2.3 | 创建 | 新目录 | `playbooks/` | 场景手册目录 |
| 2.4 | 创建 | 新文件 | `playbooks/.gitkeep` | 占位 |
| 2.5 | 创建 | 新目录 | `archive/` | 历史归档目录 |
| 2.6 | 创建 | 新文件 | `archive/.gitkeep` | 占位 |
| 2.7 | 创建 | 新目录 | `todo/` | 活跃事项目录 |
| 2.8 | 创建 | 新文件 | `todo/.gitkeep` | 占位 |
| 2.9 | 修改 | 更新 | `.agents/BRIDGE.md` | 更新状态为 Phase 2 |
| 2.10 | 创建 | 兼容说明 | `meta-rules.md` 顶部 | 标注与 AGENTS.md 的对应关系 |

#### 详细步骤

**Step 1: 创建 meta-rules.md（Layer 1）**

从 AGENTS.md 中提炼 3-5 条元规则：

```markdown
# 元规则

> 本文件定义"如何管理规则"。所有规则的宪法。
> 创建时间: 2026-01-12 | 对应旧文件: AGENTS.md

## M1: 规则分层原则
- 元规则（本文件）≤ 5 条
- 角色上下文（profile.md）≤ 1 页
- 场景手册（playbooks/）按场景拆分

## M2: 规则进化原则
- 规则变更必须经过至少 1 次实践验证
- 新增规则需说明解决什么问题
- 每季度审查一次规则有效性

## M3: 冲突解决原则
- 下层规则服从上层规则
- 同层冲突以时间最新者为准
- 刻意冲突需显式标注理由

## M4: 信息单一来源原则
- 同一信息只在一个地方是权威的
- 其他位置只能引用，不能复制
- 引用必须标注来源位置

## M5: 迁移期双轨原则
- 2026-Q1 期间新旧架构并行
- 新架构文件优先，旧文件作为备份参考
- 过渡期结束后旧文件移至 archive/legacy/

---
*旧文件对应: AGENTS.md（完整内容仍可在 archive/legacy/AGENTS.md 查看）*
```

**Step 2: 创建 profile.md（Layer 2）**

整合 README.md + 项目基本信息 + 当前状态：

```markdown
# Profile: AI_Hcy 协作规范仓库

## 身份
- **我是谁**: AI协作规范体系的治理仓库
- **用户**: AI助手与人类开发者协作使用
- **创建日期**: 2025年

## 项目速查

| 项目 | 路径 | 状态 | 最后更新 |
|------|------|------|----------|
| video_management | projects/video_management/ | 活跃 | 2026-01 |
| stboy_auto | projects/stboy_auto/ | 活跃 | - |
| quick_extract | projects/quick_extract/ | 活跃 | - |
| auto_download | projects/auto_download/ | 活跃 | - |
| context_menu_manager | projects/context_menu_manager/ | 活跃 | - |

## 当前状态快照
- 架构迁移: Phase 2/6 进行中
- 活跃P0事故: 5个（待整合）
- TODO项: ~60项（待拆分）

## 快速导航

| 场景 | 手册 |
|------|------|
| 启动会话 | playbooks/startup.md [未创建] |
| 开发任务 | playbooks/develop.md [未创建] |
| 提交工作 | playbooks/commit.md [未创建] |
| 事故响应 | playbooks/incident.md [未创建] |

## 旧文件索引
- AGENTS.md → meta-rules.md（已迁移）
- README.md → profile.md（已迁移）
- TODO.md → todo/active.md + archive/（待迁移）
```

**Step 3: 创建目录结构**

```bash
mkdir -p playbooks archive todo .migration/backup/phase2
```

#### 交付物

| 交付物 | 路径 | 验收标准 |
|--------|------|----------|
| 元规则文件 | `meta-rules.md` | 3-5条规则，每条有清晰说明，标注旧文件对应关系 |
| 角色上下文 | `profile.md` | 包含身份、项目速查、状态快照、快速导航 |
| 目录结构 | `playbooks/`, `archive/`, `todo/` | 目录存在且可写入 |
| 桥梁更新 | `.agents/BRIDGE.md` | 标注 Phase 2 状态，列出已创建的新文件 |

#### 验证标准

- [ ] meta-rules.md 内容完整，元规则清晰可执行
- [ ] profile.md 能被新会话的AI快速理解仓库结构
- [ ] 新旧文件同时存在，旧文件未被修改
- [ ] 目录结构符合目标架构规划
- [ ] 至少1名团队成员能用新文件开始工作

#### 回滚方案

```
触发条件: 新文件内容遗漏关键规则、团队反馈不好用
回滚步骤:
  1. 删除 meta-rules.md, profile.md
  2. 删除 playbooks/, archive/, todo/ 目录
  3. 恢复 .agents/BRIDGE.md 到 Phase 1 状态
  4. 保留 .migration/ 日志供分析
回滚耗时: < 10 分钟
数据影响: 零（旧文件未动）
```

---

### Phase 3: 核心 — TODO拆分 + Playbooks创建（预计 5-7 天）

**目标**: 拆分TODO.md，创建场景手册，建立双轨运行机制

#### 动作清单

| 序号 | 动作 | 类型 | 文件路径 | 来源 |
|------|------|------|----------|------|
| 3.1 | 创建 | 新文件 | `playbooks/startup.md` | 整合 .skills/todo-manager/ + TODO启动项 |
| 3.2 | 创建 | 新文件 | `playbooks/develop.md` | 整合 shared/coding-standards.md + MEMORY规则 |
| 3.3 | 创建 | 新文件 | `playbooks/commit.md` | 提炼 AGENTS.md 提交相关规则 |
| 3.4 | 创建 | 新文件 | `playbooks/incident.md` | 提炼 P0事故响应流程 |
| 3.5 | 创建 | 新文件 | `todo/active.md` | TODO.md 前50项活跃事项 |
| 3.6 | 创建 | 新文件 | `archive/todo-history-2026-01.md` | TODO.md 剩余历史项 |
| 3.7 | 创建 | 工具脚本 | `.migration/scripts/split_todo.py` | 自动拆分TODO脚本 |
| 3.8 | 修改 | 更新 | `.agents/BRIDGE.md` | 更新为 Phase 3，标注双轨运行 |
| 3.9 | 创建 | 索引文件 | `.migration/TODO_SPLIT_INDEX.md` | 记录每条TODO的拆分去向 |
| 3.10 | 创建 | 兼容说明 | 各playbook顶部 | 标注与旧文件的对应关系 |

#### 详细步骤

**Step 1: TODO拆分策略**

```python
# split_todo.py 逻辑
# 1. 读取 TODO.md (20KB)
# 2. 解析每条TODO项（按 ## 或 - [ ] 分隔）
# 3. 分类:
#    - 状态为"进行中"、"待开始"、"高优先级" → active.md
#    - 状态为"已完成"、"已关闭"、低优先级过期项 → archive/
# 4. active.md 保持 ≤50 项
# 5. 生成索引文件记录每条TODO的去向
```

**Step 2: 创建 todo/active.md**

```markdown
# 活跃事项

> 本文件只保留 ≤50 项活跃TODO。
> 创建时间: 2026-01-12 | 来源: TODO.md 拆分
> 归档位置: archive/todo-history-2026-01.md

## 活跃项 (X/50)

- [ ] P1 迁移新架构 — 当前 Phase 3/6
- [ ] P1 整合P0事故文件
- [ ] P2 优化playbooks内容
# ... 最多50项

## 规则
1. 超过50项时，最老的已完成项自动归档
2. 每项必须有优先级标签 (P0/P1/P2)
3. 每项必须有状态标签 (待开始/进行中/待验证/已完成)
```

**Step 3: 创建场景手册**

**playbooks/startup.md** — 会话启动手册
```markdown
# 会话启动手册

## 1. 首次进入仓库时
1. 读取 meta-rules.md（了解规则宪法）
2. 读取 profile.md（了解项目上下文）
3. 读取 todo/active.md（了解当前任务）

## 2. 进入具体项目时
1. 读取 profile.md 中的项目速查
2. 读取 projects/{project}/MEMORY.md
3. 读取 todo/active.md 中该项目相关项

## 3. 继续已有会话时
1. 检查 .agents/BRIDGE.md 了解架构状态
2. 读取 todo/active.md 确认当前任务状态

---
*旧文件对应: .agents/skills/todo-manager/SKILL.md*
```

**playbooks/develop.md** — 开发手册
```markdown
# 开发手册

## 编码标准
- 引用: shared/coding-standards.md
- 核心要点: [提炼关键3-5条]

## 开发流程
1. 读取 MEMORY.md 了解项目历史
2. 检查活跃TODO中的相关项
3. 开发 → 自测 → 提交

## 决策记录
- 重大决策写入 projects/{project}/DECISIONS.md
- 参考 DECISIONS.md 格式: [说明]

## 记忆管理
- 项目级记忆: projects/{project}/MEMORY.md
- 全局记忆: profile.md 状态快照
- 记忆更新规则: [简要说明]

---
*旧文件对应: shared/coding-standards.md + projects/*/DECISIONS.md*
```

**playbooks/commit.md** — 提交手册
```markdown
# 提交手册

## 提交前检查
- [ ] 代码自测通过
- [ ] 相关TODO项状态已更新
- [ ] 重大决策已记录

## 提交格式
[标准提交格式]

## 提交后
- [ ] 更新 todo/active.md 对应项状态
- [ ] 如解决P0问题，按 incident.md 流程记录

---
*旧文件对应: AGENTS.md 提交相关章节*
```

**playbooks/incident.md** — 事故响应手册
```markdown
# 事故响应手册

## 事故分级
- P0: 系统不可用 / 数据丢失风险
- P1: 功能损坏 / 性能严重下降
- P2: 一般Bug / 体验问题

## P0响应流程
1. **止损**: 立即停止相关操作
2. **记录**: 创建 incident-log.md 条目
3. **定位**: 使用根因分析模板
4. **修复**: 实施修复
5. **验证**: 回归测试
6. **复盘**: 更新 incident-log.md

## incident-log.md 格式

```template
## INC-{编号}: {标题} | {日期} | {状态}

### 现象
[描述]

### 根因
[分析]

### 修复
[方案]

### 验证
[如何确认已修复]

### 复盘
[经验教训]
```

## 历史P0事故索引
| 编号 | 标题 | 日期 | 状态 | 原文件 |
|------|------|------|------|--------|
| INC-001 | Code Review问题 | - | 已解决 | P0_CODE_REVIEW_40.md |
| INC-002 | Dead Loop问题 | - | 已解决 | P0_DEADLOOP_41.md |
| ... | ... | ... | ... | ... |

---
*旧文件对应: P0_CODE_REVIEW_40.md, P0_DEADLOOP_41.md, P0_REGRESSION_*.md, P0_ROOT_CAUSE_39.md*
```

**Step 4: 创建 archive/todo-history-2026-01.md**

将TODO.md中超出50项的历史项移入此文件，按月份归档。

#### 交付物

| 交付物 | 路径 | 验收标准 |
|--------|------|----------|
| 会话启动手册 | `playbooks/startup.md` | AI能用3步启动会话 |
| 开发手册 | `playbooks/develop.md` | 包含编码标准引用、开发流程、决策记录规则 |
| 提交手册 | `playbooks/commit.md` | 提交前后检查清单 |
| 事故响应手册 | `playbooks/incident.md` | P0响应5步法 + 记录模板 |
| 活跃TODO | `todo/active.md` | ≤50项，每条有优先级和状态 |
| TODO历史归档 | `archive/todo-history-2026-01.md` | 所有历史项完整保留 |
| 拆分索引 | `.migration/TODO_SPLIT_INDEX.md` | 每条原TODO都有去向记录 |

#### 验证标准

- [ ] todo/active.md 项数 ≤ 50
- [ ] 原TODO.md 所有条目在 active.md 或 archive/ 中可找到
- [ ] 4个playbook覆盖主要协作场景
- [ ] AI助手按新playbook能完成完整工作流
- [ ] incident.md 包含所有5个P0文件的索引

#### 回滚方案

```
触发条件: TODO拆分遗漏关键项、playbook流程不通
回滚步骤:
  1. 保留新文件（不删除，仅标记为 [DEPRECATED]）
  2. 恢复 .agents/BRIDGE.md 标注"回退到旧架构"
  3. 团队继续使用 TODO.md 和旧P0文件
  4. 分析问题后重新进入Phase 3
回滚耗时: < 15 分钟
数据影响: 零（旧文件未动，新文件保留供分析）
```

---

### Phase 4: 整合 — P0事故整合 + 内容验证（预计 4-5 天）

**目标**: 整合P0事故文件为incident-log.md，全面验证新架构完整性

#### 动作清单

| 序号 | 动作 | 类型 | 文件路径 | 说明 |
|------|------|------|----------|------|
| 4.1 | 创建 | 新文件 | `incident-log.md` | Append-only 事故日志 |
| 4.2 | 整合 | 内容写入 | `incident-log.md` | 将5个P0文件转为incident条目 |
| 4.3 | 创建 | 迁移标记 | 每个旧P0文件顶部 | 添加 "[已整合至 incident-log.md]" |
| 4.4 | 更新 | 修改 | `playbooks/incident.md` | 更新incident-log.md引用 |
| 4.5 | 创建 | 验证脚本 | `.migration/scripts/verify_integrity.py` | 完整性验证脚本 |
| 4.6 | 创建 | 对比报告 | `.migration/INTEGRITY_REPORT.md` | 新旧内容对比报告 |
| 4.7 | 更新 | 修改 | `.agents/BRIDGE.md` | 更新为 Phase 4 |
| 4.8 | 创建 | 用户指引 | `.migration/NEW_ARCH_GUIDE.md` | 新架构使用指南 |

#### 详细步骤

**Step 1: 创建 incident-log.md**

```markdown
# 事故日志

> Append-only。按时间倒序排列。
> 创建时间: 2026-01-12 | 来源: P0事故文件整合

---

## INC-005: Regression Quick Fix | 202X-XX-XX | 已解决

### 现象
[摘自 P0_REGRESSION_QUICK.md]

### 根因
[分析]

### 修复
[方案]

### 验证
[方法]

### 复盘
[经验教训]

---

## INC-004: Regression Guard | 202X-XX-XX | 已解决
[同上格式]

---

## INC-003: Root Cause 39 | 202X-XX-XX | 已解决
[同上格式]

---

## INC-002: Dead Loop 41 | 202X-XX-XX | 已解决
[同上格式]

---

## INC-001: Code Review 40 | 202X-XX-XX | 已解决
[同上格式]

---
*原文件位置: projects/video_management/P0_*.md*
*迁移时间: 2026-01-12*
```

**Step 2: 标记旧P0文件**

在每个旧P0文件顶部添加：
```markdown
> ⚠️ [已整合] 本文件内容已迁移至 incident-log.md
> 保留此处仅作备份参考，将在 Phase 6 归档
> 权威版本: incident-log.md
```

**Step 3: 完整性验证**

运行验证脚本检查：
1. 旧文件每条规则在新文件中可找到
2. 没有规则被意外遗漏
3. TODO项全部有去向
4. P0事故全部整合
5. 交叉引用链接有效

**Step 4: 编写新架构使用指南**

```markdown
# 新架构使用指南

## 快速开始

### AI助手
1. 会话开始 → 读 meta-rules.md
2. 了解上下文 → 读 profile.md
3. 执行任务 → 按场景读对应 playbook
4. 记录事故 → 追加 incident-log.md
5. 管理TODO → 更新 todo/active.md

### 人类开发者
1. 查看项目状态 → profile.md
2. 开始开发 → playbooks/develop.md
3. 处理事故 → playbooks/incident.md
4. 回顾历史 → archive/ 或 incident-log.md

## 文件层级关系
```
meta-rules.md (最高权威)
    ↓
profile.md (项目上下文)
    ↓
playbooks/*.md (场景指导)
    ↓
projects/*/ (项目具体文件)
```
```

#### 交付物

| 交付物 | 路径 | 验收标准 |
|--------|------|----------|
| 事故日志 | `incident-log.md` | 包含5个P0事故，格式统一，内容完整 |
| 完整性验证报告 | `.migration/INTEGRITY_REPORT.md` | 所有规则/事项都有映射，无遗漏 |
| 新架构指南 | `.migration/NEW_ARCH_GUIDE.md` | AI和人工都能快速上手 |
| 旧文件迁移标记 | 各P0文件顶部 | 标注清晰，指向正确 |

#### 验证标准

- [ ] incident-log.md 包含所有5个P0事故的完整信息
- [ ] 每个P0原文件顶部有迁移标记
- [ ] 完整性验证脚本通过（100%映射）
- [ ] 新架构指南能指导新用户快速使用
- [ ] 双轨运行1-2天无问题报告

#### 回滚方案

```
触发条件: incident-log.md内容缺失、验证发现遗漏
回滚步骤:
  1. 保留 incident-log.md（标记 [DEPRECATED]）
  2. 移除旧P0文件的迁移标记
  3. 恢复使用独立的P0文件
  4. 修复问题后重新整合
回滚耗时: < 20 分钟
数据影响: 低（incident-log是汇总，源文件仍在）
```

---

### Phase 5: 切换 — 新架构为主（预计 3-5 天）

**目标**: 正式切换至新架构，旧文件进入只读备份状态

#### 动作清单

| 序号 | 动作 | 类型 | 文件路径 | 说明 |
|------|------|------|----------|------|
| 5.1 | 更新 | 修改 | `.agents/BRIDGE.md` | 标注"新架构为主，旧为参考" |
| 5.2 | 创建 | 重定向文件 | `AGENTS.md` 顶部 | 添加指向 meta-rules.md 的引导 |
| 5.3 | 创建 | 重定向文件 | `README.md` 顶部 | 添加指向 profile.md 的引导 |
| 5.4 | 创建 | 重定向文件 | `TODO.md` 顶部 | 添加指向 todo/active.md 的引导 |
| 5.5 | 更新 | 修改 | `meta-rules.md` | 添加"生效声明"：本文件为当前权威来源 |
| 5.6 | 更新 | 修改 | `profile.md` | 添加"生效声明" |
| 5.7 | 创建 | 监控文件 | `.migration/SWITCH_MONITORING.md` | 切换期问题追踪 |
| 5.8 | 通知 | 团队沟通 | - | 通知全员切换 |

#### 详细步骤

**Step 1: 更新桥梁文件**

```markdown
# 架构迁移桥梁文件

> ## ⚠️ 当前阶段: Phase 5 — 新架构为主
> 新架构文件是当前权威来源。
> 旧文件仅作历史参考，内容可能已过时。

## 权威文件位置

| 旧文件 | 新权威文件 | 旧文件状态 |
|--------|-----------|-----------|
| AGENTS.md | meta-rules.md | [参考备份] |
| README.md | profile.md | [参考备份] |
| TODO.md | todo/active.md | [参考备份] |
| P0_*文件 | incident-log.md | [参考备份] |
```

**Step 2: 旧文件添加重定向**

在 AGENTS.md、README.md、TODO.md 顶部添加：
```markdown
# ⚠️ 本文件已进入维护模式

> **权威版本已迁移**，请使用新架构文件：
> - 新位置: [对应新文件路径]
> - 本文件保留作为历史参考，内容不再更新
> - 迁移详情: .migration/MIGRATION_LOG.md

---
[原文件内容不变]
```

**Step 3: 新文件添加生效声明**

在 meta-rules.md、profile.md 等文件添加：
```markdown
# 元规则

> **状态: 生效中** | 创建: 2026-01-12
> 替代旧文件: AGENTS.md（已进入维护模式）
```

**Step 4: 切换监控**

创建监控文件，追踪切换期间的问题：
```markdown
# Phase 5 切换监控

## 监控期: 2026-01-XX 至 2026-01-XX（5个工作日）

### 每日检查
- [ ] AI助手是否正确读取新文件？
- [ ] 有无"找不到规则"的情况？
- [ ] 团队反馈新架构是否好用？
- [ ] todo/active.md 是否正常工作？

### 问题记录
| 日期 | 问题 | 严重度 | 处理 | 状态 |
|------|------|--------|------|------|
| | | | | |

### 切换成功标准
- [ ] 5个工作日内无P0/P1问题
- [ ] 团队确认新架构更易用
- [ ] AI助手按新架构工作无异常
```

#### 交付物

| 交付物 | 路径 | 验收标准 |
|--------|------|----------|
| 更新后的桥梁文件 | `.agents/BRIDGE.md` | 明确标注新架构为主 |
| 重定向旧文件 | `AGENTS.md`, `README.md`, `TODO.md` | 顶部有清晰重定向指引 |
| 生效声明 | 各新文件 | 明确标注当前为权威来源 |
| 监控记录 | `.migration/SWITCH_MONITORING.md` | 5个工作日无P0/P1问题 |

#### 验证标准

- [ ] 新会话的AI助手优先读取新架构文件
- [ ] 旧文件重定向清晰可见
- [ ] 5个工作日监控期内无严重问题
- [ ] 团队书面确认新架构可用
- [ ] 所有新文件内容稳定，无频繁修改

#### 回滚方案

```
触发条件: 5天监控期内出现≥2个P1问题或≥1个P0问题
回滚步骤:
  1. 立即更新 BRIDGE.md 标注"回退到旧架构为主"
  2. 移除旧文件顶部的重定向（恢复为正常状态）
  3. 新文件标记为 [BETA]
  4. 恢复使用旧文件作为主要参考
  5. 分析问题根因
  6. 修复后重新进入 Phase 5
回滚耗时: < 30 分钟
数据影响: 中（监控期新增TODO可能需同步）
```

---

### Phase 6: 清理 — 旧文件归档（预计 2-3 天）

**目标**: 将旧文件移入归档目录，完成迁移

#### 动作清单

| 序号 | 动作 | 类型 | 文件路径 | 说明 |
|------|------|------|----------|------|
| 6.1 | 移动 | 归档 | `archive/legacy/AGENTS.md` | 原AGENTS.md |
| 6.2 | 移动 | 归档 | `archive/legacy/README.md` | 原README.md |
| 6.3 | 移动 | 归档 | `archive/legacy/TODO.md` | 原TODO.md（完整备份） |
| 6.4 | 移动 | 归档 | `archive/legacy/P0_*.md` | 5个P0原文件 |
| 6.5 | 移动 | 归档 | `archive/legacy/skills/` | todo-manager技能目录 |
| 6.6 | 删除 | 删除 | `.agents/BRIDGE.md` | 桥梁文件完成使命 |
| 6.7 | 更新 | 修改 | `.migration/MIGRATION_LOG.md` | 标记迁移完成 |
| 6.8 | 创建 | 完成报告 | `.migration/COMPLETION_REPORT.md` | 迁移完成报告 |
| 6.9 | 创建 | 新文件 | `CHANGELOG.md` | 架构变更记录 |

#### 详细步骤

**Step 1: 旧文件归档**

```bash
mkdir -p archive/legacy/
mv AGENTS.md archive/legacy/AGENTS.md
cp README.md archive/legacy/README.md  # 保留根目录一份
cp TODO.md archive/legacy/TODO.md      # 保留完整备份
mv projects/video_management/P0_*.md archive/legacy/
mv .agents/skills/todo-manager archive/legacy/skills-todo-manager/
```

**Step 2: 清理桥梁文件**

```bash
rm .agents/BRIDGE.md
```

**Step 3: 编写完成报告**

```markdown
# 架构重构迁移完成报告

## 迁移概况
- 迁移期间: 2026-01-XX 至 2026-01-XX
- 总阶段数: 6
- 实际耗时: X 个工作日
- 回滚次数: 0

## 文件变更统计

### 新建文件
- meta-rules.md (X KB)
- profile.md (X KB)
- playbooks/startup.md (X KB)
- playbooks/develop.md (X KB)
- playbooks/commit.md (X KB)
- playbooks/incident.md (X KB)
- incident-log.md (X KB)
- todo/active.md (X 项)
- archive/todo-history-2026-01.md (X 项)

### 归档文件
- archive/legacy/AGENTS.md
- archive/legacy/TODO.md
- archive/legacy/P0_*.md (5个)

## 验证结果
- [x] 完整性验证: 100% 规则映射
- [x] 双轨运行: 无问题
- [x] 切换监控: 5个工作日无P1/P0问题
- [x] 团队确认: 已书面确认

## 经验教训
[记录迁移过程中的经验]

## 后续建议
- 每季度审查 meta-rules 有效性
- 定期归档 todo/active.md 已完成项
- incident-log.md 保持 append-only
```

**Step 4: 创建 CHANGELOG.md**

```markdown
# 变更日志

## [2.0.0] 2026-01-XX — 架构重构

### 架构变更
- 采用三层治理模型：元规则 → 角色上下文 → 场景手册
- 取代旧平铺文件结构

### 新增
- meta-rules.md: 元规则层（3-5条宪法级规则）
- profile.md: 角色与上下文层
- playbooks/: 场景手册目录
  - startup.md, develop.md, commit.md, incident.md
- incident-log.md: Append-only 事故日志
- todo/active.md: 活跃事项（≤50项）
- archive/: 历史归档目录

### 归档
- AGENTS.md → archive/legacy/
- TODO.md → archive/legacy/（完整备份）
- P0_*文件 → archive/legacy/（整合入 incident-log.md）
- .agents/skills/todo-manager/ → archive/legacy/
```

#### 交付物

| 交付物 | 路径 | 验收标准 |
|--------|------|----------|
| 归档旧文件 | `archive/legacy/` | 所有旧文件完整保留 |
| 完成报告 | `.migration/COMPLETION_REPORT.md` | 统计数据完整，经验记录 |
| 变更日志 | `CHANGELOG.md` | 格式清晰，变更完整 |
| 清理后的仓库 | 根目录 | 只保留新架构活跃文件 |

#### 验证标准

- [ ] 所有旧文件在 archive/legacy/ 中可找到
- [ ] 根目录只保留新架构文件
- [ ] 新架构文件内容正确，链接有效
- [ ] AI助手能在无BRIDGE.md的情况下正常工作
- [ ] 团队确认迁移完成

#### 回滚方案

```
触发条件: 极少数情况，归档后发现严重遗漏
回滚步骤:
  1. 从 archive/legacy/ 恢复需要的旧文件到原位置
  2. 更新新文件标注"部分回退"
  3. 如有需要，重新创建 BRIDGE.md
  4. 修复问题后再次执行 Phase 6
回滚耗时: < 30 分钟
数据影响: 低（所有文件都在archive中）
```

---

## 三、风险分析

### 3.1 全局风险

| 风险ID | 风险描述 | 概率 | 影响 | 缓解措施 |
|--------|----------|------|------|----------|
| R1 | TODO拆分遗漏关键活跃项 | 中 | 高 | 拆分脚本生成索引表；人工抽检20% |
| R2 | P0事故整合丢失细节 | 中 | 高 | 逐文件人工审核；保留原文件不删除 |
| R3 | 元规则提炼过度简化 | 低 | 高 | 与AGENTS.md逐条对比；团队评审 |
| R4 | AI助手不遵循新架构 | 中 | 中 | BRIDGE.md引导；渐进切换；监控反馈 |
| R5 | 团队成员不适应新结构 | 中 | 中 | 编写使用指南；保留旧文件参考；培训 |
| R6 | 迁移期间新需求产生 | 高 | 低 | 双轨运行；新需求按新架构处理 |
| R7 | 回滚后数据不一致 | 低 | 高 | 只增不改旧文件；变更日志追踪 |
| R8 | 迁移工具脚本出错 | 中 | 中 | 脚本先测备份；手工核对结果 |

### 3.2 每阶段风险详表

**Phase 1 风险**
| 风险 | 概率 | 影响 | 缓解 |
|------|------|------|------|
| 备份不完整 | 低 | 极高 | 双重备份 + 校验和验证 |
| 文件清单遗漏 | 低 | 高 | 多种方式交叉核对 |

**Phase 2 风险**
| 风险 | 概率 | 影响 | 缓解 |
|------|------|------|------|
| 元规则提炼遗漏 | 中 | 高 | 与AGENTS.md逐条映射 |
| profile信息不全 | 中 | 中 | 对照README.md逐项检查 |

**Phase 3 风险**
| 风险 | 概率 | 影响 | 缓解 |
|------|------|------|------|
| TODO拆分错误 | 中 | 高 | 索引表 + 人工抽检 |
| Playbook内容不足 | 中 | 中 | 对照旧文件逐项验证 |
| 活跃项超过50 | 高 | 低 | 强制截断，多出的归档 |

**Phase 4 风险**
| 风险 | 概率 | 影响 | 缓解 |
|------|------|------|------|
| P0整合丢失 | 中 | 高 | 逐文件对比；保留原文件 |
| 验证脚本漏报 | 低 | 高 | 多种验证方式交叉 |

**Phase 5 风险**
| 风险 | 概率 | 影响 | 缓解 |
|------|------|------|------|
| 切换后AI找不到规则 | 中 | 高 | 5天监控期；快速回滚能力 |
| 团队不适应 | 中 | 中 | 使用指南；旧文件仍可访问 |
| 新旧规则冲突 | 低 | 高 | 规则对比审查；明确优先级 |

**Phase 6 风险**
| 风险 | 概率 | 影响 | 缓解 |
|------|------|------|------|
| 归档后发现问题 | 低 | 中 | 旧文件在archive中可恢复 |
| 误删活跃文件 | 低 | 极高 | 只移不删；核对清单 |

### 3.3 回滚触发条件汇总

| 阶段 | 触发条件 | 自动/人工 |
|------|----------|-----------|
| Phase 1 | 审计发现数据完整性问题 | 人工 |
| Phase 2 | 新文件核心规则遗漏 | 人工 |
| Phase 3 | TODO关键项丢失、playbook流程不通 | 人工+自动验证 |
| Phase 4 | 完整性验证未通过 | 自动阻止进入Phase 5 |
| Phase 5 | 5天内≥2个P1或≥1个P0问题 | 人工 |
| Phase 6 | 发现严重遗留问题 | 人工 |

---

## 四、并行处理策略

### 4.1 依赖关系图

```
Phase 1: 准备与审计
    │
    ▼
Phase 2: 创建 Layer 1 + Layer 2
    │
    ├──→ meta-rules.md ──────────────┐
    ├──→ profile.md ────────────────┤
    │                                 │
    ▼                                 │
Phase 3: TODO拆分 + Playbooks        │
    │                                 │
    ├──→ todo/active.md              │
    ├──→ todo/archive/               │ 可并行
    ├──→ playbooks/startup.md        │
    ├──→ playbooks/develop.md        │
    ├──→ playbooks/commit.md         │
    └──→ playbooks/incident.md ──────┘
    │                                 │
    ▼                                 ▼
Phase 4: P0整合 + 验证 ◄─────────────┘
    │        │
    │        └── 需要 Phase 2 + Phase 3 都完成
    ▼
Phase 5: 切换为主
    │
    └── 需要 Phase 4 验证通过
    ▼
Phase 6: 归档清理
    │
    └── 需要 Phase 5 监控通过
```

### 4.2 可并行工作项

**Phase 2 内部并行**
- meta-rules.md 和 profile.md 可同时编写
- 目录结构创建与文件编写并行

**Phase 3 内部并行**
- TODO拆分（技术工作）和 Playbooks编写（内容工作）可并行
- 4个Playbook之间可并行编写

**跨阶段并行（特殊安排）**
- Phase 3 的 todo/archive/ 可以在 Phase 2 就开始准备
- Phase 4 的验证脚本可以在 Phase 2 就开始编写

### 4.3 必须串行的项

| 串行链 | 原因 |
|--------|------|
| Phase 1 → 所有后续阶段 | 必须先完成审计和备份 |
| Phase 2 → Phase 3 | Playbooks需要引用profile.md内容 |
| Phase 3 → Phase 4 | 验证需要所有内容就绪 |
| Phase 4 → Phase 5 | 切换前必须通过完整性验证 |
| Phase 5 → Phase 6 | 归档前必须通过监控期 |

### 4.4 加速策略

1. **预编写**: Phase 1 期间就可以开始起草 meta-rules.md 和 profile.md
2. **模板化**: Playbooks使用统一模板，加快编写速度
3. **脚本自动化**: TODO拆分、P0整合、完整性验证都使用脚本
4. **增量验证**: 每个文件创建后立即验证，不堆积到最后

---

## 五、验证清单

### 5.1 每阶段验收标准

#### Phase 1 验收
- [ ] 备份文件可完整还原仓库
- [ ] SHA256校验和与当前文件100%匹配
- [ ] 新旧文件映射表覆盖所有文件
- [ ] 废弃时间表合理可行
- [ ] 团队成员已审阅迁移计划

#### Phase 2 验收
- [ ] meta-rules.md 提炼自AGENTS.md，无关键遗漏
- [ ] profile.md 整合自README.md，信息完整
- [ ] 目录结构创建正确
- [ ] 新文件顶部有旧文件对应标注
- [ ] AI助手能读取并理解新文件

#### Phase 3 验收
- [ ] todo/active.md ≤ 50项
- [ ] 原TODO.md每条都有去向（索引表可查）
- [ ] 4个playbook内容完整、流程清晰
- [ ] playbook与旧文件内容映射正确
- [ ] 拆分脚本可重复运行且结果一致

#### Phase 4 验收
- [ ] incident-log.md 包含5个P0事故完整信息
- [ ] 原P0文件顶部有迁移标记
- [ ] 完整性验证脚本100%通过
- [ ] 内容对比报告无遗漏项
- [ ] 新架构使用指南清晰可用

#### Phase 5 验收
- [ ] 5个工作日监控期无P0/P1问题
- [ ] AI助手自动读取新架构文件
- [ ] 团队书面确认新架构可用
- [ ] 新旧文件并行期间无冲突
- [ ] 所有活跃TODO在新架构中可管理

#### Phase 6 验收
- [ ] 所有旧文件在archive/legacy/中可找到
- [ ] 根目录无遗留旧架构文件（除CHANGELOG等）
- [ ] 新架构文件链接引用全部有效
- [ ] 迁移日志和完成报告完整
- [ ] 仓库可在无BRIDGE.md情况下正常工作

### 5.2 "规则遗漏"检测方案

这是本次迁移要解决的核心问题。采用多层检测：

**Layer 1: 自动化验证**
```python
# verify_integrity.py 核心逻辑
def check_rule_coverage():
    # 1. 提取旧文件所有规则条目
    old_rules = extract_rules_from_old_files()
    
    # 2. 提取新文件所有规则条目
    new_rules = extract_rules_from_new_files()
    
    # 3. 规则映射检查
    for rule in old_rules:
        if not find_in_new(rule, new_rules):
            report_missing(rule)
    
    # 4. 反向检查（新规则是否有来源）
    for rule in new_rules:
        if rule.source == "new" and not rule.approved:
            report_unapproved(rule)
    
    # 5. 生成覆盖率报告
    return coverage_report
```

**Layer 2: 人工审查**
- 每阶段至少1名团队成员审阅新文件
- 重点审查：元规则是否覆盖原AGENTS.md核心规则
- P0事故整合后，由原文件作者确认

**Layer 3: 运行时监控**
- Phase 5 监控期记录所有"找不到规则"的情况
- AI助手会话日志中搜索旧文件名引用
- 团队反馈渠道收集遗漏报告

**Layer 4: 定期审计**
- 迁移完成后1周、1个月、3个月各审计一次
- 检查是否有工作流依赖旧文件
- 检查新架构是否有效减少"规则遗漏"问题

### 5.3 成功度量指标

| 指标 | 迁移前 | 目标（迁移后） |
|------|--------|---------------|
| 核心规范文件数 | 2 (AGENTS + README) | 2 (meta-rules + profile) |
| 散落规则文件数 | 10+ | 4 (playbooks) + 1 (incident-log) |
| TODO文件大小 | 20KB | active ≤50项 + 按月归档 |
| P0事故文件数 | 5个独立文件 | 1个append-only日志 |
| 新AI上手时间 | 需读多个文件 | 3步（元规则→上下文→场景手册）|
| 规则查找成功率 | ~70% | >95% |

---

## 六、回滚计划

### 6.1 完整回滚路径

```
                    ┌─────────────────┐
                    │   任何阶段发现问题  │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
         阶段内回滚      跨阶段回滚       完全回滚
         (最快)         (中等)          (最慢)
              │              │              │
    ┌─────────┘    ┌─────────┘    ┌─────────┘
    ▼              ▼              ▼
 修正后继续    回退到上一阶段    回到Phase 0
    │              │              │
    ▼              ▼              ▼
  当前阶段      重新执行        仓库完全还原
```

### 6.2 各阶段回滚详情

#### 阶段内回滚（推荐首选）

| 阶段 | 场景 | 操作 | 耗时 |
|------|------|------|------|
| Phase 2 | meta-rules内容不对 | 修改文件，重新验证 | 30分钟 |
| Phase 3 | TODO拆分有误 | 修正脚本/手工调整，重新生成 | 1小时 |
| Phase 4 | P0整合有遗漏 | 补充incident-log.md | 30分钟 |

#### 跨阶段回滚

**从 Phase 3 回退到 Phase 2**
```bash
# 保留 playbook 和 todo 文件但标记为 DEPRECATED
for f in playbooks/*.md todo/*.md; do
    sed -i '1s/^/> [DEPRECATED] 此文件内容待修正\n\n/' "$f"
done
# 恢复 BRIDGE.md 到 Phase 2 状态
# 继续按 Phase 2 状态工作
```

**从 Phase 5 回退到 Phase 4**
```bash
# 1. 更新 BRIDGE.md 标注回退
# 2. 移除旧文件顶部的重定向标记
# 3. 新文件标记为 [BETA]
# 4. 恢复 incident-log.md 为参考状态（非权威）
# 5. 重新进入 Phase 4 验证
```

#### 完全回滚（Phase 0）

```bash
#!/bin/bash
# rollback_to_phase0.sh
# 完全回滚到迁移前状态

echo "=== 完全回滚到 Phase 0 ==="

# 1. 保留迁移日志
cp -r .migration /tmp/ai_hcy_migration_backup/

# 2. 从 Phase 1 备份还原
tar xzf .migration/backup/phase0/ai_hcy-*.tar.gz --overwrite

# 3. 删除迁移期间创建的所有文件
clean_migration_artifacts() {
    rm -f meta-rules.md
    rm -f profile.md
    rm -rf playbooks/
    rm -rf archive/
    rm -rf todo/
    rm -f incident-log.md
    rm -rf .migration/
    rm -f .agents/BRIDGE.md
    rm -f CHANGELOG.md
}

# 4. 验证还原结果
if verify_checksum .migration/backup/phase0/checksum.baseline; then
    echo "回滚成功！仓库已还原到迁移前状态。"
    echo "迁移日志保存在 /tmp/ai_hcy_migration_backup/"
else
    echo "警告：校验和不匹配，请手动检查！"
    exit 1
fi
```

### 6.3 回滚触发条件决策树

```
发现问题
    │
    ├── 是否影响当前工作？
    │       ├── 否 → 记录问题，继续迁移
    │       └── 是 →
    │               ├── 能否快速修复（<30分钟）？
    │               │       ├── 是 → 阶段内回滚，修复后继续
    │               │       └── 否 →
    │               │               ├── 是否影响核心功能？
    │               │               │       ├── 否 → 跨阶段回退，修复后重试
    │               │               │       └── 是 →
    │               │               │               ├── 是否多阶段问题？
    │               │               │                       ├── 否 → 回退到上一阶段
    │               │               │                       └── 是 → 完全回滚到Phase 0
    │               │               │
    └── 团队是否一致同意继续？
            └── 否 → 暂停迁移，讨论决策
```

### 6.4 数据一致性保障

| 保障措施 | 说明 |
|----------|------|
| 只增不改 | 前4阶段不修改任何旧文件内容 |
| 完整备份 | 每阶段开始前有备份 |
| 变更日志 | 每次文件操作都记录 |
| 校验和验证 | 关键文件有SHA256校验 |
| 双向映射 | 新旧文件间有明确映射关系 |
| 索引追踪 | TODO拆分有完整索引 |
| 脚本可重复 | 自动化脚本可重复运行验证 |

---

## 七、附录

### 附录A: 迁移时间线总览

```
第1周                    第2周                    第3周
─────────────────────────────────────────────────────────
Phase 1  Phase 2    Phase 3          Phase 4    Phase 5  Phase 6
(2天)    (3天)      (5天)            (4天)      (5天)    (2天)
├────────┼──────────┼────────────────┼──────────┼────────┼──────┤
备份审计   创建骨架    TODO拆分+         P0整合+    切换为主  归档
          元规则+上下文  Playbooks创建     完整性验证  监控期   完成
```

**总计预计: 19-21 个工作日（约4周，含缓冲）**

### 附录B: 文件操作矩阵

| 文件 | Phase 1 | Phase 2 | Phase 3 | Phase 4 | Phase 5 | Phase 6 |
|------|---------|---------|---------|---------|---------|---------|
| AGENTS.md | 备份 | - | - | 标记 | 重定向 | 移至archive |
| README.md | 备份 | - | - | - | 重定向 | 保留根目录 |
| TODO.md | 备份 | - | 拆分 | - | 重定向 | 移至archive |
| P0_*.md | 备份 | - | - | 整合 | 标记 | 移至archive |
| meta-rules.md | - | 创建 | - | - | 生效声明 | 保持 |
| profile.md | - | 创建 | - | - | 生效声明 | 保持 |
| playbooks/*.md | - | 目录 | 创建 | - | - | 保持 |
| todo/active.md | - | 目录 | 创建 | - | - | 保持 |
| incident-log.md | - | - | - | 创建 | - | 保持 |
| BRIDGE.md | 创建 | 更新 | 更新 | 更新 | 更新 | 删除 |

### 附录C: 快速参考卡片

```
═══════════════════════════════════════════════════
           AI_Hcy 新架构速查卡
═══════════════════════════════════════════════════

📋 会话启动（3步）
   1. meta-rules.md    ← 规则宪法
   2. profile.md       ← 我在哪、做什么
   3. todo/active.md   ← 当前任务

📖 场景手册
   startup.md    → 会话启动
   develop.md    → 开发工作
   commit.md     → 提交变更
   incident.md   → 事故响应

📝 记录位置
   事故 → incident-log.md (追加)
   TODO → todo/active.md (≤50项)
   历史 → archive/

⚠️ 紧急回滚
   查看 .migration/ 备份目录
   执行 rollback_to_phase0.sh
═══════════════════════════════════════════════════
```

### 附录D: 迁移检查清单（每日使用）

```markdown
## 每日迁移检查

### 开始工作前
- [ ] 读取当前迁移阶段 (BRIDGE.md 或 meta-rules.md)
- [ ] 确认使用哪个架构版本
- [ ] 检查 todo/active.md 是否有迁移相关任务

### 工作中
- [ ] 按当前阶段要求操作
- [ ] 如有问题记录到 .migration/MIGRATION_LOG.md
- [ ] 不要跳过阶段要求

### 结束工作
- [ ] 更新 todo/active.md 相关项状态
- [ ] 如完成阶段目标，更新 BRIDGE.md 阶段标记
- [ ] 提交时标注是否与迁移相关
```

---

> **文档结束**
>
> 本迁移计划遵循"桥梁并行"策略，确保AI协作零停机。
> 如有疑问，查看 .migration/MIGRATION_LOG.md 获取最新状态。
