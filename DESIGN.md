# 企业微信机器人 — 设计文档

> 状态：设计阶段（尚未开发）
> 目标：搭建一个能接收用户消息、调用业务 API 查询数据、再把结果回复给用户的企业微信机器人；同时支持普通微信用户接入。

---

## 1. 需求与目标

### 1.1 功能需求

- 用户向机器人发送文本消息（如 `查订单 12345`、`今日销售`、`help`）。
- 机器人解析消息意图，调用后端业务 API 获取数据。
- 机器人把数据格式化（文本 / Markdown / 卡片）后回复给用户。
- 支持企业内部员工（企业微信账号）。
- 支持企业外部普通微信用户（通过"微信客服"）。

### 1.2 非功能需求

| 项 | 目标 |
|---|---|
| 回调响应延迟 | < 1 s（必须 < 5 s，否则企业微信判超时） |
| 业务回复整体延迟 | < 3 s（P95） |
| 可用性 | 99.5%+ |
| 并发 | 支持 200 QPS 接收回调 |
| 安全 | HTTPS + 回调签名 + AES 解密 + 密钥不入库 |
| 可观测 | 全链路日志、消息去重、错误告警 |

---

## 2. 形态对比与选型

企业微信生态里有 3 种"机器人"形态，能力差别很大：

| 形态 | 能否接收用户消息 | 能否主动回 | 用户范围 | 是否符合本需求 |
|---|---|---|---|---|
| 群机器人 Webhook | ❌ | 只能往群里推 | 群成员 | ❌ 单向 |
| 自建应用 | ✅ 回调接收 | ✅ 调 API 发 | 企业内部员工 | ✅ |
| 微信客服 (Kf) | ✅ 回调接收 | ✅ 调 API 发 | 普通微信用户（外部 C 端） | ✅ |

**最终选型：自建应用 + 微信客服 双通道**，共享同一套后端处理逻辑。

> ⚠️ 关于"在普通微信里使用"：
> - 个人微信号严禁第三方协议自动化（itchat / wechaty 类方案违反协议，账号会被封）。
> - **官方合规路径** = 企业微信的「微信客服」。普通微信用户扫码进入客服会话，体验上跟微信好友聊天一致，界面只会多一个"企业"标识。

---

## 3. 总体架构

```
                               ┌──────────────────────────┐
   普通微信用户   ─────────────▶│  微信客服 (kf_*)          │
                               │  托管在企业微信里         │
   企业微信员工   ─────────────▶│  自建应用                 │
                               └────────────┬─────────────┘
                                            │ HTTP POST 回调（加密）
                                            ▼
                          ┌────────────────────────────────────┐
                          │            Bot Gateway              │
                          │ ┌────────────────────────────────┐ │
                          │ │ 1. 签名校验 + AES 解密          │ │
                          │ │ 2. 通道适配（App / Kf）         │ │
                          │ │ 3. 归一化为内部 Message         │ │
                          │ └─────────────┬──────────────────┘ │
                          │               ▼                     │
                          │ ┌────────────────────────────────┐ │
                          │ │ Dispatcher / Router             │ │
                          │ │  (msg_type + keyword/intent)    │ │
                          │ └─────────────┬──────────────────┘ │
                          │               ▼                     │
                          │ ┌────────────────────────────────┐ │
                          │ │ Command Handlers                │ │
                          │ │  QueryOrder / Help / Fallback…  │ │
                          │ └─────────────┬──────────────────┘ │
                          │               ▼                     │
                          │ ┌────────────────────────────────┐ │
                          │ │ Business API Client             │ │
                          │ │  超时 / 重试 / 熔断 / 缓存       │ │
                          │ └─────────────┬──────────────────┘ │
                          │               ▼                     │
                          │ ┌────────────────────────────────┐ │
                          │ │ Reply Renderer                  │ │
                          │ │  Text / Markdown / TextCard     │ │
                          │ └─────────────┬──────────────────┘ │
                          └───────────────┼────────────────────┘
                                          ▼
                          ┌────────────────────────────────────┐
                          │  WeCom API Client                   │
                          │  access_token 缓存 + 续期           │
                          │  发送消息（应用 / 客服）             │
                          │  限流 + 重试                        │
                          └───────────────┬────────────────────┘
                                          ▼
                              企业微信 OpenAPI
                                          │
                                          ▼
                                  用户收到回复
```

辅助组件：

