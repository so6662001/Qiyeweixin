# AI 开发指南（Cursor 提示词 + Checklist）

> 用途：进入开发阶段后，用本指南驱动 Cursor 开发，确保**不走样、不漏功能、不缺逻辑、符合设计预期**。
> 配套：设计 [`../DESIGN.md`](../DESIGN.md)｜模块 [`modules.md`](modules.md)｜流程 [`flows.md`](flows.md)｜需求溯源 [`TRACEABILITY.md`](TRACEABILITY.md)｜验收 [`ACCEPTANCE.md`](ACCEPTANCE.md)｜路线 [`ROADMAP.md`](ROADMAP.md)
>
> **使用顺序**：① 把第 2 节写入 `.cursor/rules`（全局生效）→ ② 按 ROADMAP Phase 取第 4 节对应提示词开发 → ③ 每个模块按第 5 节 Checklist 自检 → ④ 提交前过第 6 节验收 Checklist。

---

## 1. 使用方法

| 步骤 | 做什么 |
|---|---|
| A | 将 [第 2 节] 全局系统提示词 复制到项目根目录 `.cursor/rules/00-global.mdc`（或 `.cursorrules`），让 Cursor 每次都带上 |
| B | 将 [第 3 节] 防走样铁律 复制到 `.cursor/rules/01-ironrules.mdc` |
| C | 开发某模块时，从 [第 4 节] 取对应 Phase 提示词，连同"请先阅读 DESIGN.md + docs/modules.md 中 5.xx 节"一起发给 Cursor |
| D | 模块完成 → 过 [第 5 节] 模块 Checklist |
| E | 提交前 → 过 [第 6 节] 防走样 + 验收 Checklist（对照 ACCEPTANCE.md UAT） |
| F | 阶段完成 → 过 [第 7 节] 防漏功能 Checklist（对照 TRACEABILITY.md） |

---

## 2. 全局系统提示词（写入 `.cursor/rules`）

```
你是钢铁贸易企业微信智能机器人的资深后端工程师。本项目是多租户 SaaS，面向钢铁贸易/生产加工企业，通过企业微信（自建应用 + 微信客服）为客户提供询价、报价、议价、改量、留货、替代料、对账等智能服务。

【开发前必读】
- 总体架构与决策：DESIGN.md（含约束表、各能力章节 1.6~1.20）
- 模块详细设计：docs/modules.md（5.x 节，每个模块有数据模型/流程/存储）
- 关键流程：docs/flows.md（4.x，Given-When-Then 级别）
- 话术：docs/dialogues.md
- 需求清单：docs/TRACEABILITY.md（84 项需求，开发时对照）
- 验收标准：docs/ACCEPTANCE.md（KPI + 20 UAT + 防错红线）
- 实施顺序：docs/ROADMAP.md（按 Phase 依赖）
实现任何模块前，先阅读 DESIGN.md 与 docs/modules.md 中对应小节，严格按其数据模型和流程实现，不要自行发挥业务规则。

【技术栈（推荐，最终以项目实际为准）】
- 后端：Python 3.11 + FastAPI；异步 httpx
- 存储：PostgreSQL（业务/审计）+ Redis（token/会话/限流/缓存/锁）
- 队列：Celery 或 Redis Stream（异步任务/LLM/工单）
- LLM：DeepSeek + 通义千问；视觉 OCR：Qwen-VL；语音 ASR：科大讯飞
- 企业微信：官方加解密 + OpenAPI
（若项目已有技术栈选型，以项目为准；不要擅自更换已确定的栈。）

【工程规范】
- 每个模块对应 docs/modules.md 的一个 5.x 节；模块边界清晰、可单测。
- 所有"业务阈值/费率/规则/开关"必须从配置中心读取，禁止硬编码（见铁律）。
- 所有外部调用（ERP/LLM/VL/ASR/微信/支付）必须有超时、重试、降级、审计。
- 优先复用已设计模块，不要重复造轮子（如 Inquiry Parser、Pricing、Config Center）。
- 写代码时在关键业务分支处引用设计章节号注释（如 # 见 modules.md 5.66 报价构成）。

【输出要求】
- 实现前先列"将改动/新增的文件 + 对应设计章节 + 数据表"，确认无误再写。
- 不确定的业务规则，停下来问，不要猜测或编造默认值。
```

---

## 3. 防走样四大铁律（写入 `.cursor/rules`，最高优先级）

