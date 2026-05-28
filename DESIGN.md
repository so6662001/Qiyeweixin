# 企业微信机器人 — 设计文档（v4）

> 状态：设计阶段（尚未开发）
> 行业：**钢铁贸易**
> 目标：搭建一个企业微信智能机器人，对接公司已有的 8 项后端能力（查留货订单 / 查欠款 / **询价（4 种输入：文字 / 图片 / Excel / PDF）** / 查装车重量 / 要材质书 / 接收或查结算单 / 接收付款凭证 / 提醒销售发货），覆盖内部员工和外部微信客户，接入 LLM（DeepSeek / 通义千问，**含 VL 视觉模型**）做自然语言理解、多模态解析与多轮对话。

---

## 0. 需求方已确认的关键约束

| # | 问题 | 答复 | 对设计的影响 |
|---|---|---|---|
| 1 | 用户范围 | 内部 + 外部微信用户 | 双通道：自建应用 + 微信客服 |
| 2 | 业务 API | 已具备，**不可改造** | Bot 侧 ACL 适配 |
| 3 | 是否绑定账号 | 必须绑定 | 强绑定 + 数据级权限 |
| 4 | 多轮对话 | 跨天，多意图并存 | Topic + Task 状态机 |
| 5 | 部署 | 公网云，域名已备案 | 云原生 |
| 6 | 行业 | 钢铁贸易 | 强权限 + DLP + 审计 |
| 7 | LLM | DeepSeek + 通义千问 | LLM Orchestrator + 双供应商路由 |
| 8 | 实际后端能力 | 8 项 | 技能包对齐 + 工单引擎 + 反向回调 + 文件管道 |
| 9 | **询价输入形态（v4 新增）** | **文字 / 图片 / Excel / PDF 四类** | **新增多模态询价解析器**；询价 → **批量工单**；**强制结构化回显 + 人工确认**环节；引入 **VL 模型**（Qwen-VL / DeepSeek-VL） |

---

## 1. 需求与目标

### 1.1 8 项能力的形态分类

| # | 能力 | 输入形态 | 是否同步 | 反向回调 | 涉及文件 | 跨用户 |
|---|---|---|---|---|---|---|
| 1 | 查留货订单 | 文字 | 是 | 否 | 否 | 否 |
| 2 | 查欠款 | 文字 | 是 | 否 | 否 | 否 |
| 3 | **询价** | **文字/图片/Excel/PDF** | **否（异步工单）** | **是**（quote.ready） | 入站 + 可能出站报价单 | 客户↔销售 |
| 4 | 查装车重量 | 文字 | 是 | 否 | 可选磅单图 | 否 |
| 5 | 要材质书 | 文字 | 是 | 否 | **PDF** | 否 |
| 6 | 接收/查结算单 | 文字 / 业务侧推 | 双向 | **是** | **PDF** | 客户↔财务 |
| 7 | 接收付款凭证 | 图片 / PDF | 是（写） | 可选 | **图片/PDF** | 客户→财务 |
| 8 | 提醒销售发货 | 文字 | 否（工单） | 可选 | 否 | **客户→销售** |

### 1.2 询价 4 种形态的具体场景（v4 重点）

| 形态 | 真实场景 | 解析难点 |
|---|---|---|
| **文字** | 客户在微信里直接打字："螺纹钢 HRB400 25mm 200吨 武汉，给个价" | 行业黑话、规格简写、多 SKU 用顿号或换行混排 |
| **图片** | 客户把手写询价单、采购单截图、群聊截图发过来 | 手写字、印章遮挡、倾斜、模糊、表格识别 |
| **Excel** | 客户用自己 ERP 导出或自制的询价表，往往**几十到上百行 SKU** | 表头千奇百怪（"型号"/"规格"/"牌号"/"材质"混用）、合并单元格、多 sheet、单位不统一（吨/根/件） |
| **PDF** | 客户公司正式发出的询价函（采购计划） | 文本 PDF vs 扫描 PDF；表格跨页；公章/水印干扰 |

→ **同一份询价可能产生 1~N 个 SKU 报价项**，所以询价工单是**一对多结构**。

### 1.3 用户与场景

