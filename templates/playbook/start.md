# PLAYBOOK — 会话启动

> 目标：3步完成启动，获取完整上下文。
> 替代旧架构的5步启动协议。

---

## 启动检查清单（必须逐项确认）

```
□ Step 1: 读取 META.md（确认约束）
  命令: cat META.md
  目的: 了解当前执行规则（R1-R15）

□ Step 2: 读取 CONTEXT.md（获取上下文）
  命令: cat CONTEXT.md
  目的: 了解项目状态和本次任务

□ Step 3: 确认本次目标
  格式: "本次目标：[任务描述]，来源：[user/tech-debt/incident#N]"
  确认人: user
```

---

## 快速决策树

```
用户是否有明确任务？
├── 是 → 读取 CONTEXT.md #Session → 确认目标 → 进入开发
├── 否 → 读取 CONTEXT.md #Project → 询问用户目标
└── 不确定 → 询问 "您希望本次会话完成什么？"
```

## 规则速查（启动相关）

| 规则 | 操作 |
|------|------|
| R1 | 运行 `scripts/pre-flight.sh` |
| R12 | 确认 CONTEXT.md #Session 有内容 |
| R11 | 确认 META.md 存在（单一来源） |