```
   ┌──────────┐    ┌──────────┐    ┌──────────────┐
   │  Redis   │    │ Postgres │    │ Async Worker │
   │ token    │    │ 用户绑定 │    │ 慢任务消费    │
   │ 会话上下文│    │ 消息日志 │    │ (>2s 业务)    │
   │ 去重/限流 │    │ 审计     │    │              │
   └──────────┘    └──────────┘    └──────────────┘
```

---

## 4. 消息时序

### 4.1 快路径（业务 API 在 2s 内能返回）

```
微信好友          微信/企业微信        Bot Gateway        业务 API
   │ "查订单 123"    │                     │                  │
   │ ───────────────▶                     │                  │
   │                │ POST /callback(加密)│                  │
   │                │ ───────────────────▶│                  │
   │                │                     │ 验签 + 解密      │
   │                │                     │ 路由 → Handler   │
   │                │                     │ ──── GET ───────▶│
   │                │                     │ ◀──── JSON ─────│
   │                │                     │ 调用发送消息API   │
   │                │ ◀───────────────────│                  │
   │ ◀──────────────│                     │                  │
   │                │ 200 OK              │                  │
   │                │ ◀───────────────────│                  │
```

### 4.2 慢路径（业务 API 可能 > 3s）

```
Gateway 收到回调
  → 立刻 200 OK 返回空消息体（避免 5s 超时）
  → 把任务塞进 Redis Stream / MQ
  → Async Worker 消费：调业务 API、渲染、调"发送消息 API" 推送
  → 用户收到一条"主动消息"
```

> 企业微信回调对响应时间要求是 5 秒，超时会重试，所以**任何 > 2s 的业务一律走慢路径**。

---

## 5. 模块划分

### 5.1 Callback Server

- 路由：
  - `GET  /wecom/callback/app`   — 自建应用 URL 校验（echostr）
  - `POST /wecom/callback/app`   — 自建应用消息回调
  - `GET  /wecom/callback/kf`    — 微信客服 URL 校验
  - `POST /wecom/callback/kf`    — 微信客服事件回调
- 统一处理：`msg_signature / timestamp / nonce` 验签 → `WXBizMsgCrypt` 解密 → 解析 XML / JSON。

### 5.2 Channel Adapter

把不同来源的消息映射到内部统一模型：

```
Message {
  channel:     "app" | "kf"
  msg_id:      str           // 用于幂等
  from_user:   str           // app: UserID；kf: external_userid
  to_user:     str           // app: AgentId；kf: open_kfid
  msg_type:    "text" | "image" | "voice" | "event" | ...
  content:     str
  raw:         dict          // 原始报文，保留备用
  received_at: datetime
}
```

> 微信客服的回调结构和自建应用差异较大：客服回调里只给你一个"有新事件"的信号，需要你**主动调 `sync_msg`** 拉取消息列表；这块在 Adapter 里屏蔽掉。

### 5.3 Dispatcher / Router

- 一级路由：按 `msg_type` 分流（文本走命令，事件走事件处理）。
- 二级路由：文本消息按"前缀关键字"匹配 Handler；可后续替换为 NLP 意图识别。
- 命中不到 → `FallbackHandler` 给出帮助文案。

### 5.4 Command Handlers

每个业务命令一个 Handler，独立可测。MVP 阶段建议：

| Handler | 触发 | 行为 |
|---|---|---|
| HelpHandler | `help`、`帮助`、`?` | 返回支持的命令列表 |
| QueryOrderHandler | `查订单 <id>` | 调订单 API |
| QuerySalesHandler | `今日销售` | 调销售 API |
| BindHandler | `绑定 <token>` | 把微信用户和系统账号关联 |
| FallbackHandler | 兜底 | 提示语 + 引导到 help |

### 5.5 Business API Client

- HTTP 客户端封装（`httpx` / `resty`）。
- 公共能力：
  - 连接池
  - 超时（connect 1s, read 2s）
  - 重试（指数退避，最多 2 次，仅对幂等 GET）
  - 熔断（失败率 > 50% 半开）
  - 短期缓存（同一参数 60s 命中缓存，看业务可选）
- 鉴权：业务 API 接入侧带 `Authorization: Bearer <svc-token>` 或 HMAC 签名。

### 5.6 Reply Renderer

- 输出类型：
  - 短结果 → `text`
  - 结构化结果 → `markdown`（企业微信内）/ `text`（微信客服，markdown 支持有限）
  - 强调结果 → `textcard`（自建应用支持）
