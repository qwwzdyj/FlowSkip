# MVP 骨架

> 第二阶段产出物 · v0.1
> 更新日期：2026-04-28
> 关联文档：[决策记录.md](./决策记录.md)、[技术选型.md](./技术选型.md)、[A2A协议草案.md](./A2A协议草案.md)

---

## 一、文档目的与边界

本文档把前三份文档的决策落到**可直接开工的代码骨架**——目录布局、模块职责、关键接口、数据库 schema、配置文件、端到端调用链、第三阶段 WBS。

读完后开发同学应能：
- 一次 `uv sync` 就有可运行的项目骨架
- 知道每个模块该写什么、不该写什么
- 知道"改字段"场景在代码层会经过哪些文件

**本文档不覆盖**：
- 业务侧的 prompt 调优、加权 RAG 参数 tune（属于实装时的实验）
- 飞书应用注册的具体步骤（实操，非规划）
- 性能优化、监控告警（v2 范畴）

---

## 二、项目目录结构

```
FlowSkip/
├── docs/                          # 第二阶段所有规划文档
│   ├── 决策记录.md
│   ├── 技术选型.md
│   ├── A2A协议草案.md
│   └── MVP骨架.md
├── 项目概要.md
├── pyproject.toml                 # uv / 依赖
├── uv.lock
├── docker-compose.yml             # postgres + app
├── Dockerfile
├── alembic.ini
├── .env.example
├── README.md
├── alembic/
│   ├── env.py
│   └── versions/
├── config/
│   ├── twins.yaml                 # 分身能力静态注册（D-015）
│   ├── policies.yaml              # 各分身的授权范围
│   └── prompts/
│       ├── twin_system.md         # 分身基础 system prompt
│       ├── compact.md             # 聊天 compact 模板
│       ├── escalation.md          # escalation 摘要模板
│       └── intent_classify.md     # 意图分类模板
├── src/
│   └── flowskip/
│       ├── __init__.py
│       ├── main.py                # FastAPI app entry + 启动编排
│       ├── settings.py            # pydantic-settings
│       │
│       ├── lark/                  # 接入层
│       │   ├── adapter.py         # LarkAdapter：消息收发主类
│       │   ├── events.py          # WebSocket 事件订阅
│       │   ├── cards.py           # 消息卡片构造（escalation 用）
│       │   └── client.py          # lark-oapi 封装
│       │
│       ├── twins/                 # 分身层
│       │   ├── registry.py        # 加载 twins.yaml，索引 twin_id ↔ owner ↔ capabilities
│       │   ├── twin.py            # Twin 类：包装一个 Claude Agent 实例
│       │   ├── policy.py          # 授权范围检查
│       │   └── prompts.py         # prompt 加载与渲染
│       │
│       ├── a2a/                   # A2A 协议层
│       │   ├── messages.py        # pydantic 消息模型（envelope + 各 payload）
│       │   ├── bus.py             # MessageBus 接口 + InProcessBus 实现
│       │   ├── conversation.py    # Conversation 状态机
│       │   └── timeouts.py        # 超时检测器（30s/5min/15s）
│       │
│       ├── knowledge/             # 知识层
│       │   ├── embedding.py       # BGE 加载与 embed
│       │   ├── rag.py             # RAGService：加权召回
│       │   ├── compact.py         # 聊天 compact（Claude Haiku）
│       │   ├── etl.py             # ETL pipeline（飞书拉取 → compact → 入库）
│       │   └── weight.py          # weight 计算与衰减更新
│       │
│       ├── escalation/            # 升级层
│       │   ├── service.py         # EscalationService
│       │   └── card_builder.py    # 飞书 escalation 卡片构造
│       │
│       ├── tools/                 # Agent 工具集（注入 Claude Agent）
│       │   ├── base.py            # Tool 基类
│       │   ├── lark_tools.py      # lark.send_message / lark.read_chat / lark.write_doc
│       │   ├── rag_tools.py       # rag.search / rag.add_knowledge
│       │   ├── a2a_tools.py       # a2a.request / a2a.respond / a2a.escalate / a2a.propose
│       │   └── escalate_tools.py  # 高层 escalation 封装
│       │
│       ├── storage/               # 存储层
│       │   ├── db.py              # SQLAlchemy async engine
│       │   ├── models.py          # ORM 模型
│       │   └── repositories.py    # 仓储模式封装常用查询
│       │
│       ├── scheduler/             # 调度
│       │   ├── jobs.py            # APScheduler 注册中心
│       │   └── tasks/
│       │       ├── etl_job.py     # 定时 ETL（小时级）
│       │       ├── compact_job.py # 定时 compact（小时级）
│       │       └── weight_decay_job.py  # 定时 weight 衰减（天级）
│       │
│       ├── observability/
│       │   ├── logging.py         # structlog 配置
│       │   └── tracing.py         # trace_id 上下文管理
│       │
│       └── api/                   # 管理 HTTP API（FastAPI 路由）
│           ├── twins.py           # GET /twins, POST /twins/:id/policy
│           ├── conversations.py   # GET /conversations, GET /conversations/:id
│           └── auth.py            # POST /auth/private-chat (opt-in 开关)
│
└── tests/
    ├── conftest.py                # PG fixture, MessageBus fixture
    ├── test_a2a/
    │   ├── test_bus.py
    │   ├── test_messages.py
    │   ├── test_conversation_fsm.py
    │   └── test_timeouts.py
    ├── test_rag/
    │   ├── test_embedding.py
    │   └── test_weighted_recall.py
    ├── test_twins/
    │   └── test_twin_dispatch.py
    └── e2e/
        └── test_field_change_scenario.py    # 端到端"改字段"场景
```

