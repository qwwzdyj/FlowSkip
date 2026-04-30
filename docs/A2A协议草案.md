# A2A 协议草案（Agent-to-Agent Protocol）

> 第二阶段产出物 · v0.1
> 更新日期：2026-04-28
> 关联文档：[决策记录.md](./决策记录.md)、[技术选型.md](./技术选型.md)、[MVP骨架.md](./MVP骨架.md)

---

## 一、文档目的与范围

本文档定义 **分身之间通讯（A2A）的协议层**：消息格式、会话生命周期、超时与重试、escalation 触发规则、可解释性约定。

**本文档覆盖**：
- 协议层数据结构（envelope、payload、状态机）
- 协议层语义规则（超时、重试、幂等、隔离）
- MVP 阶段的简化策略（capability 静态注册、conflict 直接 escalate 等）

**本文档不覆盖**（去向其他文档）：
- 协议传输实现（→ [技术选型.md §3.3](./技术选型.md) D-007 已定为进程内 asyncio bus）
- 协议在代码层的目录组织、模块划分（→ [MVP骨架.md](./MVP骨架.md)）
- 飞书消息卡片层的 escalation UI（→ MVP 骨架）

---

## 二、设计原则

1. **异步优先**：所有 A2A 调用默认非阻塞——发起方不等同步返回，结果通过 `conversation_id` 回流匹配。
2. **可解释性强制**：每条 A2A 消息持久化到 `a2a_messages` 表，可经 `trace_id` 串联到飞书原始触发与知识库引用。
3. **跨语言友好**：envelope 用 JSON 表达，schema 同时给出 JSON Schema 与 pydantic 类型，未来换 Go/Rust 服务无需改协议。
4. **幂等可重放**：接收方根据 `(conversation_id, message_id)` 去重；同一消息允许被重复投递而不引发副作用。
5. **保守授权**：分身仅能通过 A2A 显式请求访问对方信息；**不允许直接读对方 memory 或私有上下文**。
6. **人类可旁路**：任何 conversation 在任意状态都可由人类介入终止或接管。

---

## 三、Conversation 模型

A2A 通讯以 **Conversation（协同会话）** 为核心单元——一次完整的"需求对齐"或"字段确认"。

### 3.1 标识

| 字段 | 含义 |
|---|---|
| `conversation_id` | UUID v4，一次协同的全局标识，贯穿所有相关消息 |
| `trace_id` | 跨服务追踪 ID；通常等于触发该会话的飞书消息 ID 或更高层 trace |
| `initiator_twin_id` | 发起方分身 ID |

`conversation_id` 是 A2A 的"主键"——所有响应、超时检测、escalation 都按它聚合。

### 3.2 状态机

```
        ┌───────────────────────────────────────────┐
        │                                           │
        ▼                                           │
   [initiated] ──→ [in_progress] ──┬──→ [agreed] ──┘    （正常达成一致）
                       │           ├──→ [conflict] ─→ [escalated]
                       │           ├──→ [timeout] ──→ [escalated]
                       │           └──→ [cancelled]
                       └──→ [escalated] ─→ [resolved_by_human]
```

| 状态 | 触发条件 | 后续 |
|---|---|---|
| `initiated` | 发起方创建会话，尚未发出第一条 request | 转 `in_progress`（出第一条 request 后） |
| `in_progress` | 至少一方已发出 request 且未完结 | 各种终态 |
| `agreed` | 所有参与方达成一致 | 终态 |
| `conflict` | 多分身意见分歧且分身层无法调和 | 转 `escalated` |
| `timeout` | 超过 conversation 总超时（默认 5 分钟） | 转 `escalated` |
| `cancelled` | 发起方撤回 / 人类终止 | 终态 |
| `escalated` | 推送到人类裁决 | 等人类回流，转 `resolved_by_human` |
| `resolved_by_human` | 人类已裁决，分身回写决议 | 终态 |

---

## 四、消息 envelope

所有 A2A 消息共享同一 envelope，业务内容放在 `payload`。

