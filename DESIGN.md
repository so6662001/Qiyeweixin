# 企业微信机器人 — 设计文档（v3）

> 状态：设计阶段（尚未开发）
> 行业：**钢铁贸易**
> 目标：搭建一个企业微信智能机器人，对接公司已有的 8 项后端能力（查留货订单 / 查欠款 / 询价（异步报价） / 查装车重量 / 要材质书 / 接收或查结算单 / 接收付款凭证 / 提醒销售发货），覆盖内部员工和外部微信客户，接入 LLM（DeepSeek / 通义千问）做自然语言理解与多轮对话。

---

## 0. 需求方已确认的关键约束

| # | 问题 | 答复 | 对设计的影响 |
|---|---|---|---|
| 1 | 用户范围 | 内部 + 外部微信用户 | 双通道：自建应用 + 微信客服 |
| 2 | 业务 API | 已具备，**不可改造** | Bot 侧必须有 API 适配层（ACL） |
| 3 | 是否绑定账号 | 必须绑定 | 强绑定流程 + 数据级权限 |
| 4 | 多轮对话 | 跨天，多意图并存 | Topic + Task 状态机 |
| 5 | 部署 | 公网云，域名已备案 | 云原生 |
| 6 | 行业 | 钢铁贸易 | 强权限 + DLP + 审计 |
| 7 | LLM | DeepSeek + 通义千问 | LLM Orchestrator + 双供应商路由 |
| 8 | **实际后端能力（v3 新增）** | **8 项**：查留货订单 / 查欠款 / 询价（异步报价） / 查装车重量 / 要材质书 / 接收或查结算单 / 接收付款凭证 / 提醒销售发货 | **技能包按实际能力收敛**；**新增异步工单模型**；**新增业务系统反向回调通道**；**新增入站文件处理** |

---

## 1. 需求与目标

### 1.1 8 项能力的形态分类（v3 核心）

| # | 能力 | 形态 | 是否需要业务侧反向通知 | 是否涉及文件 | 是否跨用户 |
|---|---|---|---|---|---|
| 1 | 查留货订单 | 同步查询 | 否 | 否 | 否 |
| 2 | 查欠款 | 同步查询 | 否 | 否 | 否 |
| 3 | 询价 → 我们发起报价 | **异步工单** | **是**（报价完成回调） | 否 | 客户↔销售 |
| 4 | 查装车重量（磅单） | 同步查询 | 否 | 可能附磅单图 | 否 |
| 5 | 要材质书 | 同步查询 + 文件下发 | 否 | **是**（PDF） | 否 |
| 6 | 接收/查结算单 | **双向**：业务系统推 + 客户查询 | **是**（结算单生成回调） | **是**（PDF） | 客户↔财务 |
| 7 | 接收付款凭证 | **入站文件** | 否（处理完后回调财务即可） | **是**（图片/PDF） | 客户→财务/销售 |
| 8 | 提醒销售发货 | **跨用户转发** + 工单 | 业务系统可选回写 | 否 | **客户→销售** |

### 1.2 用户与场景

| 角色 | 实际能用到的能力 |
|---|---|
| **客户** | 查留货订单、查欠款（自己的）、发起询价、查装车重量、要材质书、查/确认结算单、上传付款凭证、催发货 |
| **销售** | 查客户的留货订单/欠款/装车重量、收到询价工单去系统报价、收到催发货工单去处理 |
| **财务** | 查欠款（汇总）、查结算单、查付款凭证、对账 |
| **管理员** | 查全部 + 工单监控 + 配置 |

### 1.3 非功能需求

| 项 | 目标 |
|---|---|
| 回调响应延迟 | < 1 s（硬限 < 5 s） |
| 同步查询整体延迟 | P95 < 3 s |
| LLM 回复整体延迟 | P50 < 3 s, P95 < 8 s |
| 异步工单 SLA | 询价 30min 内首次响应；催发货 1h 内首次响应（销售侧） |
| 文件上传单文件 | ≤ 20 MB；总并发 50 |
| 可用性 | 99.5%+ |
| 安全 | HTTPS + 签名 + AES + KMS + DLP |
| 数据保留 | 消息 180d；工单 1y；财务相关文件 5y |

---

## 2. 通道选型