| 角色 | 用到的能力 |
|---|---|
| 客户 | 查留货订单、查欠款、**发起询价（4 种输入）**、查装车重量、要材质书、查/确认结算单、上传付款凭证、催发货 |
| 销售 | 查客户留货/欠款/装车重量，**收到询价工单（含解析结果）去 ERP 报价**，收到催发货工单去处理 |
| 财务 | 查欠款汇总、查结算单、查/审付款凭证、对账 |
| 管理员 | 全部 + 工单监控 + 配置 + 解析准确率监控 |

### 1.4 非功能需求

| 项 | 目标 |
|---|---|
| 回调响应延迟 | < 1 s（硬限 < 5 s） |
| 同步查询整体延迟 | P95 < 3 s |
| LLM 文字回复延迟 | P50 < 3 s, P95 < 8 s |
| **询价多模态解析延迟** | **图片 ≤ 8 s；Excel ≤ 5 s（≤ 50 行）；PDF ≤ 10 s** |
| **询价解析准确率（结构化字段）** | **≥ 90%**，剩余通过人工确认补齐 |
| 异步工单 SLA | 询价首次报价 30 min；催发货 1h |
| 文件上传单文件 | ≤ 20 MB |
| 可用性 | 99.5%+ |
| 安全 | HTTPS + 签名 + AES + KMS + DLP |
| 数据保留 | 消息 180d；工单 1y；财务文件 5y |

---

## 2. 通道选型

| 形态 | 接收 | 发送 | 用户 | 选用 |
|---|---|---|---|---|
| 群机器人 Webhook | ❌ | 推到群 | 群成员 | 仅做内部告警 |
| **自建应用** | ✅ | ✅ | 内部员工 | ✅ |
| **微信客服 (Kf)** | ✅ | ✅ | 普通微信用户 | ✅ |

> 严禁走个人微信号协议。外部客户只走微信客服。

---

## 3. 总体架构（v4）

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
   │  Callback → Channel Adapter → Identity & Binding → Permission   │
   │           → Fast Path Router                                     │
   │              ↓ (未命中)         ↓ (命中关键字)                   │
   │   Session State Manager        Command Handler                   │
   │              ↓                                                   │
   │   LLM Orchestrator (DeepSeek / Qwen 文本 + VL 双路由)            │
   │              ↓                                                   │
   │   Tool Registry (Function Calling)                               │
   │              ↓                                                   │
   │   Business API Adapter (ACL) ──▶ 现有业务 API                    │
   │              ↓                                                   │
   │   Reply Composer + DLP ──▶ WeCom API Client                      │
   │                                                                  │
   │  ──────────────────── v3/v4 新增/强化模块 ────────────────────   │
   │                                                                  │
   │  ① Workflow Engine （工单/任务状态机）                            │
   │      询价（含批量 SKU）/ 催发货 / 付款凭证 / 结算单确认           │
   │                                                                  │
   │  ② Inbound Webhook  POST /biz/event                              │
   │      quote.ready / settlement.created /                          │
   │      payment.verified / shipment.*                               │
   │                                                                  │
   │  ③ Media Pipeline （文件/图片）                                   │
   │      入站：付款凭证、询价图片/Excel/PDF                          │
   │      出站：材质书 / 结算单 / 报价回执 PDF                        │
   │                                                                  │
   │  ④ Inquiry Parser  ★v4 新增★                                    │
   │      文字 → LLM 抽槽                                             │
   │      图片 → Vision-OCR + VL → 表格结构化                         │
   │      Excel → openpyxl + LLM 列头映射 → 单位归一                  │
   │      PDF → 文本抽取 / 扫描转图走 VL                              │
   │      → 统一产出 InquiryItem[] → 回显客户确认 → 提交              │
   └────────────────────────────────────────────────────────────────┘

   ┌──────────┐  ┌──────────┐  ┌──────────────┐  ┌────────────────┐
   │  Redis   │  │ Postgres │  │ Async Worker │  │ Audit / DLP    │
   │ token/ctx│  │ 绑定/会话│  │ LLM/解析/    │  │ ES / SLS / OSS │
   │ 限流/锁  │  │ 工单/审计│  │ 工单/推送    │  │ Prompt/解析留存│
   └──────────┘  └──────────┘  └──────────────┘  └────────────────┘