```json
{
  "message_id":      "uuid-v4",
  "conversation_id": "uuid-v4",
  "trace_id":        "string",
  "type":            "request | response | clarify | ack | propose | escalate | cancel | heartbeat",
  "from":            { "twin_id": "...", "owner_user_id": "..." },
  "to":              { "twin_id": "..." } | { "broadcast": true, "filter": {...} },
  "parent_id":       "uuid-v4 | null",
  "created_at":      "ISO-8601",
  "expires_at":      "ISO-8601 | null",
  "payload":         { ...类型相关... },
  "metadata": {
    "confidence":    0.0,
    "knowledge_refs": ["knowledge_item_id1", "..."],
    "human_visible": true,
    "tags":          ["..."]
  }
}
```

字段说明：

| 字段 | 必填 | 说明 |
|---|---|---|
| `message_id` | 是 | 单条消息唯一 ID（去重、引用用） |
| `conversation_id` | 是 | 所属会话 ID |
| `trace_id` | 是 | 跨服务追踪；通常承接触发该会话的飞书消息 ID |
| `type` | 是 | 消息类型，见 §五 |
| `from.twin_id` | 是 | 发送方分身 ID |
| `from.owner_user_id` | 是 | 分身的本人 ID（escalation 时用） |
| `to` | 是 | 接收方；可是单点也可广播 |
| `parent_id` | 否 | 引用的上一条消息 ID（response 必须填，指向对应 request） |
| `expires_at` | 否 | 该消息层面的超时（独立于会话总超时） |
| `metadata.confidence` | 否 | 发送方对自身判断的置信度（0–1）；低于阈值时接收方应触发 clarify 或 escalate |
| `metadata.knowledge_refs` | 否 | 本消息依据的知识条目 ID 列表，用于可解释性 |
| `metadata.human_visible` | 否 | 该消息是否允许在飞书侧渲染给人类（默认 true） |

---

## 五、消息类型

MVP 定义 8 种类型，按使用频次排序：

### 5.1 `request`

发起一项需求/查询，期待 `response`。

```json
"payload": {
  "intent": "impact_assessment",          // 业务意图，枚举或字符串
  "subject": "字段:user_level",
  "change": { "from": "1-5", "to": "1-10" },
  "questions": [
    "存储层是否需要 schema 迁移？",
    "前端校验逻辑是否会受影响？"
  ],
  "deadline": "2026-04-29T18:00:00Z"
}
```

### 5.2 `response`

对 `request` 的回复（`parent_id` 必须指向原 request）。

```json
"payload": {
  "status": "ok | partial | rejected",
  "answer": "字段值域扩展涉及 user 表迁移，预计 2 人天",
  "details": {
    "schema_migration_needed": true,
    "estimated_effort_pd": 2,
    "blocking_issues": []
  },
  "follow_up_required": false
}
```

### 5.3 `clarify`

接收方信息不足，反问发起方（不是 escalation——分身之间可以澄清）。

```json
"payload": {
  "questions": ["改动是仅前端展示还是包含后端校验？"],
  "missing_context": ["后端校验当前实现位置未知"]
}
```

### 5.4 `ack`

确认已收到 request、即将处理（用于长任务，避免发起方误判超时）。

```json
"payload": {
  "estimated_completion_at": "2026-04-29T15:00:00Z"
}
```

### 5.5 `propose`

主动提议——不基于 request，而是分身基于自身观察推送给其他分身。

```json
"payload": {
  "topic": "建议同步设计稿",
  "rationale": "字段值域扩展后，UI 上等级展示需调整",
  "suggested_action": "review_design"
}
```

### 5.6 `escalate`

升级到人类。可由任一分身在任意状态触发。详见 §八。

### 5.7 `cancel`

撤回会话。发起方主动 cancel，或参与方在收到 cancel 后停止处理。

```json
"payload": {
  "reason": "需求方临时取消"
}
```

### 5.8 `heartbeat`

长任务保活。在 `ack` 之后、`response` 之前定期发送，刷新会话 `last_active_at`。

```json
"payload": {
  "progress": 0.6,
  "note": "正在检索关联设计稿"
}
```

---

## 六、Capability 注册（MVP：静态）

每个分身在启动时声明自己能处理的 `intent` 列表。MVP 用静态配置文件，不做运行时发现。