| 形态 | 接收 | 发送 | 用户 | 选用 |
|---|---|---|---|---|
| 群机器人 Webhook | ❌ | 推到群 | 群成员 | 仅做内部告警 |
| **自建应用** | ✅ | ✅ | 企业内部员工 | ✅ |
| **微信客服 (Kf)** | ✅ | ✅ | 普通微信用户 | ✅ |

> 严禁走个人微信号协议。外部客户只走微信客服。

---

## 3. 总体架构（v3）

```
                    ┌──────────────────────────────────────┐
   普通微信用户  ──▶│  微信客服 (kf_*)                       │
                    │  托管在企业微信里                      │
   企业微信员工  ──▶│  自建应用                              │
                    └────────────────┬─────────────────────┘
                                     │ 回调 (加密)
                                     ▼
   ┌────────────────────────────────────────────────────────────────┐
   │                       Bot Gateway                                │
   │                                                                  │
   │  Callback Server → Channel Adapter → Identity & Binding         │
   │       → Permission Gate → Fast Path Router                      │
   │              ↓ (未命中)        ↓ (命中关键字)                    │
   │       Session State Manager   Command Handler                    │
   │              ↓                                                   │
   │       LLM Orchestrator (DeepSeek / Qwen 双路由)                  │
   │              ↓                                                   │
   │       Tool Registry (Function Calling 工具集)                    │
   │              ↓                                                   │
   │       Business API Adapter (ACL, 防腐层) ──▶ 现有业务 API        │
   │              ↓                                                   │
   │       Reply Composer + DLP ──▶ WeCom API Client                  │
   │                                                                  │
   │  ──────────────────── v3 新增三大模块 ──────────────────────     │
   │                                                                  │
   │  ① Workflow Engine (工单/任务状态机)                              │
   │      - 询价工单 / 催发货工单 / 付款凭证审核工单                   │
   │      - SLA 超时提醒、转派、关单                                   │
   │                                                                  │
   │  ② Inbound Webhook (业务系统反向回调)                             │
   │      POST /biz/event                                             │
   │      - quote.ready / settlement.created /                       │
   │        payment.verified / shipment.scheduled                    │
   │      → 触发主动消息回推用户                                       │
   │                                                                  │
   │  ③ Media Pipeline (文件/图片处理)                                 │
   │      - 入站：付款凭证上传 → OSS → 可选 OCR → 关联工单            │
   │      - 出站：材质书 PDF / 结算单 PDF → 临时 media_id → 发给用户  │
   └────────────────────────────────────────────────────────────────┘

   ┌──────────┐  ┌──────────┐  ┌──────────────┐  ┌────────────────┐
   │  Redis   │  │ Postgres │  │ Async Worker │  │ Audit / DLP    │
   │ token/ctx│  │ 绑定/会话│  │ LLM/工单/文件│  │ ES / SLS / OSS │
   │ 限流/锁  │  │ 工单/审计│  │ 主动推送     │  │ Prompt 留存    │
   └──────────┘  └──────────┘  └──────────────┘  └────────────────┘
```

---

## 4. 关键流程（v3，按 8 项能力分别画）

### 4.1 同步查询类（① 留货订单 / ② 欠款 / ④ 装车重量）

```
客户 → "查下我那张昨天的留货订单"
   ↓ (回调)
Gateway → 绑定/权限 → Session 取/建 Topic
   ↓
LLM 识别意图 = query_held_orders
槽位补齐：customer_id 来自绑定身份；时间「昨天」=date_range
   ↓
Function Call: query_held_orders(customer_id, date_range="yesterday")
   ↓
ACL → 业务 API → JSON
   ↓
Reply Composer 渲染（Markdown 列表 + 留货编号 + 品种 + 数量 + 留货截止日）
   ↓
后置 DLP → 发送
   ↓
Topic 更新 last_active_at；记录 tool_call_log
```

### 4.2 询价（异步工单 + 业务系统反向回调）

