# CONTEXT — 项目快照

> 每次会话后自动更新 #Session 部分，其他部分手动维护。
> 目标大小：<1KB

---

## #Project

| 属性 | 值 |
|------|-----|
| 名称 | video_management |
| 路径 | `/path/to/project` |
| 技术栈 | Python 3.11 / Django / PostgreSQL |
| 版本 | v2.3.1 |
| 关键配置 | `settings.py` 第42行: DEBUG=False |

## #Constraints（仅最高频3-5条）

```
- GPU并行度 ≤ 4（硬件限制）
- 数据库不允许直接迁移（必须通过migration）
- 存储分层：hot→warm→cold，禁止跨层直接操作
```

## #Session（每次会话更新）

```
目标: 修复视频上传进度条不更新的bug
来源: user
时间: 2024-01-15 09:00
关联: incident.md #42
```

## #QuickRef（快速参考命令）

```bash
# 运行测试
pytest tests/ -x -q

# 安全检查
./scripts/verify.sh

# 数据库迁移检查
python manage.py showmigrations | grep -E "\\[ \\]"
```