```yaml
# config/twins.yaml
twins:
  - twin_id: twin-pm-alice
    owner_user_id: u-alice
    role: product_manager
    capabilities:
      - intent: impact_assessment
        domains: [field_change, scope_clarification]
      - intent: priority_decision
        domains: [field_change]

  - twin_id: twin-dev-bob
    owner_user_id: u-bob
    role: backend_engineer
    capabilities:
      - intent: impact_assessment
        domains: [field_change, api_change, schema_migration]

  - twin_id: twin-design-carol
    owner_user_id: u-carol
    role: designer
    capabilities:
      - intent: impact_assessment
        domains: [visual, user_flow]
```

发起 request 时，`MessageBus` 按 `(intent, domain)` 路由到匹配的分身。**无匹配能力时**：发起方收到 `clarify` 类型回复，附 `missing_capability` 元信息，由发起方决定是否 escalate。

> 动态 capability 发现（运行时声明、能力变更通知）留作 P-006，进暂缓决策。

---

## 七、超时、重试、幂等

### 7.1 超时层级

| 层级 | 默认值 | 说明 |
|---|---|---|
| 单条消息 `expires_at` | 30 秒 | request 默认 30s 内未收到 response 即视为超时 |
| Conversation 总超时 | 5 分钟 | 整个会话从 `initiated` 起算，无论中间有多少消息 |
| Heartbeat 间隔 | 15 秒 | 长任务每 15s 发一次 heartbeat，超过 3 个间隔未到即视为接收方失联 |

可由发起方通过 `expires_at` 显式覆盖默认值。

### 7.2 重试

```
request 发出
  └── 未在 30s 内收到 response/ack/clarify
       └── 自动重试 1 次（同 conversation_id，新 message_id）
            └── 仍未响应
                 └── escalate（type = escalate, 原因 = no_response_from <twin_id>）
```

**为什么只重试 1 次**：A2A 不是不可靠网络通讯，进程内总线丢消息说明接收方分身实质故障；多次重试只会拖延 escalation。

### 7.3 幂等性约定

接收方必须按 `(conversation_id, message_id)` 去重：

- 同一 `message_id` 重复到达 → 只处理第一次，后续直接返回上次的处理结果（或忽略）
- 同一 `conversation_id` 下相同语义的 request 重发 → 视为同一意图，不重复触发副作用

实现层面：`a2a_messages` 表的 `(conversation_id, message_id)` 是唯一约束。

---

## 八、Escalation 规则

> 这是分身可信度的关键机制——呼应项目概要 §三.3。

### 8.1 触发条件（任一即可）

1. **超出授权范围**：分身的 `policy` 表声明该决策需本人确认（如金额、人事、产品方向）
2. **知识空缺**：RAG 召回 top-K 的最高 score 低于阈值（默认 0.6）
3. **置信度过低**：分身对自身判断的 `confidence < 0.5`
4. **分身冲突**：两次互发 `propose` 后仍意见不一致（见 §九）
5. **超时无响应**：见 §7.2
6. **人类显式调用**：本人主动接管

### 8.2 Escalation payload

```json
"payload": {
  "reason": "out_of_authority | knowledge_gap | low_confidence | conflict | timeout | human_takeover",
  "summary": "用户等级值域从 1-5 改 1-10。研发分身评估需迁移 user 表（2pd），设计分身确认无视觉影响。建议本人确认是否启动迁移。",
  "context_snapshot": {
    "conversation_id": "...",
    "key_messages": ["msg-001", "msg-005", "msg-007"],
    "knowledge_refs": ["k-123", "k-456"]
  },
  "options": [
    {
      "id": "opt-1",
      "label": "确认启动迁移",
      "consequences": ["排期 2pd", "需协调测试"]
    },
    {
      "id": "opt-2",
      "label": "暂不迁移，先做兼容方案",
      "consequences": ["前端需做值域转换", "存在历史数据不一致风险"]
    },
    {
      "id": "opt-3",
      "label": "撤回此次变更"
    }
  ],
  "urgency": "low | medium | high",
  "target_human": "u-alice",
  "deadline": "2026-04-29T18:00:00Z"
}
```

### 8.3 人类回流

