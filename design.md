# Shubao（书宝）— 系统设计文档

## 1. 技术选型

| 组件 | 选型 | 选型理由 |
|------|------|---------|
| Agent 框架 | OpenClaw | 多渠道接入、Memory 持久化、Cron 定时、Skills 扩展、原生微信支持 |
| AI 推理模型 | Hermes 4 (OpenRouter API) | 强推理能力、混合推理模式、原生 Tool Calling、低拒绝率 |
| 知识本体存储 | Memory Markdown | 人可读、Git 版本控制、天然兼容 Obsidian、零门槛 |
| 知识阅读器 | Obsidian | 本地优先、双向链接、图谱可视化、1000+ 插件生态 |
| 语义索引（后期） | ChromaDB / Qdrant | 语义相似度搜索、知识关联发现 |
| Embedding 模型（后期） | BGE / text2vec | 中文友好、本地部署 |

---

## 2. 系统架构

### 2.1 四层架构总览

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        输入层 (Input Layer)                                │
│  微信 │ Telegram │ WebChat │ Email │ REST API │ OneNote 导入              │
│   (文字 / 链接 / 图片 / 音视频 / 文件 — 多渠道投喂 + 存量导入)               │
└───────────────────────────┬──────────────────────────────────────────────┘
                            ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                     处理管道 (Processing Pipeline)                         │
│  ┌──────────┐ ┌──────────┐ ┌───────────┐ ┌──────────────┐ ┌───────────┐  │
│  │ URL 抓取  │ │ 音频转写  │ │ 图片 OCR   │ │ 视频关键帧    │ │ OneNote   │  │
│  │(Browser) │ │(Whisper) │ │(Vision)   │ │(提取+描述)   │ │ 转换器    │  │
│  └──────────┘ └──────────┘ └───────────┘ └──────────────┘ └───────────┘  │
└───────────────────────────┬──────────────────────────────────────────────┘
                            ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                    AI 分析层 (Analysis Layer)                              │
│                     Hermes 4 (OpenRouter API)                             │
│  ┌──────────┐ ┌──────────┐ ┌───────────┐ ┌──────────────┐                │
│  │ 智能摘要  │ │ 自动分类  │ │ 实体提取   │ │ 标签生成      │                │
│  └──────────┘ └──────────┘ └───────────┘ └──────────────┘                │
│  ┌──────────┐ ┌──────────┐ ┌───────────┐                                 │
│  │ 关联发现  │ │ 知识盲区  │ │ 定期归纳   │                                 │
│  └──────────┘ └──────────┘ └───────────┘                                 │
└───────────────────────────┬──────────────────────────────────────────────┘
                            ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                     存储层 (Storage Layer)                                 │
│  ┌──────────────────────┐   ┌─────────────────────────────────────────┐  │
│  │   Memory Markdown     │   │  向量数据库 (Phase 3-4 引入)             │  │
│  │   (知识本体)           │   │  ChromaDB / Qdrant                      │  │
│  │   按分类目录组织        │   │  语义搜索 + 知识关联                     │  │
│  │   git 版本控制         │   │                                         │  │
│  │   对接 Obsidian 阅读   │   │                                         │  │
│  └──────────────────────┘   └─────────────────────────────────────────┘  │
└───────────────────────────┬──────────────────────────────────────────────┘
                            ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                      输出层 (Output Layer)                                 │
│  ┌──────────┐ ┌──────────┐ ┌───────────┐ ┌──────────────┐                │
│  │ 问答检索  │ │ 知识回顾  │ │ 定时周报   │ │ 关联图谱     │                │
│  │ (微信内)  │ │          │ │ (推送到微信)│ │ (Obsidian)  │                │
│  └──────────┘ └──────────┘ └───────────┘ └──────────────┘                │
└──────────────────────────────────────────────────────────────────────────┘
```

### 2.2 检索架构：混合模式

```
                    用户查询
                       │
                       ▼
              ┌──────────────────┐
              │    查询意图判断    │
              └──────┬───────────┘
                     │
         ┌───────────┴───────────┐
         ▼                       ▼
  ┌──────────────┐        ┌──────────────┐
  │ 精确查找路径   │        │ 语义搜索路径   │
  │ (文件名/目录)  │        │ (向量相似度)   │
  └──────┬───────┘        └──────┬───────┘
         │                       │
         ▼                       ▼
  ┌──────────────┐        ┌──────────────┐
  │ Memory       │        │ 向量数据库     │
  │ Markdown     │        │               │
  │ (返回完整原文) │        │ (返回 top-k    │
  │              │        │  文档 ID)     │
  └──────────────┘        └──────┬───────┘
                                 │
                                 ▼
                        ┌──────────────┐
                        │ 反查 Markdown │
                        │ (获取完整原文)  │
                        └──────────────┘