```
═══ 阶段 1：客户发起 ════════════════════════════════════
客户 → "螺纹钢 HRB400 25mm 200吨 武汉到货价，能给个报价吗"
   ↓
LLM 抽取槽位 → 验证齐全 → Function Call: submit_inquiry(...)
   ↓
ACL → 业务 API 创建询价单 INQ-20260528-001（在你们 ERP 里出现"待报价"）
   ↓
Workflow Engine 创建工单:
   workflow_id, type=inquiry, status=PENDING_QUOTE,
   creator=customer, assignee=sales_zhang,
   sla=30min, ref=INQ-20260528-001
   ↓
Bot → 客户(微信客服)："已收到询价 INQ-001，销售小张正在报价，预计 30 分钟内回复"
Bot → 销售小张(企业微信)：textcard «新询价 INQ-001 张总-武汉 螺纹钢200t,请处理»
   ↓ 同时 Workflow Engine 启动 SLA 计时

═══ 阶段 2：销售在 ERP 报价（不在 Bot 里报）════════════════
销售在公司 ERP 里填报价 → 提交 → ERP 触发 webhook
   ↓
POST /biz/event  { event:"quote.ready",
                   inquiry_id:"INQ-001",
                   price:3820, unit:"元/吨",
                   valid_until:"2026-05-29 18:00",
                   remark:"含税含运" }
   ↓
Inbound Webhook 验签 → 查 Workflow → 找到 creator(客户)
   ↓
Workflow Engine: status=PENDING_QUOTE → QUOTED
   ↓
Bot → 客户(主动消息)：
  "✅ 您的询价 INQ-001 报价：
   螺纹钢 HRB400 25mm × 200吨 武汉到货
   ¥3,820/吨（含税含运）
   有效期至 2026-05-29 18:00
   如需锁价或下单，请联系销售小张：13xxxxxxxxx"

═══ 注意点 ══════════════════════════════════════════════
- 客户在微信客服里要先发起会话才能被推主动消息。报价回推时如果超出 48h 窗口
  → 改成 SMS 通知 或 引导客户在微信里说一句"催一下报价"重新打开窗口
- 当前阶段 Bot 不直接下单/锁价（你们流程是销售在 ERP 里完成），符合 v3 约束
```

### 4.3 要材质书（同步查询 + 文件下发）

```
客户 → "炉号 24A0512 的材质书发我一份"
   ↓
LLM 意图=get_material_cert, 槽位 batch_no=24A0512
   ↓
Function Call: get_material_cert(batch_no, customer_id)
   ↓
ACL → 业务 API → 返回 PDF URL（一次性签名）
   ↓
Media Pipeline:
   1. 后端下载 PDF
   2. 上传到企业微信 media（临时素材，3 天有效）拿 media_id
   3. 缓存 (batch_no -> media_id) 1 天，避免重复上传
   ↓
WeCom 发送两条:
   - text："炉号 24A0512 材质书已发送，请查收"
   - file: media_id
   ↓
审计记录：who、批号、材质书 hash、发送时间
```

### 4.4 接收/查结算单（双向 + 文件）

```
方向 A：业务系统推送（结算单出账）
─────────────────────────────────
ERP 月初出账 → POST /biz/event { event:"settlement.created",
                                  customer_id, month, pdf_url, amount }
   ↓
Workflow Engine 创建工单 type=settlement_ack, status=PENDING_ACK, sla=72h
   ↓
Media Pipeline 拉 PDF → 上传企微 media
   ↓
Bot → 客户：
  "📄 您的 2026-05 月结算单已生成
   应付：¥1,234,567
   请查收附件并回复『确认』或『有问题』"
   附件 = settlement.pdf

客户回复『确认』 → Workflow 流转 status=ACKED → 通知财务
客户回复『有问题』+ 描述 → 创建子工单转销售/财务跟进

方向 B：客户主动查
─────────────────────────────────
客户 → "上个月的结算单还在吗"
   ↓
Function Call: query_settlement(customer_id, month="2026-04")
   ↓
直接走 4.3 的文件下发路径
```

### 4.5 接收客户付款凭证（入站文件）