```

---

## 4. 关键流程（v4，按 8 项能力）

### 4.1 同步查询类（① 留货订单 / ② 欠款 / ④ 装车重量）

```
客户 → "查下我那张昨天的留货订单"
   ↓ 回调 → 绑定/权限 → Session 取/建 Topic
   ↓
LLM 识别意图 = query_held_orders；槽位补齐
   ↓
Function Call: query_held_orders(customer_id, date_range="yesterday")
   ↓
ACL → 业务 API → JSON → Reply Composer 渲染 → DLP → 发送
```

### 4.2 询价（v4 重写，含 4 种输入）

#### 4.2.1 共同骨架

```
═══ 阶段 A：客户输入 ═══════════════════════════════════════
客户发：文字 / 图片 / Excel / PDF
   ↓
Channel Adapter 识别 msg_type
   ↓
Inquiry Intent Detector
   - 文字：LLM 判定是否询价语义 ("报价/给个价/能不能做")
   - 文件：先看文件名+前几个 token，若不能判，提示用户："这是询价单吗？回复『是』继续"
   ↓
判定为询价 → 进入 Inquiry Parser

═══ 阶段 B：多模态解析（4 条分支）══════════════════════════
                                            
            ┌─────────── 文字分支 ─────────┐
            │  LLM (文本) → InquiryItem[]   │
            └───────────────────────────────┘
            ┌─────────── 图片分支 ─────────┐
            │  OCR + Vision LLM (Qwen-VL/  │
            │  DeepSeek-VL)                │
            │  → 表格行 → InquiryItem[]    │
            └───────────────────────────────┘
            ┌─────────── Excel 分支 ───────┐
            │  openpyxl 解析单元格         │
            │  → LLM 列头映射 (规格/数量…) │
            │  → 单位换算 + 校验           │
            │  → InquiryItem[]             │
            └───────────────────────────────┘
            ┌─────────── PDF 分支 ─────────┐
            │  文本 PDF：pdfplumber 抽文本  │
            │  扫描 PDF：rasterize → VL    │
            │  → 表格结构化                │
            │  → InquiryItem[]             │
            └───────────────────────────────┘
                          ↓
            统一 InquiryDraft (含 N 个 item)

═══ 阶段 C：回显 & 确认 ════════════════════════════════════
Bot → 客户 (Markdown 表格)：
"📋 已解析到 12 条询价（置信度 87%），请确认：

| # | 品种 | 牌号 | 规格 | 数量 | 目的地 | 状态 |
|---|---|---|---|---|---|---|
| 1 | 螺纹钢 | HRB400 | 25mm | 200t | 武汉 | ✅ |
| 2 | 螺纹钢 | HRB400 | 22mm | 150t | 武汉 | ✅ |
| 3 | 螺纹钢 | ?      | 18mm | 100t | 武汉 | ⚠️ 牌号待确认 |
| ... | | | | | | |

回复『确认』提交；
回复『第3行牌号 HRB400』补齐；
回复『删除第7行』移除；
回复『取消』丢弃。"

═══ 阶段 D：提交工单 ══════════════════════════════════════
客户回『确认』
   ↓
Function Call: submit_inquiry(items=[InquiryItem×N], attachments=[原始文件])
   ↓
ACL → 业务 API 批量创建 → 拿询价单号 INQ-20260528-001
   ↓
Workflow Engine 创建工单:
   type=inquiry, items_count=12, status=PENDING_QUOTE,
   sla=30min, assignee=对应销售
   ↓
Bot → 客户："✅ 已提交询价 INQ-001（12 条），销售小张正在报价，30 分钟内回复"
Bot → 销售小张：textcard «新询价 12 条 客户张总，请处理»

═══ 阶段 E：销售在 ERP 报价 → 反向回调 ═════════════════════
销售在 ERP 里逐条/批量报价 → ERP 调 POST /biz/event
   ↓
{ "event_type":"quote.ready",
  "inquiry_id":"INQ-001",
  "quotes":[{item_id, price, unit, valid_until, remark}, …] }
   ↓
Inbound Webhook → Workflow → QUOTED
   ↓
Bot → 客户（主动消息，48h 窗口内）：
   "✅ INQ-001 报价已出（12 条）
    | # | 品种规格 | 单价 | 有效期 |
    | 1 | 螺纹钢 HRB400 25mm | ¥3,820 | 5-29 18:00 |
    | … |
    
    如需锁价/下单请联系销售小张 13xxxxxxxxx"
    （附 PDF 报价单 ← 可选，从 ERP 拉 PDF 回推）