```

---

## 3. 数据模型

### 3.1 知识条目 (Knowledge Entry)

每条知识以 Markdown 文件存储，包含 YAML Frontmatter 元数据：

```yaml
---
id: "2026-05-12-001"           # 唯一标识（日期+序号）
title: "Transformer 架构详解"   # 标题
source: "https://arxiv.org/abs/1706.03762"  # 来源 URL
source_type: "paper"            # 来源类型: article|paper|video|audio|image|note|onenote
created_at: "2026-05-12T10:30:00+08:00"
updated_at: "2026-05-12T10:30:00+08:00"
category: "AI/深度学习"         # 分类路径
tags: ["transformer", "attention", "nlp", "论文精读"]
entities: ["Google", "Vaswani", "Self-Attention"]
summary: "Transformer 是一种基于自注意力机制..."  # 200字摘要
status: "active"                # active|archived|merged
---
```

### 3.2 知识库目录结构

```
~/knowledge-base/
├── 01-技术/
│   ├── AI与机器学习/
│   ├── 编程语言/
│   ├── 系统设计/
│   └── 开发工具/
├── 02-产品/
│   ├── 产品思维/
│   ├── 用户研究/
│   └── 数据分析/
├── 03-商业/
│   ├── 商业案例/
│   └── 行业趋势/
├── 04-个人成长/
│   ├── 思维模型/
│   └── 阅读笔记/
├── 05-会议与音视频/
│   ├── 会议记录/
│   └── 播客笔记/
├── 06-孩子教育/
│   ├── 学习资源/
│   ├── 育儿心得/
│   └── 成长记录/
├── _index/                      # 自动生成
│   ├── tag-index.md             # 标签索引
│   ├── category-overview.md     # 分类总览
│   └── weekly-digest/           # 周报存档
└── _config/
    └── categories.yaml          # 分类体系定义
```

### 3.3 分类体系 (categories.yaml)

```yaml
categories:
  - name: "技术"
    slug: "tech"
    subcategories:
      - name: "AI与机器学习"
        keywords: ["ai", "machine learning", "deep learning", "llm", "transformer"]
      - name: "编程语言"
        keywords: ["python", "rust", "go", "typescript", "java"]
      - name: "系统设计"
        keywords: ["architecture", "distributed", "microservices", "scalability"]
      - name: "开发工具"
        keywords: ["git", "docker", "kubernetes", "vscode", "terminal"]

  - name: "产品"
    slug: "product"
    subcategories:
      - name: "产品思维"
      - name: "用户研究"
      - name: "数据分析"

  - name: "商业"
    slug: "business"
    subcategories:
      - name: "商业案例"
      - name: "行业趋势"

  - name: "个人成长"
    slug: "growth"
    subcategories:
      - name: "思维模型"
      - name: "阅读笔记"

  - name: "会议与音视频"
    slug: "meetings"
    subcategories:
      - name: "会议记录"
      - name: "播客笔记"

  - name: "孩子教育"
    slug: "education"
    subcategories:
      - name: "学习资源"
        keywords: ["教材", "课程", "绘本", "科普", "练习"]
      - name: "育儿心得"
        keywords: ["育儿", "亲子", "心理", "习惯", "健康"]
      - name: "成长记录"
        keywords: ["里程碑", "作品", "日记", "照片"]
```

---

## 4. 核心流程

### 4.1 知识录入流程

```
用户投喂内容（微信/Telegram/API）
    │
    ▼