```
客户 → [图片] 一张转账截图
   ↓
Channel Adapter 识别 msg_type=image
   ↓
Media Pipeline:
   1. 调企业微信 getmedia 拉原图 → 上传公司 OSS（加密 + 5 年留存）
   2. 计算 hash 去重
   3. 可选 OCR：识别金额、付款时间、收款账户末四位 → 结构化字段
   ↓
LLM/规则推断关联订单（按客户、金额、近期欠款匹配；多选时反问客户）
   ↓
Workflow Engine 创建工单 type=payment_verify, status=PENDING_VERIFY,
   assignee=finance, ref=order_ids[], oss_url
   ↓
Bot → 客户："✅ 已收到您的付款凭证（金额 ¥123,456，付款时间 5-28 10:32）
            财务正在核对，预计 2 小时内确认。如需备注请回复"
Bot → 财务（企业微信）：textcard «新付款凭证 客户张总 ¥123,456 请核对»

财务核对完成 → 在系统里点"已确认"或 Bot 内回复 /verify <workflow_id>
   ↓
Workflow → VERIFIED → Bot 主动通知客户："付款已确认"
```

### 4.6 提醒销售发货（跨用户消息转发 + 工单）

```
客户 → "我那个 SO-2026-0501 怎么还没发啊，催一下"
   ↓
LLM 意图=urge_shipment, 槽位 order_id=SO-2026-0501
   ↓
查 order → 找到对应销售 sales_zhang（来自绑定/订单字段）
   ↓
Workflow Engine 创建工单 type=urge_shipment, sla=2h,
   creator=customer, assignee=sales_zhang
   ↓
Bot → 销售小张（企业微信，文本卡片）：
  «客户催发货
   订单：SO-2026-0501
   客户：张总（武汉）
   留言：『怎么还没发啊』
   [点击查看订单] [一键回复客户]»
   ↓
Bot → 客户："✅ 已通知销售小张（13xxxxxxxxx）跟进
           SLA 2 小时内回复"
   ↓
销售在 Bot 里回复 /reply <workflow_id> "今天下午 3 点发车"
   或 在 ERP 里更新发货时间 → ERP webhook → Bot 自动回客户
   ↓
Bot → 客户："销售小张回复：今天下午 3 点发车"
   ↓ 工单关单
```

### 4.7 多意图切换示意（跨能力）

```
客户 → "螺纹钢 HRB400 25mm 200吨 武汉报价"          ← Topic#A 询价(异步)
Bot  → "已记录 INQ-001，30 分钟内回复"
客户 → "顺便看下我有多少欠款"                       ← Topic#B 查欠款(同步)
Bot  → "您当前应付：¥256,789，最近账期 5-31"
客户 → "上次那个炉号 24A0512 的材质书"              ← Topic#C 材质书
Bot  → [PDF]
... 30 分钟后 ...
Bot(主动) → "您的询价 INQ-001 报价：¥3,820/吨..."  ← Topic#A 由 webhook 唤醒
```

---

## 5. 模块设计（v3）

### 5.1 Callback Server / Channel Adapter / Identity & Binding / Permission Gate / Fast Path Router
保留 v2 设计。

### 5.2 Session State Manager
保留 v2 Topic + Task 状态机，**新增**：
- Topic 可以关联到 Workflow（如 Topic#A 询价 ↔ workflow#xxx）。
- Workflow 状态变化（如 webhook 进来）→ 自动唤醒对应 Topic → 推送主动消息。

### 5.3 LLM Orchestrator
保留 v2 双供应商路由 + 降级链 + 行业词典。**v3 强调**：

- LLM **只做意图识别、槽位抽取、文案组织**。
- **绝对不做以下事**：报价数字、欠款金额、装车吨位、订单号、付款金额——全部从工具返回直接渲染，prompt 强制约束 + 后置 DLP 校验。
- 对于"询价"，LLM 的职责是把客户的话拆成 `category/grade/spec/qty_ton/dest_city` 五个槽位，然后调用 `submit_inquiry` 创建工单——**不要试图自己报价**。

### 5.4 Tool Registry（v3 重写，对齐 8 项实际能力）