> 采用 `src/` layout：避免 import 路径与项目根目录混淆，pytest 可显式指向 `src/flowskip`。

---

## 三、各模块职责（按依赖顺序）

### 3.1 `storage/` — 数据库与 ORM

- `db.py`：建 async engine + sessionmaker；注入 FastAPI 依赖
- `models.py`：所有 ORM 表（见 §五 schema）
- `repositories.py`：封装高频查询（如 `KnowledgeRepo.weighted_search()`、`A2AMessageRepo.by_conversation()`）

**约束**：模块内禁止引用 `lark/` 或 `twins/`——存储层是最底层，被所有上层依赖。

### 3.2 `a2a/` — 协议层

- `messages.py`：pydantic envelope + 各 type 的 payload model（见 [A2A协议草案.md §11.2](./A2A协议草案.md)）
- `bus.py`：定义 `MessageBus` ABC + `InProcessBus` 实现（asyncio queue + capability 路由）
- `conversation.py`：`Conversation` 类，维护 FSM；持久化到 `conversations` 表
- `timeouts.py`：基于 asyncio 的延迟任务，超时后投递特殊事件给 conversation

**关键接口**：

```python
# a2a/bus.py
class MessageBus(ABC):
    @abstractmethod
    async def send(self, message: A2AMessage) -> None: ...

    @abstractmethod
    async def subscribe(self, twin_id: str, handler: Callable[[A2AMessage], Awaitable[None]]) -> None: ...

    @abstractmethod
    async def shutdown(self) -> None: ...


class InProcessBus(MessageBus):
    """asyncio 内存实现；启动时从 twins.yaml 构建 capability 路由表。"""
    def __init__(self, capabilities: dict[str, list[Capability]], persister: A2AMessageRepo): ...
```

### 3.3 `knowledge/` — RAG 与知识 pipeline

- `embedding.py`：BGE 模型加载（启动时 warm-up），提供 `embed(text) -> vector`
- `rag.py`：`RAGService.search(query, k, filters)`，内部走加权 SQL（[决策记录 D-012](./决策记录.md)）
- `compact.py`：用 Claude Haiku 把聊天块抽成 `{facts, decisions, todos, people, topic}`
- `etl.py`：从飞书拉群消息 → 分块 → compact → 入库
- `weight.py`：写入时初始化 weight、命中时更新 hit_count、定时衰减 last_hit_at

**关键接口**：

```python
# knowledge/rag.py
@dataclass
class RecallResult:
    item_id: UUID
    content: str
    score: float          # 最终加权 score
    cosine: float
    weight: float
    knowledge_refs: list[str]

class RAGService:
    async def search(
        self,
        query: str,
        k: int = 5,
        filters: dict | None = None,
    ) -> list[RecallResult]: ...

    async def add(self, content: str, source: str, metadata: dict) -> UUID: ...

    async def hit(self, item_ids: list[UUID]) -> None:
        """命中时更新 frequency；由 search 之后业务侧调用，避免 search 本身有副作用。"""
```