- 模板用 `Jinja2` 或等价方案，便于业务方维护。

### 5.7 WeCom API Client

- `access_token`：
  - 自建应用：`gettoken?corpid&corpsecret`
  - 微信客服：单独的客服 secret，单独的 token，**两套不能混用**。
  - 缓存到 Redis，TTL 设为返回 expires_in - 300s。
  - **分布式锁**（Redis `SET NX`）防并发刷新。
- 发送：
  - 自建应用 → `cgi-bin/message/send`
  - 微信客服 → `cgi-bin/kf/send_msg`（48 小时内）；超过 48h 用 `send_msg_on_event`（需事件凭证）
- 错误码：对 `40014/42001`（token 失效）自动刷新一次并重试。

### 5.8 Async Worker

- 队列：Redis Stream（轻量）或 RabbitMQ（重量）。
- 消费者：拉任务 → 执行 Handler → 调 WeCom 发送。
- 死信：3 次失败入死信队列 + 告警。

### 5.9 Storage

| 存储 | 用途 | 关键 Key/表 |
|---|---|---|
| Redis | access_token | `wecom:token:app` `wecom:token:kf` |
| Redis | 去重 | `wecom:msg:<msg_id>` setnx 5min |
| Redis | 会话上下文 | `wecom:ctx:<user>` Hash, TTL 30min |
| Redis | 限流 | `rl:<user>:<minute>` INCR |
| Postgres | 用户绑定 | `user_binding(wecom_user, biz_user, …)` |
| Postgres | 消息日志 | `message_log(...)` 30 天 |
| Postgres | 审计 | `audit_log(...)` 180 天 |

---

## 6. 关键设计决策

| 决策 | 选择 | 理由 |
|---|---|---|
| 是否双通道 | 是（App + Kf） | 同时覆盖内部员工和外部微信用户 |
| 同步 vs 异步回复 | 默认异步 + 快路径优化 | 企业微信 5s 超时硬限制 |
| token 管理 | 集中缓存 + 分布式锁 | 防止多实例同时刷新被限流 |
| 消息去重 | 基于 MsgId 5 分钟窗 | 企业微信会重试 |
| 意图识别 | MVP 用关键字，预留 NLP 接口 | 先跑通，再上 LLM |
| 是否引入 LLM | 后续可选 | 兜底回复或自由问答阶段接入 |
| 多轮上下文 | Redis 短期 + 显式清除指令 | 控制成本和复杂度 |

---

## 7. 安全设计

1. **传输层**：所有回调 URL 必须 HTTPS（企业微信硬性要求）。
2. **应用层**：
   - `Token / EncodingAESKey / CorpSecret / KfSecret` 走环境变量 / KMS，**不入仓库**。
   - 回调消息签名校验（`SHA1(token+timestamp+nonce+encrypt)`）。
   - AES-CBC 解密后再校验 `ReceiveId`。
3. **业务层**：
   - 业务 API 调用带服务间 Token / HMAC。
   - 涉及用户数据的命令必须先**绑定**（用户在系统里生成一次性 token，发给机器人完成绑定），避免别人冒充。
   - 命令级权限校验：哪些命令需要哪些角色。
4. **风控**：
   - 单用户限流 30 条 / 分钟。
   - 命中敏感关键词的命令记审计。

---

## 8. 可观测性

- **日志**：结构化 JSON，字段含 `trace_id / channel / user / msg_id / handler / latency_ms / status`。
- **指标**（Prometheus）：
  - `wecom_callback_total{channel,result}`
  - `wecom_send_total{channel,errcode}`
  - `handler_latency_seconds{handler}`
  - `biz_api_latency_seconds{endpoint}`
- **告警**：
  - 回调 5xx > 1%（5 分钟）
  - `errcode != 0` 占比 > 5%
  - access_token 刷新失败
  - 队列堆积 > 1000

---

## 9. 部署拓扑

```
              ┌────────────┐
   外网 ─────▶│   Nginx    │ HTTPS, 证书, 限流
              └─────┬──────┘
                    ▼
       ┌───────────────────────────┐
       │  Bot Gateway (多实例)     │  K8s Deployment / Docker Compose
       └─────┬──────────────┬──────┘
             ▼              ▼
      ┌──────────┐    ┌────────────┐
      │  Redis   │    │ Postgres   │
      └──────────┘    └────────────┘
             ▲
             │
       ┌─────┴───────┐
       │ Async Worker│ (多实例)
       └─────────────┘
```

