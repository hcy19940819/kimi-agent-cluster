# Kimi Agent Cluster — AI协作规范治理方案

> 从复杂度失控到极简治理的完整方案
> V2新增: 规则可视化 + 三门控流程 + 工具链优化

---

## 快速导航

| 你想解决什么问题 | 先看哪个文件 |
|-----------------|-------------|
| **规则自行膨胀、AI擅自添加规则** | `v2/GOVERNANCE-V2-FULL.md` → 第二部分 |
| **需求不明确就执行、臆测需求** | `v2/GOVERNANCE-V2-FULL.md` → 第三部分 |
| **Kimi Code不好用、工具优化** | `v2/GOVERNANCE-V2-FULL.md` → 第四部分 |
| **一站式完整方案（推荐）** | `v2/GOVERNANCE-V2-FULL.md` |

---

## 仓库结构

```
kimi-agent-cluster/
├── README.md                           # 本文件
│
# ===== V1: 架构治理方案 =====
├── PROPOSAL.md                         # V1主方案（原中文文件名）
├── diagnosis-report.md                 # V1诊断报告
├── migration-plan.md                   # V1迁移计划
│
├── docs/                               # V1详细设计文档
│   ├── 00-ARCHITECTURE-OVERVIEW.md
│   ├── 01-MEMORY-COMPRESSION.md
│   ├── 02-MIGRATION-GUIDE.md
│   ├── 03-RULE-MAPPING.md
│   └── 04-AUTOMATION-SCRIPTS.md
│
├── templates/                          # V1文件模板
│   ├── META.md
│   ├── CONTEXT.md
│   ├── SKILLS.md
│   └── playbook/
│       ├── start.md
│       ├── develop.md
│       ├── submit.md
│       └── incident.md
│
# ===== V2: 完整治理 + 流程 + 工具 =====
├── v2/
│   ├── GOVERNANCE-V2-FULL.md           # ⭐ V2完整方案（一站式阅读）
│   ├── RULES-GOVERNANCE-GUIDE.md       # 规则可视化 + 变更确认详细指南
│   ├── ai-collaboration-three-gate-process.md  # 三门控流程完整文档
│   ├── kimi26_toolchain_guide.md       # Kimi2.6工具链完全指南
│   ├── RULES-GOVERNANCE.md             # 可直接使用的规则看板模板
│   ├── RULES-CHANGELOG.md              # 可直接使用的变更日志模板
│   └── RULES-PROPOSAL-TEMPLATE.md      # 可直接使用的变更提案模板
```

---

## V2 核心内容概览

### Q1: 规则可视化 + 变更确认

- **三层金字塔**: 元规则(3条) / 执行规则(10条) / 场景规则(10条)，上限20条
- **可视化看板**: `RULES-GOVERNANCE.md` — 打开即看规则全貌
- **变更确认流程**: AI提案 → 你审批（同意/试运行/否决） → AI执行
- **AI红线**: 禁止先斩后奏、禁止默认同意、禁止拆分规避
- **三级告警**: 🟡警告(16条) → 🟠紧急(19条) → 🔴超标(>20条)

### Q2: 需求 → 方案 → 执行 三门控

```
用户输入 → Gate1 需求确认 → Gate2 方案评审 → Gate3 执行验证 → 关闭
              │                 │                 │
           REQ文档          DES文档          VER文档
           (你确认)          (你批准)         (你验收)
```

- **Gate 1**: AI只做需求分析师，理解记录不编码
- **Gate 2**: AI只做方案设计师，出方案不编码
- **Gate 3**: AI按方案执行，完成后自检提交验证
- **违规处理**: VIO-01~VIO-07，违规立即停止回流

### Q3: 工具链优化

- **立即迁移**: Kimi Code → Trae国内版（10分钟，免费，中文原生）
- **核心模型**: Kimi K2.6（代码编写 + 方案设计）
- **辅助模型**: Kimi K2.5（测试 + 文档，成本更低）
- **Prompt最佳实践**: 四段式写法 + 长上下文利用 + 前缀缓存

---

## 生成信息

- **V1日期**: 2026-04-29（架构治理方案）
- **V2日期**: 2026-04-29（完整治理 + 流程 + 工具）
- **诊断对象**: hcy19940819/AI_Hcy
- **生成方式**: 多Agent协作（规则治理Agent + 流程门控Agent + 工具链Agent）


---

## 新增: OpenClaw × Kimi 协作开发方案

针对你的实际工具链（OpenClaw + Kimi K2.6），设计了完整的 Skill 化协作方案。

### 仓库结构

```
kimi-agent-cluster/
├── ... (V1/V2 文件)
│
├── openclaw/                          # ⭐ OpenClaw 协作方案
│   ├── SCHEME.md                      # 完整方案文档（配置步骤 + 使用示例）
│   │
│   ├── configs/                       # 配置文件（复制到 ~/.openclaw/workspace/）
│   │   ├── AGENTS.md                  # 行为准则（三门控 + 规则治理）
│   │   ├── SOUL.md                    # 人格定义（风格 + 语气）
│   │   ├── USER.md                    # 用户偏好（你的习惯 + 项目信息）
│   │   └── TOOLS.md                   # 工具速查（命令 + 路径）
│   │
│   └── skills/                        # 4 个 Skill（对应三门控流程）
│       ├── requirement-gate/          # 需求确认门（自动触发，绝不编码）
│       │   └── SKILL.md
│       ├── design-gate/               # 方案评审门（自动触发，绝不编码）
│       ├── code-execute/              # 代码执行（仅手动 /code 触发）
│       │   └── SKILL.md
│       └── verify-gate/               # 验证门（编码完成后自动触发）
│           └── SKILL.md
```

### 核心设计

把 "需求→方案→执行" 映射为 **OpenClaw Skill 系统**的 4 个 Skill：

| Skill | 阶段 | 触发方式 | AI 角色 | 权限 |
|-------|------|---------|--------|------|
| `requirement-gate` | Gate 1 | 自动（关键词匹配） | 需求分析师 | 只读 |
| `design-gate` | Gate 2 | 自动（"确认"后） | 方案设计师 | 只读 |
| `code-execute` | Gate 3 | **手动 /code** | 执行工程师 | 可编辑 |
| `verify-gate` | Gate 3 | 自动（"完成"后） | 验证工程师 | 只读 |

**关键保障**: `code-execute` 设为 `disable-model-invocation: true`，**只能手动触发**，防止 AI 擅自编码。

### 快速开始

```bash
# 1. 安装 OpenClaw
npm install -g openclaw

# 2. 配置 Kimi
openclaw onboard --provider moonshot
openclaw models set moonshot/kimi-k2.6

# 3. 创建工作区
mkdir -p ~/.openclaw/workspace/skills

# 4. 复制配置文件
cp openclaw/configs/* ~/.openclaw/workspace/
cp -r openclaw/skills/* ~/.openclaw/workspace/skills/

# 5. 启动协作
openclaw agent --message "我是谁？"
```

### 进阶方向

- **Heartbeat 监控**: 配置 HEARTBEAT.md 定期检查项目健康
- **多渠道集成**: Telegram/微信 远程触发任务
- **子 Agent 架构**: 不同 Gate 用不同模型（需求用轻量，编码用 K2.6）
- **规则膨胀检测**: AGENTS.md 中添加规则计数器