| 工具名 | 入参 | 用户角色 | 同步/异步 | 涉及文件 | 备注 |
|---|---|---|---|---|---|
| `query_held_orders` | customer_id?, date_range? | 客户/销售 | 同步 | 否 | 查留货订单 |
| `query_debt` | customer_id?, summary? | 客户/销售/财务 | 同步 | 否 | 查欠款；客户只看自己 |
| `submit_inquiry` | category, grade, spec, qty_ton, dest_city, delivery_date? | 客户/销售 | **异步工单** | 否 | 创建询价单 → 等业务侧 quote.ready 回调 |
| `query_inquiry_status` | inquiry_id? | 客户/销售 | 同步 | 否 | 查询自己发起的询价状态 |
| `query_loading_weight` | order_id / vehicle_no | 客户/销售 | 同步 | 可选磅单图 | 查装车重量 |
| `get_material_cert` | batch_no / order_id | 客户/销售 | 同步 + 发文件 | **PDF** | 要材质书 |
| `query_settlement` | customer_id?, month | 客户/销售/财务 | 同步 + 发文件 | **PDF** | 查结算单 |
| `acknowledge_settlement` | settlement_id, action: confirm/dispute, remark? | 客户 | 同步（写） | 否 | 客户对结算单的确认/异议 |
| `submit_payment_voucher` | image/pdf, hint_order_ids? | 客户 | **入站文件** | **图片/PDF** | 上传付款凭证；创建审核工单 |
| `urge_shipment` | order_id, message? | 客户 | **异步工单** | 否 | 提醒销售发货 → 创建工单转销售 |
| `list_my_workflows` | status? | 全员 | 同步 | 否 | 查我相关的工单 |
| `escalate_to_human` | reason | 全员 | 同步 | 否 | 转人工 |

> 写操作（`submit_inquiry` / `acknowledge_settlement` / `submit_payment_voucher` / `urge_shipment`）一律**二次确认 + 60s 反悔窗**。

### 5.5 Business API Adapter（ACL）
保留 v2 设计，**v3 强调**：
- 业务 API 不改，所以对询价、付款凭证、催发货等"工单类"操作，ACL 要做"API 调用 + 本地工单落库"两件事的原子化（事务/补偿）。
- 字段映射的扩展点：业务 API 里"留货订单"可能叫 `reservedOrder`，"装车重量"可能叫 `loadingWeight`，ACL 维护一份映射表。

### 5.6 Identity & Binding / Permission Gate
保留 v2 设计。**v3 数据级权限对应到具体工具**：

| 工具 | 客户 | 销售 | 财务 | 仓管 | 管理员 |
|---|---|---|---|---|---|
| query_held_orders | 仅自己 | managed_customers | × | × | ✅ |
| query_debt | 仅自己 | managed_customers | ✅ | × | ✅ |
| submit_inquiry | ✅ | 代客户提 | × | × | ✅ |
| query_inquiry_status | 仅自己 | managed | × | × | ✅ |
| query_loading_weight | 仅自己 | managed | × | ✅ | ✅ |
| get_material_cert | 仅自己 | managed | × | × | ✅ |
| query_settlement | 仅自己 | managed | ✅ | × | ✅ |
| acknowledge_settlement | 仅自己 | × | × | × | ✅ |
| submit_payment_voucher | ✅ | × | × | × | ✅ |
| urge_shipment | 仅自己 | × | × | × | ✅ |
| list_my_workflows | 自己 | 自己 | 自己 | 自己 | 全部 |

### 5.7 DLP & Audit
保留 v2 设计。**v3 新增**：
- 付款凭证 OCR 后的明细字段（卡号、账户名）入库前脱敏（只留末四位）。
- 材质书/结算单 PDF 在 OSS 中按客户 + 订单分目录，命中 ACL 才能下载。

### 5.8 WeCom API Client / Async Worker / Storage
保留 v2 设计。**v3 Storage 新增**：

| 表 | 说明 |
|---|---|
| `workflow` | 工单主表（询价/付款凭证/催发货/结算确认） |
| `workflow_event` | 工单状态变更流水 |
| `media_object` | 入站/出站文件元数据（OSS 路径、hash、TTL） |
| `biz_event_log` | 业务系统反向回调日志 |

### 5.9 Workflow Engine（v3 新增）

**模型**：

```
Workflow {
  workflow_id (UUID)
  type:      inquiry | payment_verify | urge_shipment | settlement_ack
  status:    PENDING | IN_PROGRESS | DONE | CANCELLED | TIMEOUT
  creator:   biz_user_id (客户)
  assignee:  biz_user_id (销售/财务)
  ref_ids:   { inquiry_id?, order_id?, settlement_id?, payment_id? }
  sla:       秒数
  created_at / updated_at / closed_at
  context:   JSON（槽位、原始消息、OSS 引用等）
}
```