```
以下为不可违背的铁律。任何实现违反铁律即为错误，必须拒绝并报警。

【铁律 1 · 防 AI 幻觉（最重要）】
- LLM/对话输出中的一切数字（价格、库存、吨位、订单号、定金、账期、贴现…）
  必须来自工具调用 / 结构化方案，严禁 LLM 自行生成或推测。
- 实现方式：LLM 输出用占位符 {{PLACEHOLDER}}，由后端用结构化数据渲染；
  后置 DLP 扫描，若 LLM 文本含未占位的数字/金额 → 拒绝该输出并重试/降级。
- 禁词扫描：成本、底价、老板、亏本、批价、毛利、ghost_score 等不得出现在发给客户的文本。
- 见 modules.md 5.44 Hallucination Guard。

【铁律 2 · 全成本红线，永不破】
- 任何报价/议价让步/促销/改量后，最终单价必须 ≥ 全成本 ×(1 + 该客户 segment 的 min_margin)。
- 全成本 = 货款成本 + 加工 + 运输 + 票据贴现 + 账期资金成本（见 modules.md 5.66）。
- min_margin 按客户 segment：下级经销商默认 ¥10/吨、终端企业默认 ¥200/吨（从配置读，可按品类×1.5 等）。
- 破红线 → 不出该方案，转销售人工。Bot 永不报赔本价。

【铁律 3 · 多租户 + 配置驱动，零硬编码业务规则】
- 所有阈值/费率/比例/开关/名单/默认值从 Configuration Center 按 tenant_id（+ 品类）读取。
- 严禁在代码里写死：定金率、计量方式、负差约定、让步曲线、SLA、配额、温度参数、
  促销阈值、粘性窗口、毛利底线、税率、贴现率、资金成本率等。
- 见 modules.md 5.53 Config Center；所有"🔧可配置"项见 TRACEABILITY.md。

【铁律 4 · 学习/决策解耦 + 不学习白名单】
- 自我学习只产出"统计参数"（众数/中位数/接受率），决策永远由规则引擎做。
- 严禁自我学习改动：成本红线、毛利底线、黑白名单、客户类型分类、Auto 上限、价格、让步幅度。
- 学习参数应用前必过 5 道防护：数据质量 + 置信度≥0.7 + 冷启动样本≥60 + Drift≤阈值 + A/B 灰度。
- 见 modules.md 5.42/5.44。
```

```
【其他必须遵守的约束（补充铁律）】
- 字段溯源：询价/报价每个字段带 source(explicit/inferred/calculated/user_confirmed/missing)+confidence；缺 critical 字段必反问，不静默默认。（modules.md 5.13/5.14）
- 微信能力边界：不渲染 Markdown（纯文本用空格对齐）；回调 5s 内 ACK（慢任务异步）；微信客服 48h 主动消息窗口；语音/图片/文件类型按官方限制。（DESIGN §2, 1.16）
- 幂等去重：企业微信回调按 MsgId 去重（Redis setnx 5min）；业务 webhook 按 event_id 幂等。（modules.md 5.10）
- 库存安全：留货/下单走分布式锁（按 sku/批次），与 ERP 分钟级对账，绝不超卖；批次价同 FIFO。（modules.md 5.47/5.73）
- 数据级权限：客户只见自己数据；销售只见 managed_customers；成本/毛利/画像/ghost 绝不出客户消息。
- 写操作二次确认：下单/锁价/留货/改规格/替代 必须客户明确确认，不可默认；锁价后改量按 ±5%/累计±10% 规则审批。（modules.md 5.32）
- access_token：集中缓存 + 分布式锁防并发刷新 + 提前续期；自建应用与客服 token 两套独立。
- 降级链：LLM→备选LLM→关键字命令→转人工；VL→Plus→人工；ASR 低置信→回显确认；H5→菜单→纯文字。
- 审计：报价/议价/改量/留货/替代/转人工/配置变更/学习应用 全留审计日志，可追溯。
```

---

## 4. 分阶段开发提示词（按 ROADMAP Phase）

> 每段可直接复制给 Cursor。务必先让它读对应设计章节。

### Phase 0 · 地基