┌──────────────┐
│  类型识别     │  (URL? 图片? 文字? 音频?)
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  预处理       │  (抓取网页/转写音频/OCR 图片)
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  Hermes 4     │
│  分析          │
│              │
│  - 摘要       │
│  - 分类       │
│  - 标签       │
│  - 实体       │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  生成 Markdown│  (带 YAML Frontmatter，兼容 Obsidian [[双链]])
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  写入知识库    │  (按分类放入对应目录)
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  (Phase 3+)   │
│  向量化索引    │  (写入 ChromaDB)
└──────────────┘
```

### 4.2 OneNote 存量导入流程

```
触发导入（手动或定时）
    │
    ▼
┌──────────────────────┐
│  MS Graph API 认证     │  (OAuth 2.0 授权)
└──────┬───────────────┘
       │
       ▼
┌──────────────────────┐
│  枚举笔记本/分区/页面   │  GET /me/onenote/notebooks
└──────┬───────────────┘
       │
       ▼
┌──────────────────────┐
│  逐页拉取 HTML 内容    │  GET /me/onenote/pages/{id}/content
│  提取图片附件         │
└──────┬───────────────┘
       │
       ▼
┌──────────────────────┐
│  HTML → Markdown     │  转换器处理
│  图片 → 本地存储      │
└──────┬───────────────┘
       │
       ▼
┌──────────────────────┐
│  进入标准知识录入流程   │  → Hermes 4 分析 → 归类 → 入库
└──────────────────────┘
```

### 4.3 检索问答流程

```
用户提问（微信内直接对话）
    │
    ▼
┌──────────────────┐
│  查询意图判断      │
│  - 精确查找？      │
│  - 模糊语义搜索？   │
└──────┬───────────┘
       │
       ├── 精确查找 ──▶ grep/文件名匹配 ──▶ 返回完整 Markdown
       │
       └── 语义搜索 ──▶ 查询向量化 ──▶ ChromaDB 搜索 top-k
                              │
                              ▼
                       ┌──────────────┐
                       │  获取 Markdown │
                       │  原文片段      │
                       └──────┬───────┘
                              │
                              ▼
                       ┌──────────────┐
                       │  Hermes 4     │
                       │  生成回答     │
                       │  (结合检索    │
                       │   结果+提问)  │
                       └──────┬───────┘
                              │
                              ▼
                       ┌──────────────┐
                       │  返回（微信内）： │
                       │  - 答案       │
                       │  - 引用来源    │
                       │  - 相关笔记链接 │
                       └──────────────┘
```

### 4.4 定期归纳流程

```
Cron 触发 (每日/每周/每月)
    │
    ▼
┌──────────────────┐
│  Hermes 4         │
│  Thinking 模式    │  (激活深度推理链)
└──────┬───────────┘
       │
       ▼
┌────────────────────────┐
│  读取指定时间段的新增知识  │
│  (从 Markdown 或向量库)  │
└────────┬───────────────┘
         │
         ▼
┌────────────────────────┐
│  分析任务：              │
│                         │
│  每日:                   │
│  - 今天新增了什么        │
│  - 最重要的 3 个收获    │
│                         │
│  每周:                   │
│  - 知识间的交叉关联      │
│  - 主题演化趋势         │
│  - 生成知识周报         │
│                         │
│  每月:                   │
│  - 合并相似主题笔记      │
│  - 优化分类体系         │
│  - 生成知识图谱         │
│  - 清理低价值内容       │
└────────┬───────────────┘
         │
         ▼
┌────────────────────────┐
│  输出归纳结果            │
│  - 写入 _index/ 目录     │
│  - 推送到微信给用户      │
└────────────────────────┘
```

---

## 5. 模块划分

### 5.1 模块依赖关系

```
┌─────────────┐
│   Channel    │  微信插件 / Telegram / REST API
│   (渠道层)    │
└──────┬──────┘
       ▼
┌─────────────┐
│   Router     │  消息路由：识别类型，分发到对应 Processor
│   (路由层)    │
└──────┬──────┘
       ▼
┌─────────────────────────────────────────────┐
│              Preprocessor (预处理)            │
│  ┌──────────┐ ┌──────────┐ ┌───────────┐    │
│  │URLFetcher│ │AudioTrans│ │ImageOCR   │    │
│  │          │ │(Whisper) │ │(Vision)   │    │
│  └──────────┘ └──────────┘ └───────────┘    │
│  ┌──────────┐ ┌──────────┐ ┌───────────┐    │
│  │FileParser│ │VideoProc │ │OneNote    │    │
│  │(PDF/DOC) │ │          │ │Loader     │    │
│  └──────────┘ └──────────┘ └───────────┘    │
└──────────────────────┬──────────────────────┘
                       ▼