**能力**：
- 状态机定义在配置里（YAML/JSON），便于业务方调整。
- SLA 超时 → 自动提醒 assignee → 二次超时升级 → 转管理员。
- 转派、关单、追加备注。
- 与 `Topic` 联动：状态变更触发主动消息。

### 5.10 Inbound Webhook（业务系统反向回调，v3 新增）

**接口约定**（业务系统调 Bot）：

```
POST /biz/event
Headers:
  X-Biz-Signature: HMAC-SHA256(secret, timestamp + body)
  X-Biz-Timestamp: 1716912000
Body:
{
  "event_id": "uuid",        // 幂等键
  "event_type": "quote.ready" | "settlement.created" |
                "payment.verified" | "shipment.scheduled" |
                "order.status_changed" | ...,
  "occurred_at": "2026-05-28T15:00:00+08:00",
  "data": { ... 事件特定字段 ... }
}
```

**处理**：
1. 验签 + 时间戳防重放（5 分钟）+ event_id 幂等。
2. 路由到对应 Event Handler。
3. 关联 Workflow → 更新状态 → 触发用户消息。
4. 失败重试由业务系统侧负责（推荐指数退避）；Bot 侧 200 表示已收到。

> **新增的依赖**：你们的业务系统需要在 8 个关键时刻调用这个 webhook：
> - `quote.ready`（销售在 ERP 里报完价）
> - `settlement.created`（月度结算单出账）
> - `payment.verified` / `payment.rejected`（财务核对付款凭证完成）
> - `shipment.scheduled` / `shipment.shipped` / `shipment.delivered`（发货状态变更）
> - 可选：`order.held` / `order.released`（留货变更）

如果业务系统暂时不能改造，可以用"轮询 + 状态对账"作为过渡：Bot 定时（每 30s/1min）拉取处于 PENDING 状态的工单对应的业务对象，发现变化再推。但**推荐还是上 webhook**，否则延迟和成本都会高。

### 5.11 Media Pipeline（文件/图片处理，v3 新增）

**入站（付款凭证、客户发图）**：

```
微信图片消息 →
  Channel Adapter 拿 media_id →
  调企业微信 media/get 拉原图 →
  存 OSS (kms 加密, customer/yyyy-mm/ 分区) →
  计算 hash 去重 →
  可选 OCR (阿里云/腾讯云通用OCR / 票据OCR) →
  规则/LLM 关联订单 →
  创建 Workflow →
  通知财务
```

**出站（材质书、结算单 PDF 发用户）**：

```
ACL 拿到业务 API 返回的 PDF URL →
  下载 →
  企业微信「上传临时素材」API → 拿 media_id (3 天有效) →
  缓存 (业务唯一键 → media_id, TTL 1 天) →
  发送 file 消息 →
  发送辅助 text 消息（说明）
```

**注意**：
- 微信客服支持发送图片/文件，但格式和大小有限制（图片 ≤ 2MB，文件 ≤ 20MB，PDF 支持）。
- 入站文件必须在 5 分钟内拉走（企业微信 media 有效期）。
- 单文件 OCR 异步处理，超时 30s。

---

## 6. 钢铁贸易话术与体验（v3）

- 行业语兼容（v2 沿用）："螺四"=HRB400 螺纹钢、"工"=工字钢、"H 钢"=H 型钢、"中板"=中厚板。
- **询价话术**：「品种 + 牌号 + 规格 + 数量 + 目的地 + 期望交货日」 → 不齐则反问 → 齐了才创单 → 创单后回 INQ 号 + SLA 预期。
- **报价回推**：包含价格 + 单位 + 含税含运说明 + 有效期 + 销售联系方式 + 引导（"如需下单/锁价请联系销售"）。
- **材质书/结算单**：双发——文字摘要 + PDF 附件。
- **付款凭证**：客户上传后即回执"已收到 + 金额 + 时间"，让客户安心；财务确认后再发"已确认"。
- **催发货**：把工单 ID + 预计回复时间告诉客户，避免客户反复催。
- **结算单确认**：客户必须显式回复"确认"或"有问题"，"有问题"自动转销售/财务跟进。

---

## 7. 安全与合规（钢铁贸易适配）

保留 v2 设计。**v3 新增**：