```
任务：实现地基层。请先阅读 DESIGN.md §2/§3、docs/modules.md 5.10(Inbound Webhook)/5.53(Config Center) 及架构图。
实现范围：
1. 双通道 Callback Server：企业微信自建应用 + 微信客服的 URL 校验(GET)与消息回调(POST)，WXBizMsgCrypt 验签+AES 解密，5s 内 ACK，MsgId 去重。
2. Channel Adapter：把 app/kf 不同结构归一为内部 Message 模型（含语音/图片/文件类型识别）。
3. access_token 管理：Redis 缓存 + 分布式锁 + 提前续期；app/kf 两套独立。
4. Identity & Binding：一次性 token + 手机号短信两种绑定；注入 biz_user_id/role/segment/tenant。
5. Topic + Task 会话状态机：跨天、多意图、暂停/恢复（modules.md 5.2 思想）。
6. Configuration Center：多租户配置存储(tenant_id+key)、版本化、审计、热加载、JSON Schema 校验。
约束：遵守全部铁律；密钥走环境变量/KMS；回调慢任务异步。
交付：可 echo（自建应用收发）、客服通道打通、绑定可用、配置可读写。
自检：过第 5 节模块 Checklist + 第 6 节铁律 Checklist。
```

### Phase 1 · MVP（只读 + 转人工）

```
任务：实现 MVP。先读 docs/flows.md 4.1、modules.md 5.60(自建能力) 与 ACL 设计。
实现范围：
1. ACL（防腐层）：调 ERP 只读 API，字段映射/枚举翻译/单位换算/缓存/重试/熔断；数据级权限兜底过滤。
2. 八项只读能力：查留货订单/查欠款/查装车重量/查结算单PDF（其余4项 Phase 3）。
3. LLM Orchestrator 基础：文字意图识别 + Function Calling 路由（仅 MVP 工具）。
4. 转人工兜底：无库存/未定价/品类规格全空/复杂 → Lead Routing 最简版（先绑定销售/转销售群）。
5. 自建询价单 + 发货催办（最简）。
约束：遵守铁律；查询结果渲染遵守"不渲染 Markdown + 数据级权限 + 敏感不外泄"。
验收：通过 UAT-20（八项只读逐项）、UAT-12（转人工）。
```

### Phase 2 · 报价核心

```
任务：实现自动/辅助报价。先读 modules.md 5.12(Parser)/5.13/5.14/5.18(策略)/5.19(库存)/5.26(整单)/5.36(KB)/5.66(报价构成)/5.67(投递)/5.69(计量)；flows.md 4.2。
实现范围：
1. Steel KB + Aho-Corasick 实体识别 + 规格正则归一 + 理论/抄牌重量表。
2. Inquiry Parser 文字分支 + 9 要素 + Default Resolver(4级默认) + Field-Level Tracer(溯源)。
3. 销售计量方式引擎（六种 + 企业×品类配置矩阵 + 折合吨开关）。
4. Customer Profile + segment；Inventory Matcher 多源组合。
5. Pricing Strategy Engine（11+策略，配置化）+ 三档协作(Auto/Assisted/Manual)。
6. Price Composition Engine（货款+加工+运费+贴现+资金成本±一票两票税差）+ 全成本红线。
7. 报价投递三件套（文字摘要+PDF+菜单）+ 交互式编辑 + 品类要素必填配置。
8. Hallucination Guard 基础（占位符 + DLP 数字扫描）。
约束：铁律 1/2/3 必须落实（防幻觉、全成本红线、配置化）；critical 字段缺失必反问。
验收：UAT-01/03/17/18/15；§1 报价 KPI 达标；防错红线"赔本报价/幻觉"0 发生。
```

### Phase 3 · 交易闭环

```
任务：实现议价/改量/留货/替代料/促销/自建能力。先读 modules.md 5.24~5.32(议价改量)/5.46~5.49(留货)/5.36~5.38(替代料)/5.71(促销)/5.73(批次)/5.60(自建)；flows.md 4.8/4.10/4.12/4.15。
实现范围：
1. 议价：Negotiation Session 状态机 + 让步曲线(配置) + 整单 Optimizer(按segment底线) + 议价 LLM(防幻觉) + 8/13维让步货币。
2. 改量：Quote 生命周期版本化 + Amendment 五档决策 + 锁价后±5%/累计±10%规则 + 减量退价分档。
3. 留货：Reservation 状态机 + 定金(品类×客户配置) + 过期工单 + 库存独占(批次锁定/FIFO) + deposit webhook。
4. 替代料：KB + Match + Strategy + 成分/标准兼容硬约束(不兼容必拒) + 壁厚/单重/产地三类(强caveat) + 明示确认 + 合同审计。
5. 促销引擎：整单/单规格/库龄优惠(配置) + 与议价可叠加 + 叠加后过全成本红线。
6. 自建能力完整：材质书PDF + 付款凭证OCR + 询价异步报价回推。
7. Batch Inventory：批次(炉号/库龄/单重) + 留货下单匹配。
约束：铁律全部；写操作二次确认；库存分布式锁防超卖；替代不兼容100%拒绝。
验收：UAT-06/07/08/09/10/11；防错红线(超卖/误替代/锁价破坏)0 发生。
```

