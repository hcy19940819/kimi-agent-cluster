# USER.md — 用户档案

> 位置: ~/.openclaw/workspace/USER.md
> 作用: 让 AI 了解你是谁、你的偏好

---

## 基本信息

- **Name**: 开发者
- **Timezone**: Asia/Shanghai (GMT+8)
- **Role**: 独立开发者

## 沟通偏好

- **风格**: 只看逻辑，不看代码细节
- **语言**: 中文优先，技术术语保留英文
- **格式**: 表格 > 列表 > 段落

## 技术栈

- **后端**: Python 3.11, Flask, SQLite
- **前端**: Vanilla JS, HTML
- **环境**: Windows 10/11, 本地开发
- **硬件**: Intel i5-12600KF, 32GB RAM, AMD RX 6750 XT

## 项目信息

- **视频管家**: D:\AI\AI_Tool\Video_management
  - 端口: 5014
  - 数据库: data/video_master.db
  - 版本: v4.1

## 权限边界

### 无需确认（自动执行）
- 读取文件（只读操作）
- 搜索代码
- 运行测试（非破坏性）

### 必须确认（先问你）
- 修改源代码
- 删除文件
- 安装依赖
- 执行数据库操作
- 提交代码到 git

## Don't-ask-first

- 查看 TODO.md
- 查看项目结构
- 搜索代码中的函数定义
- 运行 lint/check（只读检查）

## Always-ask

- 修改超过 3 个文件
- 删除任何文件
- 修改数据库 schema
- 推送到远程仓库
- 安装新依赖包