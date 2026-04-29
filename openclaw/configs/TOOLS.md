# TOOLS.md — 工具使用说明

> 位置: ~/.openclaw/workspace/TOOLS.md
> 作用: 教 AI 怎么用项目中的工具

---

## 项目命令速查

### 启动服务
```bash
cd D:\AI\AI_Tool\Video_management
python app.py
```

### 运行测试
```bash
pytest tests/ -x -q
```

### 代码检查
```bash
ruff check .
ruff format .
```

### 前端构建
```bash
python build_frontend.py
```

### 回归检查
```bash
python scripts/check_regression.py
# 或
.\check_regression.ps1
```

### 数据库备份
```bash
python scripts/db_safety_check.py --strict
```

## 常用路径

| 用途 | 路径 |
|------|------|
| 视频管家项目 | D:\AI\AI_Tool\Video_management |
| 数据库 | D:\AI\AI_Tool\Video_management\data\video_master.db |
| 前端源码 | D:\AI\AI_Tool\Video_management\parts/ |
| 测试目录 | D:\AI\AI_Tool\Video_management\tests/ |
| AI_Hcy 规范 | D:\AI\AI_Hcy |

## Git 常用

```bash
# 检查状态
git status

# 提交（遵循规范: <type>(<scope>): <subject>）
git commit -m "feat(recognize): add priority sorting"

# 查看最近提交
git log --oneline -10
```

## 约束

- Windows 环境，使用 PowerShell 或 CMD
- 路径使用反斜杠或正斜杠均可，但 PowerShell 中反斜杠需转义
- 中文路径可能编码问题，优先使用英文路径