人类通过飞书消息卡片点击选项后，分身侧收到回调，构造一条 `type=response, from.twin_id=null, from.owner_user_id=...` 的特殊消息，写入会话，conversation 状态转 `resolved_by_human`，分身基于决议继续后续动作（如回写飞书文档）。

> 人类的回流消息在 envelope 中以 `from.twin_id = null` 标识，便于审计区分"人类决议" vs "分身决议"。

---

## 九、Conflict reconciliation

两个分身对同一问题给出不一致回复时：

```
1. 发起方收到分歧回复
   └── 发起方分身整理双方依据，给两方各发 type=propose，附对方观点
        └── 各方有 1 次机会修正（在新 propose 中给出更新后的判断）
             ├── 修正后一致 → conversation 转 agreed
             └── 仍不一致 → 自动 escalate（reason=conflict）
```

**MVP 不引入"调解分身"**——直接 escalate 给人类（呼应 P-003）。

调解分身机制是 v2 扩展点：可设计 `twin-arbiter`（如该字段的 owner，或资历更高的角色），在多分身僵持时介入仲裁。

---

## 十、可解释性与审计

### 10.1 持久化

每条 A2A 消息**进入 `MessageBus` 时同步写 `a2a_messages` 表**——同步而非异步，保证不丢日志。

```sql
CREATE TABLE a2a_messages (
  message_id      UUID PRIMARY KEY,
  conversation_id UUID NOT NULL,
  trace_id        TEXT NOT NULL,
  type            TEXT NOT NULL,
  from_twin_id    TEXT,
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
```

### 10.2 链路追溯

`trace_id` 串联：

```
飞书消息（用户："改字段")
   ├── trace_id = lark-msg-xyz
   └── 触发分身 A 的决策
        └── A 发起 conversation
             └── A2A 消息 1, 2, 3...   （同 trace_id）
                  └── 引用 knowledge_item k-123
                       └── 该 knowledge_item 的来源是 lark-msg-abc（更早的群聊）
```

**任意一条 A2A 消息**都能通过 `trace_id` + `knowledge_refs` 反向追到"为什么这么决策"。

### 10.3 Memory 隔离

- 分身 A 不允许直接读取分身 B 的 memory
- 分身 A 需要 B 的某项信息时，**必须**发 `request` 类型 A2A 消息
- B 决定是否在 `response` 中暴露该信息（基于 B 自己的 policy）

这是 D-017（待补登）的核心约束。

---

## 十一、JSON Schema 与 pydantic 类型

### 11.1 JSON Schema（envelope 摘要）

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "A2AMessage",
  "type": "object",
  "required": ["message_id", "conversation_id", "trace_id", "type", "from", "to", "created_at", "payload"],
  "properties": {
    "message_id":      { "type": "string", "format": "uuid" },
    "conversation_id": { "type": "string", "format": "uuid" },
    "trace_id":        { "type": "string" },
    "type": {
      "type": "string",
      "enum": ["request", "response", "clarify", "ack", "propose", "escalate", "cancel", "heartbeat"]
    },
    "from": {
      "type": "object",
      "required": ["owner_user_id"],
      "properties": {
        "twin_id":       { "type": ["string", "null"] },
        "owner_user_id": { "type": "string" }
      }
    },
    "to": {
      "oneOf": [
        { "type": "object", "required": ["twin_id"], "properties": { "twin_id": { "type": "string" } } },
        { "type": "object", "required": ["broadcast"], "properties": { "broadcast": { "const": true }, "filter": { "type": "object" } } }
      ]
    },
    "parent_id":  { "type": ["string", "null"], "format": "uuid" },
    "created_at": { "type": "string", "format": "date-time" },
    "expires_at": { "type": ["string", "null"], "format": "date-time" },
    "payload":    { "type": "object" },
    "metadata":   { "type": "object" }
  }
}
```

各 `type` 的 `payload` schema 见 §五各小节示例（实装时每种 type 对应一个独立 JSON Schema 文件）。

### 11.2 pydantic 类型（草稿）

```python
from datetime import datetime
from enum import Enum
from typing import Literal, Union
from uuid import UUID

from pydantic import BaseModel, Field


