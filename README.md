<div align="center">

<img src="docs/assets/logo.svg" width="80" alt="FlowSkip Logo" />

# FlowSkip

**分身协同，人类裁决**

> 为每位员工构建具备专业逻辑的 AI 赛博分身，让跨职能协同从"低效会议"升级为"秒级对齐"

[![License](https://img.shields.io/badge/license-Apache%202.0-blue.svg)](LICENSE)
[![Platform](https://img.shields.io/badge/platform-飞书%20Lark-blue)](https://open.feishu.cn)
[![Build](https://img.shields.io/badge/build-passing-brightgreen)]()
[![Version](https://img.shields.io/badge/version-1.0.0--beta-orange)]()
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](CONTRIBUTING.md)

[快速开始](#快速开始) · [产品演示](#产品演示) · [架构设计](#架构设计) · [文档](#文档) · [贡献指南](#贡献指南)

</div>

---

## 目录

- [为什么需要 FlowSkip](#为什么需要-flowskip)
- [核心能力](#核心能力)
- [产品演示](#产品演示)
- [架构设计](#架构设计)
- [快速开始](#快速开始)
- [配置说明](#配置说明)
- [技术栈](#技术栈)
- [路线图](#路线图)
- [贡献指南](#贡献指南)
- [开源协议](#开源协议)

---

## 为什么需要 FlowSkip

一个字段改名，需要开三个会。

这是任何一家规模化公司的日常。随着团队规模扩大，**跨职能协同的沟通成本呈指数级增长**：产品改了一个字段，后端、前端、美术、测试、数据分析都需要知道，每个人的影响评估不同，一来一回反复拉齐，一个"微小"决策可能消耗一整周。

我们认为，这不是人的问题，而是**协同范式**的问题。

**FlowSkip 的答案是：让 AI 分身替你"跑腿对齐"，让真实的你只做最终裁决。**

| 传统协同 | FlowSkip 协同 |
|---------|--------------|
| 发起人手动 @ 所有相关方 | 分身自动识别影响范围并广播 |
| 各方回复散落在不同群 | 分身汇总各方响应，生成结构化报告 |
| 冲突靠人推动解决 | 分身检测冲突，自动 escalation 给决策者 |
| 最终决议靠会议纪要记录 | 决议自动回写飞书文档和代码仓库 |
| 协同周期：天～周 | 协同周期：分钟～小时 |

> "AI 最大的价值，不是帮人干活，而是帮人沟通。"

---

## 核心能力

### 🤖 赛博分身（Digital Twin）

为产品、后端、前端、美术、测试等不同职能构建具备**专业逻辑**的 AI 分身。分身拥有：

- **职能知识**：理解本角色在项目中的职责范围与历史决策
- **工具执行能力**：可操作飞书文档、多维表格、审批、代码仓库
- **授权边界**：只在授权范围内代理行动，超出范围主动升级给本人

### ⚡ A2A 协商协议（Avatar-to-Avatar）

分身之间通过结构化的 A2A 协议通信，无需人类中转：

```
产品分身 ──[变更提案]──▶ 后端分身  ──[影响评估]──▶
                      ▶ 前端分身  ──[影响评估]──▶  产品分身汇总 ──▶ 人类裁决
                      ▶ 测试分身  ──[用例清单]──▶
```

协商过程**透明可观测**：每条 A2A 消息记录在协商日志中，人类可随时介入查看或接管。

### 🧠 带权重的企业知识库

FlowSkip 将飞书中散落的隐性知识显性化，构建带权重的 RAG 检索层：

- **时效权重（Recency）**：近期知识优先，按时间衰减
- **命中权重（Frequency）**：高频被引用的知识常驻热层，降低召回延迟
- **来源**：飞书聊天记录、文档、Wiki、会议纪要，自动 ETL 入库

### 🔔 主动 Escalation 机制

分身不会"擅自做主"。当遇到以下情形，主动推送本人接管：

- 超出授权范围的决策（金额、人事、产品方向）
- 知识库无法覆盖的信息空缺
- 多个分身协商后产生无法自动调和的冲突
- 自身置信度低于安全阈值

Escalation 卡片包含**完整上下文摘要 + 建议选项**，人类一键裁决后结果自动回流。

---

## 产品演示

以下为一个真实的 MVP 演示场景：**后端字段改名，触发全职能分身协商**。

### 触发变更

在飞书中发送消息给 FlowSkip Bot：

```
支付图标版权问题，icon_type 字段需要改名为 new_icon_path
```

### 分身协商过程（全程自动，约 30 秒）

```
[12:00:01] ⚙️ 后端分身       扫描发现 icon_type 共被引用 12 处，涉及支付方式
                              配置表字段、支付渠道返回 DTO、管理端支付图标展
                              示接口和 3 个前端透传字段；需统一改为 new_icon_path
                              并补充兼容旧数据的字段映射。
                              → 已向所有分身发出协商提案 ⚙️🖥️🎨📋🔍

[12:00:04] 🖥️ 前端分身       检查到 PaymentMethodSelector、OrderDetail、
                              CheckoutIcon 等 5 个组件直接读取 icon_type，字段
                              缺失会导致图标区域空白；建议先加兼容层
                              icon_type ?? new_icon_path，再渐进替换。

[12:00:06] 🎨 美术分身       新版支付图标（alipay_v3.svg、wechat_v3.svg 等共
                              8 张）已上传至飞书云文档 /design/payment-icons/v3/，
                              new_icon_path 路径已对齐前端资源目录，可直接引用。

[12:00:08] 📋 产品分身       PRD《支付模块 - UI 规范》第 3.2 节「图标字段说明」
                              需同步更新字段名，草案已在飞书文档起草，待后端确
                              认字段映射关系后即可发布。

[12:00:10] 🔍 测试分身       需重写支付图标渲染测试套件中 4 个用例，并新增
                              new_icon_path 路径有效性校验用例，预计 2 小时完成。

[12:00:11] 🤝 共识达成        各相关方已完成对齐，等待你确认执行方案
```

### 人类裁决

FlowSkip 在飞书推送确认卡片，你只需点击**「确认执行」**：

- 后端提交字段重命名 PR（含向下兼容适配层）
- 前端进入渐进替换流程
- PRD 文档自动更新
- 测试用例重写计划写入飞书任务

**整个流程从发起到执行，传统需要 3 天，FlowSkip 需要 30 秒 + 你的一键确认。**

---

## 架构设计

```
┌─────────────────────────────────────────────────────────────────┐
│                     飞书 Lark 平台                                │
│        IM · 文档 · 日历 · 审批 · 多维表格 · Webhook              │
└───────────────┬──────────────────────────────┬──────────────────┘
                │ 飞书 Open API / CLI            │ 事件推送
                ▼                               ▼
┌──────────────────────────────────────────────────────────────────┐
│                      FlowSkip Core                                │
│                                                                  │
│   ┌──────────────┐   A2A Protocol   ┌──────────────┐            │
│   │  后端分身     │ ◄──────────────► │  前端分身     │            │
│   │  ⚙️ Backend   │                  │  🖥️ Frontend  │            │
│   └──────┬───────┘                  └──────┬───────┘            │
│          │         ┌──────────────┐         │                   │
│          └────────►│  协商引擎     │◄────────┘                   │
│   ┌──────────────┐ │  Negotiation │ ┌──────────────┐            │
│   │  美术分身     │►│  Engine      │◄│  产品分身     │            │
│   │  🎨 Art       │ └──────┬───────┘ │  📋 PM        │            │
│   └──────────────┘        │          └──────────────┘            │
│   ┌──────────────┐        │                                      │
│   │  测试分身     │        │ Escalation                           │
│   │  🔍 QA        │        ▼                                      │
│   └──────────────┘ ┌──────────────┐                             │
│                    │  裁决推送层   │ ──► 飞书消息卡片             │
│                    └──────────────┘                              │
│                                                                  │
│   ┌──────────────────────────────────────────────────────────┐  │
│   │                    企业知识库                              │  │
│   │    向量检索层  +  Recency × Frequency 双权重 RAG           │  │
│   │    聊天记录 ETL · 飞书文档同步 · 决议自动入库               │  │
│   └──────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
                              │
                    ┌─────────▼──────────┐
                    │      人类本体        │
                    │  接收 Escalation    │
                    │  一键裁决           │
                    │  管理授权范围       │
                    └─────────────────────┘
```

### 关键设计原则

1. **人类是裁决者，不是执行者** — 分身完成信息汇聚和方案生成，人类做最终判断
2. **可解释性优先** — 每个分身动作可追溯：依据哪条知识、由哪个分身发起、时间戳完整
3. **保守的授权边界** — 默认宁可多 escalate，也不擅自决策
4. **知识沉淀闭环** — 每次协同产物（决议、冲突、纠正）自动回流知识库，越用越准

---

## 快速开始

### 环境要求

- Node.js >= 20.0
- 飞书企业账号（需开通开放平台权限）
- OpenAI / 兼容 API（用于分身推理）

### 安装

```bash
# 克隆仓库
git clone git@github.com:qwwzdyj/FlowSkip.git
cd FlowSkip

# 安装依赖
npm install

# 复制配置文件
cp .env.example .env.local
```

### 配置飞书应用

1. 前往 [飞书开放平台](https://open.feishu.cn) 创建企业自建应用
2. 开启以下权限：
   - `im:message`（消息读写）
   - `im:message.group_at_msg`（群聊 @ 消息）
   - `docx:document`（文档读写）
   - `approval:approval`（审批）
3. 配置 Webhook 事件回调地址为你的服务地址

### 填写环境变量

```bash
# .env.local

# 飞书应用凭证
FEISHU_APP_ID=cli_xxxxxxxxxx
FEISHU_APP_SECRET=your_app_secret

# AI 推理（支持 OpenAI 兼容接口）
OPENAI_API_KEY=sk-xxxxxxxxxxxx
OPENAI_BASE_URL=https://api.openai.com/v1

# 服务地址（用于飞书回调）
NEXT_PUBLIC_BASE_URL=https://your-domain.com

# 可选：飞书文档 Token（用于协商结果自动写入文档）
DEMO_DOC_TOKEN=your_doc_token
```

### 启动服务

```bash
# 开发模式
npm run dev

# 生产构建
npm run build && npm start
```

访问 `http://localhost:3000/dashboard` 查看实时协商仪表盘。

### 在飞书中使用

向 FlowSkip Bot 发送任意变更请求：

```
icon_type 字段需要改名为 new_icon_path，请评估影响
```

FlowSkip 会自动唤醒相关分身，完成协商并推送确认卡片。

---

## 配置说明

### 分身能力配置

每个分身的职责边界和知识域可在 `config/avatars.yaml` 中配置：

```yaml
avatars:
  backend:
    label: 后端分身
    expertise:
      - 数据库 Schema 变更
      - API 接口设计
      - 数据迁移脚本
    tools:
      - github_pr
      - feishu_doc
    escalation_threshold: 0.7   # 置信度低于此值时主动升级

  frontend:
    label: 前端分身
    expertise:
      - 组件库
      - 页面路由
      - 状态管理
    tools:
      - github_pr
      - feishu_doc
    escalation_threshold: 0.7
```

### 知识库配置

```yaml
knowledge:
  sources:
    - type: feishu_chat    # 飞书群聊记录
      sync_interval: 1h
    - type: feishu_wiki    # 飞书 Wiki 知识空间
      sync_interval: 6h
    - type: github_pr      # GitHub PR 评论和描述
      sync_interval: 30m

  rag:
    recency_decay: 0.95    # 时效衰减系数（每天）
    frequency_boost: 1.2   # 高频命中加权
    top_k: 8               # 每次召回条目数
```

---

## 技术栈

| 层级 | 技术选型 | 说明 |
|------|---------|------|
| **运行时** | Next.js 16 App Router | SSE 实时推送，API Routes |
| **分身推理** | GPT-4o / GPT-5 | OpenAI 兼容接口，支持自定义 baseURL |
| **实时通信** | Server-Sent Events | 仪表盘实时协商动态 |
| **飞书集成** | 飞书 Open API v2 | 消息、文档、审批、多维表格 |
| **知识库** | 向量检索 + 权重层 | 自研双权重 RAG |
| **状态管理** | globalThis 单例 | 跨模块热更新安全的状态共享 |
| **UI** | Tailwind CSS + 毛玻璃设计语言 | Apple Glassmorphism 风格 |

---

## 路线图

### v1.0（当前）
- [x] 多分身 A2A 协商协议
- [x] 飞书 Webhook 集成
- [x] 实时协商仪表盘（SSE）
- [x] Escalation 确认卡片
- [x] 代码变更自动生成（Diff + PR）
- [x] 企业知识库（基础版）

### v1.1（规划中）
- [ ] 分身人格自定义（语气、决策风格）
- [ ] 多项目隔离的知识空间
- [ ] 飞书日历联动（自动排期）
- [ ] 冲突仲裁引擎（多分身僵持时的调解机制）

### v2.0（远期）
- [ ] 私有化部署支持
- [ ] 企业级权限管理（RBAC）
- [ ] 接入 GitHub Actions CI/CD 流水线
- [ ] 移动端 Escalation 快速裁决

---

## 贡献指南

欢迎任何形式的贡献！请阅读 [CONTRIBUTING.md](CONTRIBUTING.md) 了解完整流程。

### 本地开发

```bash
# Fork 并克隆
git clone git@github.com:your-username/FlowSkip.git

# 创建功能分支
git checkout -b feat/your-feature

# 提交并推送
git commit -m "feat: add your feature"
git push origin feat/your-feature

# 发起 Pull Request
```

### 代码规范

- 使用 TypeScript，开启严格模式
- 提交信息遵循 [Conventional Commits](https://www.conventionalcommits.org/)
- 核心逻辑变更需附带测试用例

### 提交 Issue

- **Bug 报告**：使用 Bug Report 模板，附上复现步骤和日志
- **功能建议**：使用 Feature Request 模板，描述场景和期望行为

---

## 开源协议

本项目基于 [Apache License 2.0](LICENSE) 开源。

---

<div align="center">

**FlowSkip** · 分身协同，人类裁决

[GitHub](https://github.com/qwwzdyj/FlowSkip) · [Issues](https://github.com/qwwzdyj/FlowSkip/issues) · [Discussions](https://github.com/qwwzdyj/FlowSkip/discussions)

</div>