要求：
- 服务器必须有公网域名 + ICP 备案（企业微信回调要求）。
- 建议放在公司现有 K8s / 云上，套 Nginx + WAF。

---

## 10. 接口契约（草案）

### 10.1 回调（由企业微信发起）

- `POST /wecom/callback/app?msg_signature=&timestamp=&nonce=`
  - Body: 加密 XML
- `POST /wecom/callback/kf?msg_signature=&timestamp=&nonce=`
  - Body: 加密 XML，事件里带 `Token`，需要再调 `sync_msg` 拉真消息

### 10.2 业务 API（由 Bot 调你方系统）

约定一套统一前缀，例如：

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/api/v1/orders/{id}` | 查订单 |
| GET | `/api/v1/sales/today` | 查今日销售 |
| POST | `/api/v1/bind` | 绑定微信用户与系统账号 |

返回统一：

```json
{ "code": 0, "msg": "ok", "data": {...} }
```

### 10.3 命令规范（用户视角）

```
help                       查看帮助
查订单 <订单号>             订单详情
今日销售                    今日销售汇总
绑定 <token>                绑定账号（token 由系统生成）
解绑                        解除绑定
```

---

## 11. 技术选型对比

| 维度 | 路线 A：Python | 路线 B：Go |
|---|---|---|
| Web | FastAPI | Gin / Echo |
| 企业微信加解密 | wechatpy / 官方示例 | github.com/silenceper/wechat |
| 队列 | Celery + Redis | Asynq + Redis |
| HTTP Client | httpx | resty / net/http |
| ORM | SQLAlchemy | GORM |
| 适合场景 | 业务逻辑迭代快、可能接 LLM | 高并发、低延迟、稳定 |

> 推荐 MVP 用 **路线 A（Python + FastAPI）**，后期高并发再考虑迁移。

---

## 12. 里程碑（按交付物，不估时间）

| 里程碑 | 交付物 |
|---|---|
| M1 基础设施 | 域名/HTTPS、企业微信后台创建应用 + 客服、密钥下发到环境变量 |
| M2 通信骨架 | 回调验签、解密、URL 校验通过；access_token 缓存可用 |
| M3 最小双向 | 自建应用收到文本能 echo 回去 |
| M4 命令体系 | Dispatcher、HelpHandler、FallbackHandler、消息日志 |
| M5 业务接入 | 第一个业务命令（查订单）打通，含异步链路 |
| M6 微信客服 | Kf 通道接入，普通微信用户可对话 |
| M7 完整功能 | 绑定/解绑、权限校验、限流、审计 |
| M8 可观测与上线 | 指标、告警、压测、灰度发布 |

---

## 13. 风险与对策

| 风险 | 影响 | 对策 |
|---|---|---|
| 业务 API 慢导致 5s 超时 | 回调被企业微信判失败、重复推送 | 全部异步化 + 去重 |
| access_token 并发刷新被限频 | 全局发不出消息 | 分布式锁 + 提前续期 |
| 微信客服 48h 窗口限制 | 主动消息发不出 | 引导用户先发起会话；或申请 `send_msg_on_event` |
| 个人微信封号风险 | 服务中断 | 严格不走个人微信协议，仅走客服通道 |
| 密钥泄漏 | 整个企业微信能被冒用 | KMS / 仅环境变量 / 严格审计 |
| 业务命令滥用 | 信息泄漏 | 绑定 + 权限校验 + 限流 + 审计 |

---

## 14. 后续可演进方向

- 接入 LLM（GPT/通义/文心）做自由问答和意图识别，命令体系作为"工具调用 (function calling)"。
- 富交互：textcard、模板卡片、按钮（仅自建应用支持）。
- 主动推送：业务侧事件 → MQ → Bot 主动推消息（注意客服 48h 窗口）。
- 多租户：一个 Bot Gateway 服务多家企业。
- 知识库 + RAG：对接公司文档，做内部问答机器人。

---

## 15. 待确认问题（需求方反馈）

1. 用户范围：仅企业内部，还是也面向外部微信用户？（影响是否需要微信客服）
2. 业务 API 已经具备吗？大致响应时间？是否能改造？
3. 是否需要绑定系统账号？还是匿名问答即可？
4. 是否需要多轮对话（上下文）？
5. 部署目标：自有机房 / 公司云 / 公网云？是否有现成域名和备案？
6. 是否需要审计/合规要求（金融、政企）？
7. 是否计划接 LLM？预算如何？