```

#### 4.2.2 边界情况

| 情况 | 处理 |
|---|---|
| Excel 100+ 行 | 分批解析；回显时分页（每页 20 行）；客户可"显示下一页/全部确认/逐页确认" |
| 解析置信度 < 70% | 不回显表格，直接说"识别不太清晰，方便人工录入吗？回复 1 → 转销售；回复 2 → 再发清晰一点的文件" |
| 文件含多 sheet | 反问客户："发现 3 个 sheet，要询哪个？" |
| 客户连发多张图（分页拍） | 等 60s 合并解析，或客户回"发完了"触发解析 |
| 同一询价反复改 | 保留 Draft 30 min，期间客户可继续追加/修改，不创建新工单 |

### 4.3 要材质书

```
客户 → "炉号 24A0512 的材质书发我"
   ↓ LLM 意图 = get_material_cert
   ↓ Function Call → ACL → PDF URL
   ↓ Media Pipeline 下载 → 上传企微 media → 缓存
   ↓ WeCom 发 file + text 摘要 → 审计
```

### 4.4 接收/查结算单

```
方向 A：ERP 推
  POST /biz/event { event:"settlement.created", customer_id, month, pdf_url, amount }
    → Workflow 创建 type=settlement_ack, sla=72h
    → Media Pipeline 拉 PDF → 企微 media
    → Bot 发客户：摘要 + PDF + "请回复『确认』/『有问题』"

方向 B：客户查
  客户 → "上个月结算单"
    → query_settlement(customer_id, month="2026-04")
    → 走 4.3 文件下发
```

### 4.5 接收客户付款凭证

```
客户 → [图片] 转账截图
   ↓ msg_type=image
   ↓ Media Pipeline 拉原图 → OSS (KMS) → hash 去重
   ↓ 可选 OCR：金额/时间/账户末四位
   ↓ 关联订单（规则+LLM；不唯一就反问）
   ↓ Workflow 创建 type=payment_verify, assignee=finance
   ↓ Bot → 客户回执 "已收到，¥xxx，财务核对中"
   ↓ Bot → 财务 textcard 提醒
   ↓ 财务确认 → /biz/event payment.verified → Bot 通知客户
```

### 4.6 提醒销售发货

```
客户 → "SO-2026-0501 怎么还没发"
   ↓ 意图=urge_shipment, slot=order_id
   ↓ 查 order 找到 sales_zhang
   ↓ Workflow 创建 type=urge_shipment, sla=2h
   ↓ Bot → 销售小张 textcard «客户催发货»
   ↓ Bot → 客户 "已通知销售小张，SLA 2h"
   ↓ 销售在 Bot 内 /reply 或 ERP 更新 → Bot 回客户 → 关单