### 3.4 `twins/` — 分身

- `registry.py`：启动加载 `config/twins.yaml`，提供 `get(twin_id) / get_by_owner(user_id) / find_by_capability(intent, domains)`
- `twin.py`：核心 `Twin` 类
- `policy.py`：基于 `policies.yaml` 判断"是否需要 escalate"（D-018 / D-019 阈值在此触发）
- `prompts.py`：渲染 system prompt（拼入 owner 信息、capabilities、policy 摘要）

**关键接口**：

```python
# twins/twin.py
class Twin:
    twin_id: str
    owner_user_id: str
    capabilities: list[Capability]
    policy: Policy

    def __init__(
        self,
        config: TwinConfig,
        agent: ClaudeAgent,           # 包装 claude-agent-sdk
        bus: MessageBus,
        rag: RAGService,
        lark: LarkAdapter,
    ): ...

    async def receive_lark(self, msg: LarkMessage) -> None:
        """飞书消息进来，分身决策 → 可能触发 A2A 或直接回复。"""

    async def receive_a2a(self, msg: A2AMessage) -> None:
        """A2A 消息进来，按 type dispatch。"""

    # 内部：以下都是给 Agent 的 tool（见 tools/）
```

### 3.5 `tools/` — Agent 工具集

每个 `Tool` 是 Claude Agent SDK 注册的 function。

| 工具 | 签名 | 用途 |
|---|---|---|
| `lark.send_message` | `(chat_id, text, card?)` | 发飞书消息 |
| `lark.read_chat` | `(chat_id, since)` | 读群历史（仅授权群） |
| `lark.write_doc` | `(doc_id, content)` | 写文档 |
| `lark.create_bitable_record` | `(app_token, table_id, fields)` | 写多维表格 |
| `rag.search` | `(query, k, filters)` | 加权召回 |
| `rag.add_knowledge` | `(content, source, metadata)` | 入库（如 escalation 决议） |
| `a2a.request` | `(intent, to, payload)` | 发起 request |
| `a2a.respond` | `(parent_id, payload, confidence)` | 回复 |
| `a2a.propose` | `(to, payload)` | 主动提议 |
| `a2a.escalate` | `(reason, summary, options)` | 升级到人类 |
| `a2a.cancel` | `(conversation_id, reason)` | 撤回 |

每个 tool 内部强制：
- 写入 `audit_logs`（呼应原则 §八.2 可解释性）
- trace_id 透传

### 3.6 `lark/` — 飞书接入

- `client.py`：`lark-oapi` 客户端单例
- `events.py`：WS 长连接订阅 `im.message.receive_v1`、`card.action.trigger`
- `adapter.py`：`LarkAdapter`，统一收发入口
- `cards.py`：构造 escalation 卡片（一键选项 + footer 标识"分身代发"）

**关键接口**：

```python
# lark/adapter.py
class LarkAdapter:
    async def start(self) -> None: ...   # 启动 WS

    async def send_message(self, chat_id: str, text: str) -> str: ...

    async def send_card(self, chat_id: str, card: dict) -> str: ...

    def on_message(self, handler: Callable[[LarkMessage], Awaitable[None]]) -> None: ...

    def on_card_action(self, handler: Callable[[LarkCardAction], Awaitable[None]]) -> None: ...
```

`LarkAdapter` 不知道分身——它只知道"消息进来，叫谁处理"。具体路由由 `main.py` 把 `TwinRegistry.dispatch_lark` 注册成 handler。

### 3.7 `escalation/` — 升级层

- `service.py`：监听 `MessageBus` 上 `type=escalate` 的消息，转译为飞书卡片；监听飞书卡片回调，把人类决议转回 A2A `response`（`from.twin_id=null`）
- `card_builder.py`：把 `EscalatePayload.options` 渲染为 interactive card

### 3.8 `scheduler/` — 后台任务

| Job | 频率 | 任务 |
|---|---|---|
| `etl_job` | 每小时 | 拉取上一小时的群聊新消息（仅 `chat_sources` 中授权的） |
| `compact_job` | 每小时（错开 ETL） | 对未 compact 的会话块跑 Haiku，入库 |
| `weight_decay_job` | 每天 | 触发 `weight.py` 重算 recency 因子（也可以 lazy 在查询时算） |