- **业务系统反向回调**：必须 HMAC 签名 + 时间戳 + event_id 幂等；建议双向 IP 白名单。
- **付款凭证文件**：OSS KMS 加密；明细字段（账户、卡号）脱敏；保留 5 年（财税要求）。
- **材质书 / 结算单 PDF**：临时签名 URL（5 分钟）；不允许直接在公网放裸 URL。
- **跨用户消息转发**（催发货）：销售看到的客户消息要走 DLP，避免把客户敏感信息（手机号、地址）泄漏给不该看的销售。

---

## 8. 可观测性

保留 v2 指标，**v3 新增**：
- `workflow_total{type,result}` / `workflow_sla_breach_total{type}`
- `inbound_webhook_total{event_type,result}` / 延迟
- `media_pipeline_total{direction,result}` / 大小分布
- 业务对账：每天对 `workflow` 表和业务系统对账，发现遗漏的事件（webhook 丢了）。

---

## 9. 部署拓扑（公网云）

保留 v2 拓扑，**v3 新增**：

```
                          ↘
   公司业务系统 / ERP ────▶  /biz/event Inbound Webhook
                          ↗   (走 VPC 内网 / HTTPS + HMAC)

   OSS (KMS 加密)          ↔  Media Pipeline
   阿里云/腾讯云 OCR        ↔  Media Pipeline (按需调用)
```

---

## 10. 里程碑（v3，按 8 项能力分批）

| 里程碑 | 交付物 |
|---|---|
| **M1 基础设施** | 域名/证书、企业微信后台建应用 + 客服、KMS 密钥、云资源、OSS |
| **M2 通信骨架** | 双通道回调验签/解密通过；token 缓存；自建应用能 echo |
| **M3 绑定 + 权限** | 一次性 token 绑定打通；命令级权限 Gate v1 |
| **M4 ACL + 同步查询能力** | ① 查留货订单 ② 查欠款 ④ 查装车重量 端到端跑通；缓存+审计 |
| **M5 LLM 编排（自然语言入口）** | DeepSeek + Qwen 双路由；Function Calling 路由 M4 的 3 个工具；降级链可用 |
| **M6 Session 状态机** | Topic + Task；跨天恢复；多意图切换 |
| **M7 Workflow Engine + Inbound Webhook** | 工单模型；/biz/event 接入；SLA 提醒 |
| **M8 询价 + 报价回推（③）** | 客户发起询价 → 创建工单 → 业务系统 quote.ready 回调 → 主动推回客户 |
| **M9 Media Pipeline + 文件下发（⑤）** | 出站 PDF：材质书；缓存 media_id |
| **M10 结算单（⑥）** | 出站 + 入站确认：业务系统 settlement.created → 推送 PDF → 客户回"确认"/"有问题" |
| **M11 付款凭证（⑦）** | 入站文件：客户上传 → OSS + OCR + 工单 → 通知财务 |
| **M12 提醒销售发货（⑧）** | 跨用户转发工单；SLA 提醒；销售在 Bot 内回复 |
| **M13 微信客服通道** | Kf 接入；外部客户可对话；48h 窗口策略 |
| **M14 DLP + 数据级权限完善** | 前/后置 DLP；数据级过滤；销售看客户消息时脱敏 |
| **M15 可观测 + 风控** | 指标/告警/限流/token 配额；业务对账定时任务 |
| **M16 灰度上线** | 内部销售试用 → 部分外部客户 → 全量；双 LLM A/B |

---

## 11. 风险与对策（v3 更新）

