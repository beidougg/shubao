# Shubao（书宝）

个人知识管理 AI 助理系统 — 多渠道投喂、智能分析、结构化存储、自然语言检索。

## 项目结构

| 文档 | 说明 |
|------|------|
| [requirements.md](./requirements.md) | 需求规格说明书（业务目标、功能需求、非功能需求） |
| [design.md](./design.md) | 系统设计文档（架构、数据模型、核心流程、技术选型） |
| [tasks.md](./tasks.md) | 任务拆解文档（分阶段任务、依赖关系、验收标准） |

## 技术栈

- **Agent 框架**: OpenClaw
- **AI 模型**: Hermes 4 (OpenRouter API)
- **存储**: Memory Markdown
- **阅读器**: Obsidian
- **主要渠道**: 微信

## 快速开始

### 前置条件

- OpenClaw >= v2026.3.22
- 微信 iOS >= 8.0.70 / Android >= 8.0.69

### 安装

```bash
# 安装微信插件
npx -y @tencent-weixin/openclaw-weixin-cli install
openclaw plugins install "@tencent-weixin/openclaw-weixin"
openclaw config set plugins.entries.openclaw-weixin.enabled true

# 重启 Gateway
openclaw gateway restart

# 扫码登录微信
openclaw channels login --channel openclaw-weixin
```

## 知识库目录

```
~/knowledge-base/
├── 01-技术/          # AI与机器学习、编程语言、系统设计、开发工具
├── 02-产品/          # 产品思维、用户研究、数据分析
├── 03-商业/          # 商业案例、行业趋势
├── 04-个人成长/       # 思维模型、阅读笔记
├── 05-会议与音视频/    # 会议记录、播客笔记
├── 06-孩子教育/       # 学习资源、育儿心得、成长记录
├── _index/           # 标签索引、分类总览、周报存档
└── _config/          # 分类体系配置
```