> 实装时 weight 可纯计算列（不存物化值），decay job 就只是采样监控。

### 3.9 `observability/`

- `logging.py`：structlog JSON 输出，每条带 `trace_id` / `conversation_id` / `twin_id`
- `tracing.py`：基于 `contextvars` 的 trace 上下文（无需 OpenTelemetry）

---

## 四、端到端调用链（改字段场景）

呼应 [A2A协议草案.md §十二](./A2A协议草案.md)，标注每步落在哪个文件：

```
1. Alice 在飞书私聊产品分身："改用户等级 1-5 → 1-10"
   └─ lark/events.py 收到 im.message.receive_v1
       └─ lark/adapter.py:LarkAdapter.on_message 触发
            └─ main.py 注册的 TwinRegistry.dispatch_lark
                 └─ twins/twin.py:Twin.receive_lark()
                      └─ Claude Agent 推理（带 system prompt + 工具集）

2. 分身决定调用 a2a.request 广播
   └─ tools/a2a_tools.py:a2a_request()
        └─ a2a/messages.py 构造 envelope
        └─ a2a/bus.py:InProcessBus.send()
             ├─ storage/repositories.py:A2AMessageRepo.persist()  # 同步写表
             └─ 路由到 twin-dev-bob, twin-design-carol

3. 各分身收到 A2A request
   └─ a2a/bus.py 投递到订阅 handler
        └─ twins/twin.py:Twin.receive_a2a() → dispatch by type
             └─ Claude Agent 推理 → 调用 rag.search
                  └─ tools/rag_tools.py:rag_search()
                       └─ knowledge/rag.py:RAGService.search()  # 加权 SQL
             └─ Agent 调用 a2a.respond
                  └─ tools/a2a_tools.py:a2a_respond()

4. 产品分身收齐 response
   └─ twins/twin.py:Twin.receive_a2a() 第 2、3 次
        └─ Agent 决定 a2a.escalate（涉及迁移成本，超 policy）
             └─ tools/a2a_tools.py:a2a_escalate()
                  └─ a2a/bus.py 路由 type=escalate 消息
                       └─ escalation/service.py 监听到

5. EscalationService 转译为飞书卡片
   └─ escalation/card_builder.py 构造 interactive card
        └─ lark/adapter.py:LarkAdapter.send_card()

6. Alice 点击 "确认启动迁移"
   └─ lark/events.py 收到 card.action.trigger
        └─ lark/adapter.py:LarkAdapter.on_card_action
             └─ escalation/service.py 把人类决议转为 A2A response
                  └─ a2a/bus.py 投递（from.twin_id = null）

7. 产品分身收到人类回流，conversation 转 resolved_by_human
   └─ Agent 调用 lark.write_doc 把决议写入飞书文档
        └─ tools/lark_tools.py:lark_write_doc()
```

---

## 五、数据库 schema（DDL 草稿）