┌─────────────────────────────────────────────┐
│             AI Analyzer (分析引擎)             │
│  ┌──────────┐ ┌──────────┐ ┌───────────┐    │
│  │Summarizer│ │Classifier│ │Tagger     │    │
│  └──────────┘ └──────────┘ └───────────┘    │
│  ┌──────────┐ ┌──────────┐ ┌───────────┐    │
│  │EntityExtr│ │LinkFinder│ │DigestGen  │    │
│  └──────────┘ └──────────┘ └───────────┘    │
└──────────────────────┬──────────────────────┘
                       ▼
┌─────────────────────────────────────────────┐
│               Storage (存储层)                │
│  ┌──────────────────┐ ┌──────────────────┐  │
│  │ Memory Markdown   │ │ ChromaDB (后期)   │  │
│  │ ~/knowledge-base/ │ │ Vector Index     │  │
│  └──────────────────┘ └──────────────────┘  │
└──────────────────────┬──────────────────────┘
                       ▼
┌─────────────────────────────────────────────┐
│                Output (输出层)                │
│  ┌──────────┐ ┌──────────┐ ┌───────────┐    │
│  │QAReply   │ │ReviewGen │ │WeeklyRpt  │    │
│  │          │ │          │ │           │    │
│  └──────────┘ └──────────┘ └───────────┘    │
└─────────────────────────────────────────────┘
```

### 5.2 模块职责

| 模块 | 职责 | 对应 Skill |
|------|------|-----------|
| URL Fetcher | 抓取网页全文并清洗正文 | `url-fetcher` |
| Audio Transcriber | Whisper 音频转写 | `audio-transcribe` |
| Image OCR | OCR 提取 + Vision 描述 | `image-analyzer` |
| File Parser | 解析 PDF/Office/Markdown | `file-parser` |
| Video Processor | 提取关键帧 + 字幕/转写 | `video-processor` |
| OneNote Loader | MS Graph API → Markdown | `onenote-loader` |
| Summarizer | 生成 200 字以内摘要 | AI Prompt |
| Classifier | 自动归类到分类体系 | AI Prompt |
| Tagger | 提取 5-10 个标签 | AI Prompt |
| Entity Extractor | 识别人名/公司/概念 | AI Prompt |
| Link Finder | 发现知识关联 | AI Prompt |
| Digest Generator | 日/周/月报生成 | AI Prompt |
| QA Reply | 问答对话生成 | AI Prompt + RAG |

---

## 6. 存储策略

### 6.1 Memory Markdown vs 向量数据库

| 维度 | Memory Markdown | 向量数据库 |
|------|:---------------:|:----------:|
| 检索方式 | 关键词匹配 / 文件名 | 语义相似度搜索 |
| 同义词理解 | ❌ | ✅ |
| 跨语言检索 | ❌ | ✅ |
| 返回完整性 | ✅ 完整原文 | ⚠️ 片段 |
| 可读性 | ✅ 人可读 | ❌ 向量不可读 |
| 版本控制 | ✅ Git 支持 | ❌ |
| 存储开销 | 小 | 大（双份） |
| 大规模检索 | 慢（线性） | 快（ANN） |
| 关联发现 | ❌ 靠人工 | ✅ 聚类分析 |

### 6.2 演进策略

| 阶段 | 存储方案 | 触发条件 |
|------|---------|---------|
| Phase 1-2 | 纯 Memory Markdown | 系统起步，内容量 < 100 条 |
| Phase 3-4 | Markdown + ChromaDB 混合 | 内容量 > 100 条，语义检索需求显现 |
| Phase 5 | 混合架构持续优化 | 完善检索路由，优化关联推荐 |

> 核心原则：Markdown 是「知识本体」，向量库是「智能索引层」。两者互补，不替代。

---

## 7. AI 模型部署

### 7.1 Hermes 4（OpenRouter API）

| 方案 | 模式 | 适用场景 |
|------|------|---------|
| A. API 直连 | OpenRouter API | 快速启动，无需 GPU |
| B. 本地部署 | Ollama + Hermes 4 14B/70B | 数据隐私优先，需 GPU（后期可选） |
| C. 混合模式 | 简单任务本地 14B + 复杂推理 API 70B | 成本最优（后期可选） |

> Phase 1 采用方案 A（OpenRouter），月成本约 $5-20，零运维负担。

### 7.2 Prompt 模板

系统维护以下标准化 Prompt 模板：

| 模板文件 | 用途 |
|---------|------|
| `summarize.md` | 智能摘要 Prompt |
| `classify.md` | 自动分类 Prompt |
| `tagging.md` | 标签提取 Prompt |
| `entity.md` | 实体识别 Prompt |
| `qa.md` | 问答检索 Prompt |
| `digest.md` | 定期归纳 Prompt |

---

## 8. 渠道配置

### 8.1 微信渠道

OpenClaw 通过腾讯官方插件 `@tencent-weixin/openclaw-weixin` 接入微信，基于 iLink 智联协议。

**前置条件：** OpenClaw >= v2026.3.22，微信 iOS >= 8.0.70 / Android >= 8.0.69

**安装：**

```bash
npx -y @tencent-weixin/openclaw-weixin-cli install
openclaw plugins install "@tencent-weixin/openclaw-weixin"
openclaw config set plugins.entries.openclaw-weixin.enabled true
openclaw gateway restart
openclaw channels login --channel openclaw-weixin
```

**配对授权：**

```bash
openclaw pairing list openclaw-weixin
openclaw pairing approve openclaw-weixin <CODE>
```

支持消息类型：文字、图片、视频、文件、语音。

### 8.2 Obsidian 配置

**基础配置：** 将 `~/knowledge-base/` 作为 Obsidian Vault 打开。

**OpenClaw 端配置（`~/.openclaw/openclaw.json`）：**

```json
{
  "agents": {
    "defaults": {
      "workspace": "~/knowledge-base",
      "memorySearch": {
        "additionalPaths": [
          "~/knowledge-base/01-技术",
          "~/knowledge-base/02-产品",
          "~/knowledge-base/03-商业",
          "~/knowledge-base/04-个人成长",
          "~/knowledge-base/05-会议与音视频",
          "~/knowledge-base/06-孩子教育"
        ]
      }
    }
  }
}
```

**Obsidian 插件：** [obsidian-openclaw](https://github.com/AndyBold/obsidian-openclaw) 实现双向同步。

**推荐插件组合：** Dataview、Graph View、Templater、Calendar、Excalidraw。

### 8.3 双链语法兼容

系统生成的 Markdown 笔记使用 Obsidian 兼容的 `[[双链]]` 语法：

```markdown
## 与我已有知识的关系
- 与 [[RNN vs LSTM 对比]] 相关
- 是 [[BERT 论文笔记]] 的前置知识
- 参考了 [[2026-03-15-LLM训练范式演进]]
```

---

## 9. OneNote 集成设计

### 9.1 API 调用链

```
GET /me/onenote/notebooks                     → 获取所有笔记本
GET /me/onenote/notebooks/{id}/sections       → 获取分区
GET /me/onenote/sections/{id}/pages           → 获取页面列表
GET /me/onenote/pages/{id}/content            → 获取页面 HTML（含图片）
GET /me/onenote/resources/{id}/$value         → 下载图片资源
```

### 9.2 认证

1. Azure Portal 注册应用 → 授权 Notes.Read 权限
2. 获取 refresh token（首次 OAuth 交互授权）
3. Token 存储到环境变量 `ONENOTE_REFRESH_TOKEN`

### 9.3 导入模式

| 模式 | 描述 | 适用场景 |
|------|------|---------|
| 全量导入 | 一次性拉取所有 OneNote 内容 | 首次使用，历史迁移 |
| 增量同步 | 按 `lastModifiedTime` 只拉变更 | 持续使用 OneNote |
| 指定分区 | 只导入特定笔记本/分区 | 部分导入 |

### 9.4 注意事项

- OneNote API 返回 HTML，需转换为 Markdown
- 图片通过独立资源端点下载
- 建议 Phase 4 实施，前期以微信投喂为主

---

## 10. 安全设计

- 敏感内容不入知识库的过滤机制
- API Key 通过环境变量管理，不写入配置文件
- 微信渠道通过配对码授权
- 知识库文件仅本地可读
- OneNote API Token 仅请求 Notes.Read 只读权限