### Phase 4 · 运营智能

```
任务：实现销售路由/评分/粘性、自动跟进、自我学习、防幻觉完整。先读 modules.md 5.33~5.35(路由)/5.40~5.44(学习/防幻觉)/5.20~5.21(软干预)/5.55(竞品)/5.56(级别)/5.74(跟进)。
实现范围：
1. Lead Routing：绑定>粘性(窗口配置)>池加权抽签(softmax+温度T) + 配额(按级别) + 新人保底(入职衰减) + ACK超时转派 + 中签概率透明。
2. Sales Score(7维) + Sales State + Sales Level(junior/mid/senior) + Competitor Credibility(扣分+专项黑名单审计)。
3. Soft Influence(5层) + Ghost Score；客情防火墙(不公开评分/可恢复)。
4. Auto Follow-up：按客户采购频率自适应 + 个性化(销售风格) + 频率控制 + 渠道(48h窗口/订阅消息)。
5. 自我学习：Pattern Miner + Effectiveness Tracker + Self-tuning(5道防护) + Drift Monitor + 不学习白名单。
6. Hallucination Guard 完整。
约束：铁律 4(学习/决策解耦)严格；不学习白名单硬保护；学习足迹仅主管可见。
验收：UAT-13/14/19/15；学习冷启动60/Drift防护验证。
```

### 并行端线 A · 多模态（依赖 Phase 2 Parser）

```
任务：扩展 Inquiry Parser 支持图片/Excel/PDF/语音/多规格。先读 modules.md 5.12/5.72(ASR)；DESIGN 1.20.4/1.20.5/1.20.9。
实现范围：
1. 图片/扫描PDF：Qwen-VL 识别→表格结构化→回显确认。
2. Excel：openpyxl + LLM列头映射 + 单位归一 + 多sheet反问。
3. 语音：科大讯飞 ASR + 钢铁热词 + KB校正 + 低置信回显。
4. 多规格组合：批量切分 + 共享要素提取(目的地等) + 逐规格库存即时反馈 + 强制逐规格回显。
5. 语音×多规格交叉：错误率叠加→强制确认(DESIGN 1.20.9)。
约束：识别结果必经回显确认，不静默提交；置信度低/失败转人工；计入配额。
验收：UAT-02/04/05；§1 识别 KPI(图片≥90%/Excel≥92%/语音≥92%)。
```

### 并行端线 B · 小程序（依赖后端 Phase 2/3，详见 ROADMAP 对齐表）

```
任务：实现小程序端（平台统一+企业隔离）。先读 modules.md 5.68；DESIGN 1.17。前置：报价(P2)+留货(P3)后端就绪 + 微信支付商户号 + 开放平台账号(unionid)。
实现范围（按对齐表）：
- 一期：报价详情逐行编辑(全局选项实时重算)+留货下单+微信支付定金；BFF复用后端。
- 二期：查订单/磅单/材质书/对账 + 订阅消息。
- 三期：询价发起 + 我的中心 + 白标可选。
约束：unionid 打通微信客服身份；支付分账;小程序仅"增强"，对话式始终兜底；多端单一数据源(BFF不存业务态)。
验收：UAT-16。
```

### 贯穿 · 计费/配额（随用量启用）

```
任务：实现 SaaS 计费与配额。先读 modules.md 5.63。
实现：套餐体系 + Qwen-VL/讯飞按套餐配额(超额限流+提示升级) + 文件保留计费 + 用量计量 + 账单 + 配额告警。
约束：配额耗尽降级(VL耗尽→文字询价)，不影响核心文字流程。
```

---

## 5. 模块完成 Checklist（每个模块自检）