class MessageType(str, Enum):
    REQUEST = "request"
    RESPONSE = "response"
    CLARIFY = "clarify"
    ACK = "ack"
    PROPOSE = "propose"
    ESCALATE = "escalate"
    CANCEL = "cancel"
    HEARTBEAT = "heartbeat"


class FromAddress(BaseModel):
    twin_id: str | None  # null 表示人类回流
    owner_user_id: str


class ToAddressDirect(BaseModel):
    twin_id: str


class ToAddressBroadcast(BaseModel):
    broadcast: Literal[True] = True
    filter: dict | None = None


class Metadata(BaseModel):
    confidence: float | None = None
    knowledge_refs: list[str] = Field(default_factory=list)
    human_visible: bool = True
    tags: list[str] = Field(default_factory=list)


class A2AMessage(BaseModel):
    message_id: UUID
    conversation_id: UUID
    trace_id: str
    type: MessageType
    from_: FromAddress = Field(alias="from")
    to: Union[ToAddressDirect, ToAddressBroadcast]
    parent_id: UUID | None = None
    created_at: datetime
    expires_at: datetime | None = None
    payload: dict
    metadata: Metadata = Field(default_factory=Metadata)

    class Config:
        populate_by_name = True


# payload 各类型用单独的 model 表达，pydantic 在反序列化时按 type 分发
class RequestPayload(BaseModel):
    intent: str
    subject: str
    questions: list[str]
    deadline: datetime | None = None
    # 业务字段允许 extra
    class Config:
        extra = "allow"


class ResponsePayload(BaseModel):
    status: Literal["ok", "partial", "rejected"]
    answer: str
    details: dict = Field(default_factory=dict)
    follow_up_required: bool = False