```sql
-- 启用扩展
CREATE EXTENSION IF NOT EXISTS vector;
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- 知识条目（带权重的 RAG 核心表）
CREATE TABLE knowledge_items (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    content         TEXT NOT NULL,
    embedding       vector(1024) NOT NULL,        -- BGE-large-zh 维度
    source          TEXT NOT NULL,                -- 'lark_chat' / 'lark_doc' / 'bitable' / 'escalation_decision'
    source_ref      TEXT,                         -- 飞书消息 ID / 文档 ID 等
    metadata        JSONB DEFAULT '{}'::jsonb,
    -- 加权字段
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    last_hit_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    hit_count       INTEGER NOT NULL DEFAULT 0
);
CREATE INDEX idx_knowledge_embedding ON knowledge_items USING ivfflat (embedding vector_cosine_ops);
CREATE INDEX idx_knowledge_source ON knowledge_items (source);

-- 分身状态
CREATE TABLE twin_state (
    twin_id         TEXT PRIMARY KEY,
    owner_user_id   TEXT NOT NULL,
    role            TEXT NOT NULL,
    config          JSONB NOT NULL,               -- capabilities, policy 引用
    memory_summary  TEXT,                         -- 缓存的人格化上下文摘要
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- A2A 消息（审计 + 可解释性）
CREATE TABLE a2a_messages (
    message_id      UUID PRIMARY KEY,
    conversation_id UUID NOT NULL,
    trace_id        TEXT NOT NULL,
    type            TEXT NOT NULL,
    from_twin_id    TEXT,                          -- null 表示人类回流
    from_owner_id   TEXT NOT NULL,
    to_twin_id      TEXT,
    to_broadcast    BOOLEAN DEFAULT FALSE,
    parent_id       UUID,
    created_at      TIMESTAMPTZ NOT NULL,
    expires_at      TIMESTAMPTZ,
    payload         JSONB NOT NULL,
    metadata        JSONB,
    UNIQUE (conversation_id, message_id)
);
CREATE INDEX idx_a2a_conversation ON a2a_messages (conversation_id, created_at);
CREATE INDEX idx_a2a_trace ON a2a_messages (trace_id);

-- 会话状态机持久化
CREATE TABLE conversations (
    conversation_id  UUID PRIMARY KEY,
    trace_id         TEXT NOT NULL,
    initiator_twin_id TEXT NOT NULL,
    state            TEXT NOT NULL,                -- initiated / in_progress / agreed / conflict / timeout / cancelled / escalated / resolved_by_human
    started_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    last_active_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    ended_at         TIMESTAMPTZ,
    summary          TEXT
);
CREATE INDEX idx_conversations_state ON conversations (state, last_active_at);

-- 升级队列
CREATE TABLE escalations (
    id               UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    conversation_id  UUID NOT NULL REFERENCES conversations(conversation_id),
    target_user_id   TEXT NOT NULL,
    reason           TEXT NOT NULL,
    summary          TEXT NOT NULL,
    options          JSONB NOT NULL,
    urgency          TEXT NOT NULL DEFAULT 'medium',
    created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    resolved_at      TIMESTAMPTZ,
    selected_option  TEXT,
    lark_card_id     TEXT
);

-- 抓取源配置（隐私边界落地：D-003）
CREATE TABLE chat_sources (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    chat_type       TEXT NOT NULL,                  -- 'group' / 'private'
    chat_id         TEXT NOT NULL,
    enabled         BOOLEAN NOT NULL DEFAULT TRUE,  -- 群默认 true，私聊 opt-in
    authorized_by   TEXT,                            -- 私聊：哪个用户授权的
    last_synced_at  TIMESTAMPTZ,
    UNIQUE (chat_type, chat_id)
);

-- 审计日志（所有分身行为可追溯）
CREATE TABLE audit_logs (
    id              BIGSERIAL PRIMARY KEY,
    trace_id        TEXT NOT NULL,
    twin_id         TEXT,
    action          TEXT NOT NULL,                   -- 'tool_call' / 'lark_send' / 'a2a_send' / ...
    payload         JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_audit_trace ON audit_logs (trace_id, created_at);
```

---

## 六、配置文件

### 6.1 `.env.example`

```bash
# 飞书
LARK_APP_ID=cli_xxx
LARK_APP_SECRET=xxx
LARK_VERIFICATION_TOKEN=xxx
LARK_ENCRYPT_KEY=xxx

# Anthropic
ANTHROPIC_API_KEY=sk-ant-xxx
CLAUDE_MODEL_PRIMARY=claude-opus-4-7
CLAUDE_MODEL_LIGHT=claude-haiku-4-5

# 数据库
DATABASE_URL=postgresql+asyncpg://flowskip:flowskip@localhost:5432/flowskip

# RAG
EMBEDDING_MODEL=BAAI/bge-large-zh-v1.5
WEIGHT_LAMBDA=0.05
RAG_KNOWLEDGE_GAP_THRESHOLD=0.6
TWIN_CONFIDENCE_THRESHOLD=0.5

# A2A
A2A_MESSAGE_TIMEOUT_SEC=30
A2A_CONVERSATION_TIMEOUT_SEC=300
A2A_HEARTBEAT_INTERVAL_SEC=15
```

### 6.2 `config/twins.yaml`

```yaml
twins:
  - twin_id: twin-pm-alice
    owner_user_id: u-alice
    role: product_manager
    display_name: "Alice·分身"
    capabilities:
      - intent: impact_assessment
        domains: [field_change, scope_clarification]
      - intent: priority_decision
        domains: [field_change]

  - twin_id: twin-dev-bob
    owner_user_id: u-bob
    role: backend_engineer
    display_name: "Bob·分身"
    capabilities:
      - intent: impact_assessment
        domains: [field_change, api_change, schema_migration]

  - twin_id: twin-design-carol
    owner_user_id: u-carol
    role: designer
    display_name: "Carol·分身"
    capabilities:
      - intent: impact_assessment
        domains: [visual, user_flow]
```