| 风险 | 影响 | 对策 |
|---|---|---|
| 业务系统不能改造，无法发 Inbound Webhook | 异步流程（询价/结算/付款审核）走不通 | 过渡方案：Bot 轮询业务对象状态对账；长期还是要推 webhook |
| 报价/欠款数字被 LLM "聊"出来错的 | 商业事故 | LLM 严禁出数字；后置 DLP 校验数字必须来自工具调用 |
| 微信客服 48h 窗口外推送报价失败 | 客户收不到 | 引导客户在群里发"查询报价"重新打开窗口；或退化 SMS |
| 入站图片识别错关联到别的订单 | 误冲账 | OCR 仅作建议；金额/订单不唯一时 Bot 必反问；财务最终人工确认才落账 |
| 文件 OSS 泄漏 | 客户机密泄漏 | KMS + 临时签名 URL + ACL 鉴权 + 日志审计 |
| 跨用户转发把客户敏感信息暴露给错的销售 | 合规问题 | 数据级权限 + DLP；销售只看 managed_customers |
| 工单堆积无人处理 | 客户体验差 | SLA 超时升级 + 管理员看板 |
| LLM 供应商故障 | 自然语言入口不可用 | 双供应商热切换；关键字命令永远可用 |
| 客户绑定身份被冒用 | 数据泄漏 | 一次性 token 短 TTL + 解绑需要双因素 |
| 业务 API 不稳定 | 用户体验差 | ACL 重试 + 熔断 + 缓存 + "暂不可用 + 转人工" |

---

## 12. 后续演进

- **OCR + LLM 抽取**：结算单、磅单、合同照片自动结构化。
- **主动智能提醒**：账期临近、留货到期、库存到货提醒。
- **销售 Copilot**：今日待跟进客户、待处理工单、销售业绩看板。
- **RAG 知识库**：钢材标准、公司政策、常见问题问答。
- **小程序辅助**：复杂表单（如多 SKU 询价）用小程序填，文字对话做入口。
- **私有化 LLM**：如对数据出域顾虑增强，切换到本地 Qwen / DeepSeek 蒸馏版。

---

## 13. 版本演进对比

| 维度 | v1 | v2 | **v3** |
|---|---|---|---|
| 用户范围 | 通用 | 内外双通道 | 同 v2 |
| 意图识别 | 关键字 | LLM 双供应商 | 同 v2 |
| 多轮 | Redis 最近 N 条 | Topic + Task 状态机 | 同 v2 |
| 绑定 | 草案 | 三方式 + 数据级权限 | 同 v2 |
| 业务 API | 假设可改造 | ACL 适配 | 同 v2 + **额外 Inbound Webhook 约定** |
| 行业 | 通用 | 钢铁贸易（假想能力） | **钢铁贸易（对齐 8 项实际能力）** |
| 安全/DLP | 基础 | 强化 | 同 v2 + 文件 KMS + 临时 URL |
| 部署 | 通用 | 云原生 | 同 v2 + OSS + OCR |
| 写操作 | — | 二次确认 + 反悔窗 | 同 v2 |
| **工单模型** | — | — | **新增 Workflow Engine** |
| **业务系统反向通知** | — | — | **新增 Inbound Webhook** |
| **文件/图片消息** | — | — | **新增 Media Pipeline** |
| **跨用户消息转发** | — | — | **新增（催发货/付款凭证）** |
| **能力清单** | 假想 | 假想丰富版 | **严格对齐 8 项实际能力** |

---

## 14. 待需求方确认（v3）

1. **业务系统能否配合开发 Inbound Webhook**？这是 `询价回推 / 结算单推送 / 付款核对结果 / 发货状态` 等异步流程的关键。如果不能，需要走轮询过渡方案，延迟和系统压力都会变大。
2. **业务 API 是否支持以下查询**？（用于 ACL 适配）
   - 按客户 + 时间范围查留货订单列表
   - 按客户查欠款汇总/明细
   - 创建询价单（写）
   - 按订单号/车牌查装车重量（含磅单图 URL）
   - 按炉号/订单号查材质书（PDF URL）
   - 按客户 + 月份查结算单（PDF URL）
   - 提交付款凭证（写，含图片 URL + 金额 + 时间 + 关联订单）
   - 提交发货催办（写）
3. **付款凭证是否需要 OCR 自动识别**？还是只需要存档转发？OCR 会增加 LLM/OCR 服务成本。
4. **销售在 Bot 里回复客户**，是希望走"在 Bot 里发 /reply"还是"在 ERP 里点确认"？或两者都支持？
5. **询价 SLA**（销售首次报价时限）、**催发货 SLA**、**付款核对 SLA**：分别是多少？
6. **客户绑定流程**：你们希望是「销售在系统里生成 token 给客户」、还是「客户自助输入手机号收验证码」、还是两者都要？
7. **是否需要群聊场景**（销售拉客户进企业微信群，机器人在群里待命）？还是只一对一私聊？