# ... 其他 type 的 payload 类似
```

---

## 十二、完整示例：改字段场景

呼应项目概要 §七。完整消息序列（省略部分字段以聚焦关键内容）：

### 步骤 1：产品本人触发

飞书私聊 → 产品分身 `twin-pm-alice`：
> "把用户等级值域从 1–5 改成 1–10"

产品分身解析意图，创建 conversation `conv-001`，trace_id 承接飞书消息 ID。

### 步骤 2：广播 request

```json
{
  "message_id": "msg-001",
  "conversation_id": "conv-001",
  "trace_id": "lark-msg-xyz",
  "type": "request",
  "from": { "twin_id": "twin-pm-alice", "owner_user_id": "u-alice" },
  "to":   { "broadcast": true, "filter": { "intent": "impact_assessment", "domains": ["field_change"] } },
  "created_at": "2026-04-28T10:00:00Z",
  "expires_at": "2026-04-28T10:00:30Z",
  "payload": {
    "intent": "impact_assessment",
    "subject": "字段:user_level",
    "change": { "from": "1-5", "to": "1-10" },
    "questions": [
      "存储层是否需要 schema 迁移？",
      "前端校验逻辑是否会受影响？",
      "有无关联的视觉/交互需要更新？"
    ]
  },
  "metadata": { "confidence": 0.9 }
}
```

`MessageBus` 路由到 `twin-dev-bob` 与 `twin-design-carol`。

### 步骤 3：研发分身回复

```json
{
  "message_id": "msg-002",
  "conversation_id": "conv-001",
  "trace_id": "lark-msg-xyz",
  "type": "response",
  "from": { "twin_id": "twin-dev-bob", "owner_user_id": "u-bob" },
  "to":   { "twin_id": "twin-pm-alice" },
  "parent_id": "msg-001",
  "created_at": "2026-04-28T10:00:08Z",
  "payload": {
    "status": "ok",
    "answer": "字段值域扩展涉及 user 表迁移，预计 2 人天",
    "details": {
      "schema_migration_needed": true,
      "estimated_effort_pd": 2,
      "blocking_issues": []
    }
  },
  "metadata": {
    "confidence": 0.85,
    "knowledge_refs": ["k-user-schema-v3", "k-migration-history"]
  }
}
```

### 步骤 4：设计分身回复

```json
{
  "message_id": "msg-003",
  "conversation_id": "conv-001",
  "trace_id": "lark-msg-xyz",
  "type": "response",
  "from": { "twin_id": "twin-design-carol", "owner_user_id": "u-carol" },
  "to":   { "twin_id": "twin-pm-alice" },
  "parent_id": "msg-001",
  "created_at": "2026-04-28T10:00:11Z",
  "payload": {
    "status": "ok",
    "answer": "等级展示组件支持任意区间，无视觉影响",
    "details": { "components_affected": [] }
  },
  "metadata": { "confidence": 0.95, "knowledge_refs": ["k-design-system-level"] }
}
```

### 步骤 5：产品分身汇总并 escalate（涉及迁移成本，超出授权）

```json
{
  "message_id": "msg-004",
  "conversation_id": "conv-001",
  "trace_id": "lark-msg-xyz",
  "type": "escalate",
  "from": { "twin_id": "twin-pm-alice", "owner_user_id": "u-alice" },
  "to":   { "twin_id": null },
  "parent_id": "msg-001",
  "created_at": "2026-04-28T10:00:12Z",
  "payload": {
    "reason": "out_of_authority",
    "summary": "用户等级值域 1-5 → 1-10。研发：需 user 表迁移（2pd），无阻塞；设计：无视觉影响。建议确认是否启动迁移。",
    "context_snapshot": {
      "conversation_id": "conv-001",
      "key_messages": ["msg-001", "msg-002", "msg-003"],
      "knowledge_refs": ["k-user-schema-v3", "k-design-system-level"]
    },
    "options": [
      { "id": "opt-1", "label": "确认启动迁移（2pd）", "consequences": ["排期 2pd", "需协调测试"] },
      { "id": "opt-2", "label": "暂不迁移，做前端兼容方案" },
      { "id": "opt-3", "label": "撤回此次变更" }
    ],
    "urgency": "medium",
    "target_human": "u-alice"
  }
}
```

### 步骤 6：飞书消息卡片推给本人

产品分身把 `escalate` 转译为飞书 interactive card 推给 Alice，3 个选项一键点击。

### 步骤 7：人类回流

Alice 点 `opt-1`，飞书回调进 `LarkAdapter` → 转换为 `type=response, from.twin_id=null` 的特殊消息：

```json
{
  "message_id": "msg-005",
  "conversation_id": "conv-001",
  "trace_id": "lark-msg-xyz",
  "type": "response",
  "from": { "twin_id": null, "owner_user_id": "u-alice" },
  "parent_id": "msg-004",
  "payload": {
    "status": "ok",
    "answer": "确认启动迁移",
    "details": { "selected_option": "opt-1" }
  }
}
```

Conversation 转 `resolved_by_human`。产品分身基于决议执行后续动作（写飞书文档、通知研发分身排期）。

---

## 十三、本文档暴露的新决策（待回写到决策记录）

| 编号 | 议题 | 选择 |
|---|---|---|
| D-013 | 单条消息默认超时 | 30 秒；conversation 总超时 5 分钟；heartbeat 间隔 15 秒 |
| D-014 | 重试策略 | 单消息超时仅自动重试 1 次，再失败即 escalate |
| D-015 | Capability 发现 | MVP 用静态 yaml 注册表（运行时发现进 P-006 暂缓） |
| D-016 | Conflict 仲裁 | MVP 直接 escalate，不引入"调解分身"（呼应 P-003） |
| D-017 | Memory 隔离 | 分身间不直接读取对方 memory，必须经 A2A 显式 request |
| D-018 | 知识空缺 escalation 阈值 | RAG 召回 top-1 score < 0.6 触发 |
| D-019 | 置信度 escalation 阈值 | 分身自评 `confidence < 0.5` 触发 |

> 确认本文后我会按 ADR 格式补登。

---

## 十四、暂缓 / 后续迭代

| 编号 | 议题 | 暂缓原因 |
|---|---|---|
| P-006 | 动态 capability 发现 | MVP 静态注册够用 |
| P-007 | A2A 消息加密 / 签名 | 进程内通讯无需，跨机时再加 |
| P-008 | 调解分身（arbiter twin） | conflict 直接 escalate 已能满足 MVP |
| P-009 | 多人类协同决策（escalate 给多个本人） | MVP 单人裁决先跑通 |
| P-010 | 消息优先级 / 队列 | MVP 任务量低，FIFO 即可 |