```

### 4.7 多意图切换

```
客户 → [Excel 询价 50 行]              ← Topic#A 询价（解析中）
Bot  → "正在解析，约 5 秒…"
客户 → "对了，先看看我有多少欠款"      ← Topic#B 查欠款（同步）
Bot  → "应付 ¥256,789"
Bot  → "（询价已解析完）请确认 12 条…"  ← Topic#A 解析完唤醒
...
Bot(主动) → "您的询价 INQ-001 报价已出..."  ← Topic#A 由 webhook 唤醒
```

---

## 5. 模块设计（v4）

### 5.1 Callback Server / Channel Adapter / Identity & Binding / Permission Gate / Fast Path Router
保留 v3。Channel Adapter 在 v4 强化：识别并分流图片/Excel/PDF 消息。

### 5.2 Session State Manager
保留 v3。**v4 新增**：Topic 可挂载 `inquiry_draft_id`，30 分钟内同一询价的多次输入合并。

### 5.3 LLM Orchestrator（v4 增加 VL 模型）

**模型路由（v4 扩展）**：

| 任务 | 首选 | 备选 | 备注 |
|---|---|---|---|
| 意图分类 | Qwen-Turbo | DeepSeek-V3 | 便宜快 |
| 文本槽位抽取 | DeepSeek-V3 | Qwen-Plus | function calling |
| Excel 列头映射 | Qwen-Plus | DeepSeek-V3 | 处理表头变体 |
| **图片询价表识别** | **Qwen-VL-Max** | **DeepSeek-VL2** | 中文表格 OCR 强 |
| **扫描 PDF** | **Qwen-VL-Max** | DeepSeek-VL2 | 同上 |
| 复杂推理 | DeepSeek-R1 | Qwen-Max | 多约束询价归类 |
| 兜底闲聊 | Qwen-Turbo | DeepSeek-V3 | 便宜 |

**强约束**：LLM 严禁出报价/欠款/重量/订单号数字；这些字段必须来自工具返回。后置 DLP 校验。

### 5.4 Tool Registry（v4 更新）

| 工具 | 入参 | 角色 | 同/异步 | 文件 | 备注 |
|---|---|---|---|---|---|
| `query_held_orders` | customer_id?, date_range? | 客户/销售 | 同步 | × | |
| `query_debt` | customer_id?, summary? | 客户/销售/财务 | 同步 | × | |
| **`parse_inquiry`** | content_type: text/image/excel/pdf, payload/file_ref | 客户/销售 | 同步 | 入站 | **v4 新增**：返回 InquiryDraft（不直接提交） |
| **`submit_inquiry`** | draft_id 或 items[InquiryItem] | 客户/销售 | **异步工单** | × | 提交（含批量） |
| `query_inquiry_status` | inquiry_id? | 客户/销售 | 同步 | × | |
| `query_loading_weight` | order_id / vehicle_no | 客户/销售 | 同步 | 可选 | |
| `get_material_cert` | batch_no / order_id | 客户/销售 | 同步 + 发件 | PDF | |
| `query_settlement` | customer_id?, month | 客户/销售/财务 | 同步 + 发件 | PDF | |
| `acknowledge_settlement` | settlement_id, action, remark? | 客户 | 同步（写） | × | 二次确认 |
| `submit_payment_voucher` | image/pdf, hint_order_ids? | 客户 | 入站文件 | 图/PDF | |
| `urge_shipment` | order_id, message? | 客户 | 异步工单 | × | |
| `list_my_workflows` | status? | 全员 | 同步 | × | |
| `escalate_to_human` | reason | 全员 | 同步 | × | |

> **关键拆分**：v4 把 v3 的 `submit_inquiry` 拆成 `parse_inquiry` → 客户确认 → `submit_inquiry`，防止解析错误直接进 ERP。

**InquiryItem 结构**：
```json
{
  "row_no": 1,
  "category": "螺纹钢",
  "grade": "HRB400",
  "spec": "25mm",
  "qty": 200, "qty_unit": "吨",
  "dest_city": "武汉",
  "delivery_date": "2026-06-05",
  "remark": "含税含运",
  "confidence": 0.92,
  "raw_excerpt": "原始片段，用于回显追溯",
  "missing_fields": []
}
```

### 5.5 Business API Adapter（ACL）
保留 v3。**v4 新增**：`submit_inquiry` 支持批量；`parse_inquiry` 调 Inquiry Parser 而非业务 API。

### 5.6 Identity & Binding / Permission Gate
保留 v3。

### 5.7 DLP & Audit
保留 v3。**v4 新增**：解析原始文件 + 解析结果 + 客户确认前后差异都要审计（事后追责"是 Bot 解析错了，还是客户确认错了"）。

### 5.8 WeCom API Client / Async Worker / Storage
保留 v3。**v4 Storage 新增**：

| 表 | 说明 |
|---|---|
| `inquiry_draft` | 询价草稿（30 分钟 TTL，client 端可继续追加） |
| `inquiry_item` | 解析出的逐项 SKU |
| `parse_log` | 解析过程、模型、置信度、耗时、用户修正记录（用于后续质量评估） |

### 5.9 Workflow Engine
保留 v3。**v4 调整**：`inquiry` 工单包含 `items_count` 和 `quoted_count`，可部分报价部分待报。

### 5.10 Inbound Webhook
保留 v3。**v4 调整**：`quote.ready` 事件支持批量报价（`quotes: [{item_id, price, …}]`）。

### 5.11 Media Pipeline
保留 v3 入站/出站基础流程。**v4 新增分支**：识别到询价相关文件后，**不直接走付款凭证流程**，而是路由到 Inquiry Parser。判定逻辑：

1. 用户当前 Topic 意图是询价 → 进 Inquiry Parser
2. 文件名含"询价/报价/采购/PO/RFQ/inquiry/quote" → 进 Inquiry Parser
3. 文件是图片且金额特征明显（金额、付款时间、银行 logo）→ 付款凭证
4. 无法判定 → Bot 反问："这是『询价单』还是『付款凭证』？"

### 5.12 Inquiry Parser（v4 核心新增模块）

#### 5.12.1 模块职责
把 4 种异构输入归一为标准 `InquiryDraft { items: InquiryItem[] }`，并打置信度。

#### 5.12.2 文字分支
- 调 DeepSeek-V3 / Qwen-Plus 的 function calling，输入文本 + 行业词典 system prompt → 输出 `items[]`。
- 单条/多条都支持（用户可能一句话报多种）。

#### 5.12.3 图片分支
- Step 1：图片预处理（去黑边、旋转校正、增亮）。
- Step 2：调 VL 模型（Qwen-VL-Max 首选）直接做"图表→JSON"。
- Step 3：必要时叠加专业 OCR（PaddleOCR / 阿里云通用OCR / 腾讯云表格OCR）做二次校验，比对数字字段的一致性。
- Step 4：失败/低置信 → 把整张图转给销售 + 提示客户人工。

#### 5.12.4 Excel 分支
- Step 1：`openpyxl` 读 sheet（多 sheet → 反问）。
- Step 2：取前 10 行作为"列头候选"喂 LLM，让 LLM 输出列映射（`{品种:B, 规格:C, 数量:D, 单位:E, 目的地:F, 牌号:G}`）。
- Step 3：按映射遍历所有数据行，应用单位换算（吨/千克/根/件 → 吨）、规格归一（`Φ25` / `25mm` / `D25` → `25mm`）。
- Step 4：跳过空行/小计/合计行（规则识别）。
- Step 5：每行打置信度；缺失字段标记 `missing_fields`。

#### 5.12.5 PDF 分支
- Step 1：`PyMuPDF` 试抽文本。
- Step 2：
  - 抽到结构化文本（含表格线）→ 走文字+表格规则解析
  - 抽不到（扫描件） → rasterize 转 PNG → 走图片分支（VL）
- Step 3：跨页表合并（同列宽 + 表头一致）。

#### 5.12.6 共性能力
- **行业词典**注入 prompt：螺纹钢/盘螺/线材/热轧卷/冷轧卷/中厚板/H 型钢/工字钢/角钢/槽钢/无缝管/焊管…
- **规格归一**：`Φ` `D` `直径` → 统一为 mm；`6.0×1500×C` 卷板规格保留原始。
- **数量归一**：单位换算到吨；按"理论重量"反算的允许标注 `qty_calc=true`。
- **置信度**：字段级置信度（VL/LLM 概率） + 规则校验加分（如规格落在合理钢标范围）。
- **回显模板**：Markdown 表格 + "缺失/异常"高亮，每行附 `raw_excerpt` 让客户能对照原文。
- **客户修正闭环**：客户每一次修正落 `parse_log`，用于评估和后续微调。

#### 5.12.7 失败降级链
```
VL 模型超时 / 失败
  → 切备选 VL
  → 切纯 OCR + 规则解析
  → 解析失败 → 提示客户"我没看清这份文件，方便用文字简单说一下/或者发清楚一点吗？"
                + 一键转销售人工