### 6.3 `config/policies.yaml`

```yaml
policies:
  default:
    auto_decisions:
      - field_value_validation        # 字段值校验
      - schedule_meeting              # 建会
    require_escalation:
      - schema_migration              # 涉及迁移
      - cross_team_resource           # 跨组资源
      - financial                     # 金额相关
      - hr                            # 人事相关
    confidence_threshold: 0.5
    knowledge_gap_threshold: 0.6

  twin-pm-alice:
    inherits: default
    auto_decisions_extra:
      - field_rename                  # 仅命名变更可代决
```

---

## 七、启动顺序与进程模型

单进程，`main.py` 编排：

```python
# src/flowskip/main.py 伪代码
async def lifespan(app: FastAPI):
    # 1. 配置 + 日志
    settings = Settings()
    setup_logging(settings)

    # 2. DB
    engine = create_async_engine(settings.database_url)
    await run_alembic_upgrade(engine)

    # 3. RAG
    embedding = await EmbeddingModel.load(settings.embedding_model)  # warm up BGE
    rag = RAGService(engine, embedding)

    # 4. MessageBus
    twin_configs = load_twins_yaml("config/twins.yaml")
    bus = InProcessBus(twin_configs, A2AMessageRepo(engine))

    # 5. 飞书接入
    lark = LarkAdapter(settings)
    await lark.start()  # WS 长连接

    # 6. 分身实例化
    registry = TwinRegistry()
    for cfg in twin_configs:
        agent = ClaudeAgent(model=settings.claude_model_primary, tools=build_tools(cfg, bus, rag, lark))
        twin = Twin(cfg, agent, bus, rag, lark)
        await bus.subscribe(twin.twin_id, twin.receive_a2a)
        registry.register(twin)

    # 7. 飞书消息路由
    lark.on_message(registry.dispatch_lark)
    lark.on_card_action(EscalationService(bus, registry).handle_card_action)

    # 8. EscalationService
    escalation = EscalationService(bus, lark, registry)
    await bus.subscribe_type("escalate", escalation.handle_escalate)

    # 9. APScheduler
    scheduler = AsyncIOScheduler()
    register_jobs(scheduler, rag, lark)
    scheduler.start()

    yield

    # 关停
    await lark.stop()
    scheduler.shutdown()
    await bus.shutdown()
    await engine.dispose()


app = FastAPI(lifespan=lifespan)
app.include_router(twins_router)
app.include_router(conversations_router)
app.include_router(auth_router)
```

> 整个系统在**一个 Python 进程**中：FastAPI server + APScheduler + LarkAdapter WS + 所有 Twin actors，通过 asyncio 协作调度。MVP 单点部署，Docker Compose 起 `postgres + app` 两个容器。

---

## 八、Prompt 模板骨架

放在 `config/prompts/`，运行时由 `twins/prompts.py` 加载并填模板变量。

### 8.1 `twin_system.md`（分身基础 system prompt）

```markdown
你是 {{owner_display_name}} 的赛博分身（"{{display_name}}"）。

## 你的职责
- 代理 {{owner_display_name}} 在飞书内的常规协同工作
- 在已授权范围内做决策；超出范围必须 escalate
- 与其他分身通过 A2A 协议协作

## 你的角色
{{role_description}}

## 你的授权范围（policy 摘要）
**可自动决策**：{{auto_decisions}}
**必须升级**：{{require_escalation}}

## 工具
你可以调用以下工具完成工作：{{tools_summary}}

## 决策原则
1. 不确定时优先 escalate，不要猜
2. 每次行动都要给出 `confidence`（0–1）。confidence < 0.5 时不直接回复，转 escalate
3. 引用知识时必须列出 `knowledge_refs`
4. 与其他分身通讯仅通过 a2a.* 工具——不要尝试直接访问对方信息

## 当前上下文
{{recent_memory_summary}}
```

### 8.2 `compact.md`（聊天 compact）