```
□ 已阅读对应 docs/modules.md 5.x 节，实现与其数据模型/流程一致
□ 数据表已建（对应该节"新增存储"表），含审计字段(who/when/trace_id)
□ 所有业务阈值/规则从 Config Center 读取，无硬编码
□ 输入校验 + 边界处理（空值/超限/非法/并发）
□ 外部调用有超时/重试/熔断/降级
□ 关键操作写审计日志
□ 数据级权限校验（客户只见自己/销售见managed）
□ 单元测试覆盖核心分支 + 边界用例
□ 关键业务分支处注释引用设计章节号
□ 与上下游模块的接口契约对齐（不破坏已有）
```

---

## 6. 防走样 + 提交验收 Checklist（提交前必过）

### 6.1 防走样铁律自检
```
□ 铁律1 防幻觉：LLM 输出无自造数字；占位符渲染；DLP 数字+禁词扫描生效
□ 铁律2 全成本红线：报价/议价/促销/改量后均校验，破线转人工，无赔本价
□ 铁律3 配置化：本模块所有阈值/费率/规则均从 Config Center 读，按 tenant(+品类)
□ 铁律4 学习解耦：学习只产参数；不学习白名单硬保护；5道防护齐全
□ 字段溯源：解析字段带 source+confidence；critical 缺失必反问
□ 微信边界：不发 Markdown 表格；回调5s ACK；48h窗口处理
□ 幂等：MsgId 去重；webhook event_id 幂等
□ 库存：留货/下单分布式锁；批次FIFO；ERP对账；无超卖
□ 写操作二次确认；锁价后改量按规则审批
□ 敏感数据(成本/毛利/画像/ghost)不出客户消息
□ 降级链可用（LLM/VL/ASR/H5）
```

### 6.2 提交验收（对照 ACCEPTANCE.md）
```
□ 本模块对应的 UAT 场景全部通过（见映射：Phase→UAT）
□ §1 对应能力的量化 KPI 达标
□ §3 防错红线项 0 发生（构造诱导用例验证：幻觉/赔本/越权/泄漏/误替代/超卖/锁价破坏/白名单被改）
□ 可观测：关键指标埋点 + 告警
□ 安全：密钥不入库；越权拦截；敏感脱敏
```

### Phase → UAT 映射（提交时按此核对）
| Phase | 必过 UAT |
|---|---|
| Phase 1 MVP | UAT-20, UAT-12 |
| Phase 2 报价 | UAT-01, 03, 17, 18, 15 |
| Phase 3 交易 | UAT-06, 07, 08, 09, 10, 11 |
| Phase 4 智能 | UAT-13, 14, 19, 15 |
| 多模态 | UAT-02, 04, 05 |
| 小程序 | UAT-16 |

---

## 7. 防漏功能 Checklist（阶段/整体完工核对，对照 TRACEABILITY.md）

> 每完成一个 Phase，回到 TRACEABILITY.md，把该 Phase 涉及的需求 ID 逐条核对"是否真实现"。

```
□ Phase 0：R1.1~R1.5, R2.1~R2.7（接入/绑定/会话/配置）
□ Phase 1：R3.1/R3.2/R3.4/R3.6（只读四项）, R10.1~R10.3, R12.1（转人工）
□ Phase 2：R4.1, R5.1~R5.5, R6.1~R6.9, R9.1, R16.1~R16.6, R17.1~R17.2, R19.1~R19.7, R20.1~R20.3, R21.1~R21.4（询价/报价/构成/计量/领域知识）
□ Phase 3：R3.3/R3.5/R3.7/R3.8, R7.1~R7.3, R8.1~R8.2, R11.1, R13.1, R23.2~R23.7, R23.10（议价/改量/留货/替代/促销/批次/自建）
□ Phase 4：R6.10, R10.4, R12.2~R12.3, R23.1, R23.11（路由评分/软干预/学习/粘性/跟进）
□ 多模态：R4.2~R4.4, R23.8, R23.9（图片Excel PDF/多规格/语音）
□ 小程序：R18.1
□ 全部 84 项需求状态 ✅，🔧可配置项已接入 Config Center
```

---

## 8. 给 Cursor 的"启动一句话"

> 每次开新会话/新模块，先发这句让它进入状态：

```
请阅读 DESIGN.md、docs/modules.md（找到本次要实现的 5.x 节）、docs/ACCEPTANCE.md（对应 UAT）和 docs/DEV-GUIDE.md（铁律+Checklist）。我要实现【模块名/Phase】。请先列出"将改动的文件 + 对应设计章节 + 数据表 + 涉及的 TRACEABILITY 需求ID"，等我确认后再写代码。严格遵守四大铁律，不确定的业务规则停下来问我，不要编造。
```