```

#### 5.12.8 反作弊与边界
- 单客户每小时解析次数限流（10 次 / 小时）。
- 单文件大小 ≤ 20 MB；Excel 行数 ≤ 500；PDF 页数 ≤ 30。超出走人工。
- 不接受可执行文件、压缩包；提示客户解压后再发。

---

## 6. 钢铁贸易话术与体验（v4）

- **询价回执统一格式**：
  - 已解析数 / 待确认数 / 异常数
  - 表格分页
  - 操作菜单（确认 / 修改第 X 行 / 删除第 X 行 / 全部取消 / 转人工）
- **报价回推统一格式**：
  - 包含品种规格 + 单价 + 含税含运 + 有效期 + 销售联系方式
  - 大于 N 行的报价以 PDF 报价单形式发送（出站 Media Pipeline）
- **多模态友好提示**：
  - 客户发图后立即回 "正在识别…"（避免客户连发）
  - 解析慢于 5s 推中间状态
- **结算单/材质书**：文字摘要 + PDF 附件。
- **付款凭证**：即时回执 + 财务确认后再通知。
- **催发货**：工单 ID + SLA 透明。
- **行业语兼容**：螺四=HRB400 螺纹钢，工=工字钢，中板=中厚板。

---

## 7. 安全与合规

保留 v3 全部要求。**v4 新增**：

- 入站文件（询价/付款凭证）一律 KMS 加密存 OSS，按客户+日期分区。
- 询价文件保留 1 年（业务对账）；付款凭证保留 5 年（财税）。
- LLM/VL 调用：prompt 不带成本/毛利；解析结果出门前 DLP 二次扫敏感字段。
- 客户上传的文件，绝不能被其他客户检索到。
- 解析过程的截图/中间产物只在内部审计可见，不回显给客户。

---

## 8. 可观测性

保留 v3。**v4 新增**：
- `inquiry_parse_total{content_type,result}`
- `inquiry_parse_latency_seconds{content_type}`
- `inquiry_parse_confidence{content_type}` 分布
- `inquiry_correction_rate`（客户人工修正比例，用于评估解析准确率）
- `vl_call_total / vl_tokens_total / vl_cost`

---

## 9. 部署拓扑（公网云）

保留 v3。**v4 新增组件**：
- VL 模型调用：DashScope（Qwen-VL）/ DeepSeek API → NAT 出公网
- OCR 服务：阿里云 / 腾讯云 OCR（按需）
- PDF / Excel 解析依赖：服务镜像内置 PyMuPDF / openpyxl / pandas（无网络依赖）

---

## 10. 里程碑（v4，新增多模态询价拆分）

| 里程碑 | 交付物 |
|---|---|
| M1 基础设施 | 域名/证书、企业微信后台、KMS、云资源、OSS |
| M2 通信骨架 | 双通道回调验签/解密；token；echo |
| M3 绑定 + 权限 | 一次性 token；命令级 Gate |
| M4 ACL + 同步查询 | ① 留货 ② 欠款 ④ 装车重量 端到端 |
| M5 LLM 编排（文本） | DeepSeek + Qwen 双路由；function calling 跑通 M4 工具 |
| M6 Session 状态机 | Topic + Task；跨天恢复；多意图切换 |
| M7 Workflow + Inbound Webhook | 工单引擎；/biz/event 接入 |
| **M8 询价（文字）** | parse_inquiry 文字分支 → 回显确认 → submit_inquiry → quote.ready 回推 |
| **M9 询价（图片）** | VL 模型接入；图片询价表识别；置信度与回显 |
| **M10 询价（Excel）** | openpyxl + 列头映射；批量 SKU；分页确认 |
| **M11 询价（PDF）** | 文本 PDF + 扫描 PDF 双分支 |
| M12 Media Pipeline + 文件下发（⑤） | 出站材质书 PDF |
| M13 结算单（⑥） | 业务系统推 + 客户回确认 |
| M14 付款凭证（⑦） | 入站图片/PDF + 可选 OCR + 工单转财务 |
| M15 提醒发货（⑧） | 跨用户工单；Bot 内 /reply 或 ERP 回写 |
| M16 微信客服通道 | Kf 接入；外部客户对话；48h 窗口 |
| M17 DLP + 数据级权限完善 | 前后置 DLP；销售看客户消息脱敏 |
| M18 可观测 + 风控 + 对账 | 指标/告警/限流/token 配额；解析准确率监控 |
| M19 灰度上线 | 内部销售 → 部分外部客户 → 全量 |

---

## 11. 风险与对策（v4 更新）

| 风险 | 影响 | 对策 |
|---|---|---|
| 多模态解析准确率不够 | 错单/客户体验差 | 强制回显 + 人工确认；置信度低自动转销售；持续收集 correction 做评估 |
| Excel 表头千奇百怪 | 列映射错位 | LLM 列头映射 + 多版本测试集；客户可手动修正映射 |
| 扫描 PDF 模糊 | 解析失败 | VL + 专业 OCR 双校验；失败转人工 |
| VL 模型费用 | 成本上升 | 缓存（按 file_hash）；图片预过滤（明显非询价直接走付款凭证）；按客户配额 |
| 客户连发多张图被反复触发解析 | 浪费/不一致 | 60s 合并窗口 + "发完了"触发 |
| 业务系统不能开发 Webhook | 异步流程走不通 | 过渡轮询；长期推 webhook |
| LLM/VL 编造数字 | 商业事故 | 严禁出数字；后置 DLP 校验 |
| 微信客服 48h 窗口外推送报价失败 | 客户收不到 | 引导客户发起会话；退化 SMS |
| 入站文件含病毒/恶意宏 | 系统风险 | OSS 上传前查杀；Excel 关闭宏；PDF 解析隔离 |
| 客户机密数据外泄到 LLM 厂商 | 合规 | prompt 脱敏；如客户要求可切私有化 |
| 解析中间产物泄漏 | 合规 | 中间产物只内部审计可见；客户不可下载 |
| 跨用户转发暴露敏感信息 | 合规 | DLP；销售只看 managed_customers |

---

## 12. 后续演进

- **询价 → 报价 PDF 出 → OCR 客户回执** 闭环。
- **OCR + LLM 微调**：拿真实询价语料微调 Qwen-VL，提升钢材表格识别率。
- **客户专属模板**：识别熟客户的固定表格格式，跳过列头映射直走规则解析。
- **主动智能**：账期临近提醒、留货到期提醒、报价跌价提醒。
- **小程序辅助**：超大 Excel（500+ 行）引导走小程序专用上传通道。
- **RAG 知识库**：钢材标准、政策、FAQ。
- **私有化 LLM/VL**：客户对数据出域顾虑增强时切换。

---

## 13. 版本演进对比

| 维度 | v1 | v2 | v3 | **v4** |
|---|---|---|---|---|
| 用户范围 | 通用 | 内外双通道 | 同 | 同 |
| 意图识别 | 关键字 | LLM 双供应商 | 同 | 同 + **VL 多模态** |
| 多轮 | Redis N 条 | Topic/Task | 同 | 同 |
| 绑定 | 草案 | 三方式 + 数据级权限 | 同 | 同 |
| 业务 API | 假设可改 | ACL | + Inbound Webhook | 同 |
| 行业能力 | 通用 | 假想 | **8 项实际能力** | 同 + **询价 4 形态** |
| 工单 | — | — | Workflow Engine | + **批量 SKU + 草稿** |
| 文件 | — | — | Media Pipeline | + **询价多模态分支** |
| **询价输入** | 文字 | 文字 | 文字 | **文字/图片/Excel/PDF** |
| **询价提交流** | 一步 | 一步 | 一步 | **解析→回显→确认→提交** |
| **VL 模型** | — | — | — | **Qwen-VL / DeepSeek-VL** |
| **解析审计** | — | — | — | **新增 parse_log** |

---

## 14. 待需求方确认

1. **业务系统能否配合开发 Inbound Webhook**？关键。
2. **业务 API 是否支持以下操作**？
   - 按客户 + 时间查留货订单
   - 按客户查欠款汇总/明细
   - **创建询价单（支持批量 N 个 item）**
   - 按订单/车牌查装车重量（含磅单图）
   - 按炉号/订单查材质书 PDF
   - 按客户 + 月份查结算单 PDF
   - 提交付款凭证（写）
   - 提交发货催办（写）
3. **询价 ERP 报价流是什么**？销售在 ERP 里是逐 item 报价还是整单报价？
4. **报价回推**是否需要同时附 PDF 报价单（除了消息内文字表格）？
5. **付款凭证是否需要 OCR**？OCR 服务选阿里 / 腾讯 / 自建？
6. **多 sheet Excel 询价处理策略**：默认询第一个 sheet？还是必反问？
7. **询价文件保留多久**？默认 1 年是否合规？
8. **VL 模型预算**：每月预估调用次数和成本上限？
9. **销售在 Bot 里回复客户** vs **ERP 里操作** 偏好哪种？
10. **询价/催发货/付款核对 SLA** 各是多少？
11. **客户绑定方式**：销售生成 token / 手机号短信 / 都要？
12. **是否需要群聊场景**？