```markdown
你的任务是从下面的飞书聊天记录中抽取**结构化知识**，输出严格 JSON：

{{chat_block}}

输出 schema：
{
  "facts": ["..."],         // 客观事实
  "decisions": ["..."],     // 已达成的决策
  "todos": [{"owner": "...", "task": "..."}],
  "people": ["..."],        // 涉及的人物
  "topic": "..."            // 一句话主题
}

注意：
- 仅抽取明确陈述，不要推测
- 每条 fact / decision 独立成项，便于后续单独入库与检索
```

### 8.3 `escalation.md`（escalation 摘要）

```markdown
基于以下 conversation，为 {{target_user_display_name}} 生成 escalation 摘要：

{{conversation_messages}}

输出 schema：
{
  "summary": "...",         // 1-2 句话讲清楚发生了什么
  "options": [
    {"label": "...", "consequences": ["..."]}
  ],
  "urgency": "low | medium | high",
  "confidence": 0.0
}

要求：
- summary 是给本人看的，不是给其他分身的
- options 至少 2 个、最多 4 个
- 每个 option 必须列后果
```

---

## 九、测试策略

| 层级 | 工具 | 覆盖 |
|---|---|---|
| 单元 | pytest + pytest-asyncio | a2a/messages, a2a/conversation, knowledge/weight |
| 集成 | pytest-postgresql | knowledge/rag（加权 SQL）、storage/repositories |
| 端到端 | pytest + 真实 LLM 调用（小模型） | tests/e2e/test_field_change_scenario.py |

**E2E 关键点**：
- 用真实 PG（pgvector） + mock 飞书（不发真消息，但模拟 WS 事件）
- 至少 mock 一组分身的 Claude 输出（避免比赛期 quota 烧掉）
- 验证：消息序列 → conversation 状态 → 最终决议落表

---

## 十、第三阶段（实装）WBS

按依赖顺序，预计 3 天：

### Day 1：底座

1. `uv init` + `pyproject.toml` 锁定依赖
2. Docker Compose（postgres + pgvector 扩展）
3. SQLAlchemy 模型 + Alembic 初始迁移（覆盖 §五全部表）
4. `settings.py` + `.env.example`
5. structlog + trace_id 上下文
6. **冒烟**：`uvicorn` 起得来、连得上 PG

### Day 2：核心环路

7. `a2a/messages.py` + 单测
8. `a2a/bus.py:InProcessBus` + `a2a/conversation.py` + 超时 + 单测
9. `knowledge/embedding.py`（BGE 加载）
10. `knowledge/rag.py`（加权 SQL）+ 集成测
11. `twins/registry.py` + `twins/twin.py` 骨架
12. `tools/a2a_tools.py` + `tools/rag_tools.py`
13. **冒烟**：两个分身在测试里能完整走一次 request → response

### Day 3：飞书接入 + E2E

14. `lark/adapter.py` WS 长连接 + 事件订阅
15. `lark/cards.py` escalation 卡片
16. `tools/lark_tools.py`
17. `escalation/service.py`
18. `main.py` lifespan 编排所有
19. ETL job（可极简：只支持手动触发拉指定群）
20. **E2E 验收**：
    - 在测试群发"改字段"消息
    - 看到产品分身、研发分身、设计分身的 A2A 流转
    - 飞书弹出 escalation 卡片
    - 点选项后看到决议写回 / 知识入库

### 不在第三阶段范围（推到第四阶段或之后）

- 加权参数 tune（先用 D-012 的初值跑通）
- 私聊 opt-in UI（后台 API 可写，前端不做）
- Reranker 二阶段
- 监控大盘
- 性能压测

---

## 十一、第二阶段交付收口

至此，`docs/` 下四份文档形成闭环：

| 文档 | 角色 |
|---|---|
| [项目概要.md](../项目概要.md) | 愿景与机制（第一阶段） |
| [决策记录.md](./决策记录.md) | 19 条已决 + 10 条暂缓的 ADR 总账 |
| [技术选型.md](./技术选型.md) | 各层栈选型 + 依赖清单 |
| [A2A协议草案.md](./A2A协议草案.md) | 协议层数据结构与语义规则 |
| [MVP骨架.md](./MVP骨架.md) | 代码骨架、schema、调用链、第三阶段 WBS |

第三阶段开工时按本文 §十 WBS 推进即可。
