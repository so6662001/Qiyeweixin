# 企业微信机器人 — 设计文档（v6）

> 状态：设计阶段（尚未开发）
> 行业：**钢铁贸易**
> 目标：搭建一个企业微信智能机器人，对接 8 项后端能力，覆盖内部员工和外部微信客户，接入 LLM（DeepSeek / 通义千问，含 VL 视觉模型）。
> **v6 核心**：把"询价 → 销售单干报价"升级为「**自动/辅助/人工」三档报价 + 客户画像引擎 + 报价策略引擎 + 库存组合匹配 + 行为干预**」。

---

## 0. 需求方已确认的关键约束

| # | 答复 | 影响 |
|---|---|---|
| 1 | 内 + 外双通道 | 自建应用 + 微信客服 |
| 2 | 业务 API 不可改 | ACL |
| 3 | 必须绑定 | 数据级权限 |
| 4 | 多轮、跨天、多意图 | Topic + Task |
| 5 | 公网云、域名已备案 | 云原生 |
| 6 | 钢铁贸易 | 强权限 + DLP + 审计 |
| 7 | DeepSeek + 通义千问 | LLM 双路由 |
| 8 | 8 项后端能力 | 工单 + 反向回调 + 文件管道 |
| 9 | 询价 4 形态 | 多模态解析 + VL |
| 10 | 询价 9 要素口语化 | KB + 默认推断 + 字段级溯源 |
| **11（v6 新）** | **多库存/多价格、客户特性差异、量级分层、MOQ、黑白名单、白嫖客户管理** | **报价策略引擎 + 客户画像引擎 + 三档协作 + 行为干预** |

---

## 1. 需求与目标

### 1.1~1.4 同 v5
略。

### 1.5 报价决策矩阵（v6 核心）

#### 1.5.1 三档协作模型

| 档位 | 触发条件 | 谁出价 | 客户体感 |
|---|---|---|---|
| **A. 即时自动报价 (Auto)** | 白/标准客户 ∧ 标准品 ∧ 量在阈值内 ∧ 库存充足 ∧ 风险低 | Bot 直接出 | 秒级响应 |
| **B. 辅助报价 (Assisted)** | 大多数情况 | Bot 出**建议方案**→销售一键确认/调整→回客户 | 几分钟内 |
| **C. 人工报价 (Manual)** | 黑名单 / 超大单 / 非标 / 复杂组合 / 信用警告 | 销售全程，Bot 仅传话 | 30 min 内 |

**判定流程**：
```
parse_inquiry 完成
   ↓
风险评估（5 项）：
  1. 客户名单（白/普/黑）
  2. 客户信用（正常/警告/冻结）
  3. 单笔总额 vs 阈值
  4. 品类是否标准品
  5. 是否需要锁价/账期
   ↓
→ Manual：任一红灯
→ Auto：全绿灯 + 白名单/老客 + 量 ≤ 阈值
→ Assisted：其余
```

#### 1.5.2 策略选择矩阵

| 客户类型 → | 新客 | 量型 | 利型 | 战略 | 流失 |
|---|---|---|---|---|---|
| **白名单** | 入门优惠 + 短锁价 | 量优底价 + 长锁价 | 质优策略 + 标准锁价 | 战略价 + 超长锁价 | 唤回价 + 限时 |
| **普通** | 标准价 + 短锁价 | 量阶梯 + 标准锁价 | 质优策略 | 战略价 | 标准价 + 销售跟进 |
| **黑名单** | 拒报 / 转销售 | 上浮 5%~20% + 现款 | 上浮 + 现款 | （不应存在） | 拒报 |

#### 1.5.3 量级策略

| 询价量 / 整单量 | 处理 |
|---|---|
| 量 < 品类 MOQ | 标"**不过磅销售**"（按支/根/件计费） |
| MOQ ≤ 量 < 标准段 | 标准报价（按吨过磅） |
| 量 ≥ 大宗阈值 | 阶梯优惠 + 可锁价更长 |
| **没有量（纯询价）** | 出**指导价 (indicative)** + 注明"实际成交以提货时为准" |
| 多规格组合询价 | 总量合并算阶梯（按客户类型决定） |

---

## 2. 通道选型 / 3. 总体架构（v5 基础 + v6 新模块）

```
   ...省略前置（Callback / Channel / Identity / Permission / Router / Session / LLM / Tools / ACL / Reply / WeCom）...

   ─────────── v6 新增模块 ───────────

   ⑨ Customer Profile Engine（客户画像引擎）
      - 自动分类：新客/量型/利型/战略/流失
      - 名单：白/普/黑（黑名单分级）
      - 信用：正常/警告/冻结
      - 行为画像：转化率/平均吨位/DSO/品类集中/白嫖系数
      - 销售可手动覆盖
      - 周期更新（小时 + 实时事件）

   ⑩ Pricing Strategy Engine（报价策略引擎）
      - 输入：InquiryItem + customer_profile + inventory_options
      - 策略库（可配置规则）：
          new_customer_attractive
          volume_driven_lowest
          profit_quality_first
          strategic_anchor
          churn_callback
          blacklist_uplift / cash_only / refuse
          whitelist_discount
          moq_unweighed
          no_qty_indicative
          repeat_inquiry_quote_reuse
          explorer_throttle
          ghost_minimum
      - 输出：QuoteOption[] + price_breakdown + valid_until + 备注

   ⑪ Inventory Matcher（库存组合匹配）
      - 同规格多库存源（仓库/批次/品质）
      - 组合优化：单源最便宜 vs 多源拼单 vs 含运费总成本
      - MOQ 校验 + 不过磅模式

   ⑫ Conversion Behavior Tracker（行为漏斗追踪）
      - 询价→报价→锁价→下单 漏斗
      - 时间窗口指标（7/30/90 天）
      - 自动分类信号 + 异常告警

   ⑬ Soft Influence Module（软影响 / 行为干预）
      - 回执话术按客户画像差异化
      - 锁价时长差异化
      - 重复询价压制
      - 销售介入触发
      - 5 层渐进式机制（详见 5.21）
```

---

## 4. 关键流程（v6 重写询价报价闭环）

### 4.2 询价 → 报价 全链路（v6 重写）

```
═══ 阶段 A：解析（v4/v5）═════════════════════════════════════
parse_inquiry：文字/图/Excel/PDF → InquiryDraft → 客户确认

═══ 阶段 B：客户画像注入 (v6) ═════════════════════════════════
Customer Profile Engine 查：
  profile = {
    type: 量型, list: 普通, credit: 正常,
    behavior: {
      inquiry_30d: 12, deal_30d: 2, conversion_rate: 0.17,
      avg_ton: 80, dso_days: 45, profit_margin_hist: 4.2%,
      categories_dominant: [螺纹钢, 中厚板],
      ghost_score: 0.62,    // 越高越像白嫖
    }
  }

═══ 阶段 C：风险评估 + 档位判定 (v6) ═══════════════════════════
risk = {
  list_red: false,        // 黑名单
  credit_red: false,      // 信用警告/冻结
  amount_red: false,      // 单笔 > 500w
  nonstandard: false,     // 非标规格
  needs_lock: false       // 客户要求锁价
}
→ 档位 = Assisted（默认）

═══ 阶段 D：库存匹配 (v6) ════════════════════════════════════
Inventory Matcher 查：
  options = [
    {warehouse:'武汉江夏', stock_ton:30, base_price:3820, attrs:'标准品'},
    {warehouse:'武汉江夏', stock_ton:50, base_price:3790, attrs:'倍尺'},
    {warehouse:'襄阳',     stock_ton:100,base_price:3750, attrs:'长锈轻锈'},
    {warehouse:'武汉江岸', stock_ton:20, base_price:3850, attrs:'优级'}
  ]

═══ 阶段 E：策略应用 (v6 核心) ═══════════════════════════════
Pricing Strategy Engine 按 (profile.type × profile.list × 量级) 选策略：

  客户=量型 + 普通 + 询价50t:
    主策略 = volume_driven_lowest
    输出 QuoteOption[]:
      [1] 推荐方案：50t 拆 30 武汉 + 20 襄阳
          均价 ¥3,792  含倒短运费 ¥30/t  总价 ¥189,600
          锁价 24h
      [2] 标准方案：50t 武汉江夏倍尺
          单价 ¥3,790  锁价 24h
      [3] 优质方案：50t 武汉江夏标准品（库存 30+20）
          均价 ¥3,820  锁价 24h
          [建议销售跟进：客户偏低价时可不推荐]

  若客户=利型 + 普通:
    主策略 = profit_quality_first
    输出：优先推 [3]，备选 [1]

  若客户=新客 + 普通:
    主策略 = new_customer_attractive
    输出：推 [2]，并补"首单优惠 -¥10/t"，锁价 6h（短）
          附话术：欢迎首单合作，本次为新客特批

  若客户=黑名单（普通级别）:
    主策略 = blacklist_uplift
    输出：[3] 上浮 ¥80/t，注明"现款现货"，锁价 2h

═══ 阶段 F：行为干预 (v6 软影响) ═════════════════════════════
Soft Influence 检测到 ghost_score 0.62（询多买少）:
  - 锁价时长降为 6h（默认 24h）
  - 不展示阶梯优惠（不让客户拿着低价四处比）
  - 附"销售小张稍后联系您"，标记 sales_followup=true
  - 不在回执文字里抱怨；仅通过差异化让客户"感知到"

═══ 阶段 G：MOQ 校验 ═══════════════════════════════════════
品类 MOQ 表查：螺纹钢 MOQ = 30t
本次 50t ≥ 30t → 正常报价
若 < 30t → 标 unweighed_only = true，价格按"不过磅 ¥/支"

═══ 阶段 H：档位执行 ═════════════════════════════════════════
档位 = Assisted:
  - Bot 把 QuoteOption[] 发给销售小张（企微 textcard）
    «客户张总询价 INQ-001
     画像：量型 普通 信用正常 ghost=0.62 ⚠️白嫖偏高
     系统建议：方案 1 ¥3,792 均价（拆单），锁价 6h
     [一键确认] [调整价格] [改方案] [转人工]»
  - 销售 10s 内点"一键确认" → Bot 回客户
  - 或销售调整后确认 → Bot 回客户

档位 = Auto:
  - 跳过销售，Bot 直接回客户
  - 异步抄送销售（"已为白名单客户 XX 自动报价 ¥xxx"）

档位 = Manual:
  - Bot 不出价，仅创建工单 + 摘要给销售
  - 销售全自助回价

═══ 阶段 I：客户收到报价 ════════════════════════════════════
（话术按客户画像差异化，见 6.4）

═══ 阶段 J：转化追踪 ═══════════════════════════════════════
报价后 24h / 7d / 30d 未成交 → 触发不同策略：
  - 12h 未回应：销售自动收到"客户未回复"提醒
  - 锁价过期：Bot 主动询问"价格已到期，是否需要重新报价"（限频）
  - 持续 N 次报价未成交 → ghost_score↑，下次报价更短锁价 + 销售关怀
```

### 4.3 重复询价压制（v6 新增）

```
客户在 24h 内询同样规格的螺纹 HRB400 25mm 沙钢 50t：
  - Bot 不重新走策略
  - 直接引用上次 QuoteOption
  - 话术："此规格 24h 内已为您报价 ¥3,792（剩余有效 4h）
           如需续期请回复『续期』；如需新方案请回复『重新报价』"
  - 销售可手动豁免（如行情大跳）
```

### 其他流程（4.1 同步查询 / 4.4~4.7）同 v5
略。

---

## 5. 模块设计

### 5.1~5.16 同 v5
略。

### 5.17 Customer Profile Engine（v6 新增核心）

**5.17.1 属性体系**

```yaml
CustomerProfile:
  identity:
    customer_id, name, contact, region, salesperson_id
  type:                    # 客户类型（自动 + 销售可覆盖）
    primary: new | volume | profit | strategic | churn
    confidence: 0.0~1.0
    last_classified_at: timestamp
    override: { by: sales_xxx, type: profit, reason: "..." }
  list:                    # 名单
    status: whitelist | normal | blacklist
    blacklist_level: null | uplift | cash_only | prepay | refuse
    expires_at: timestamp  # 黑名单可设有效期
  credit:
    status: normal | warning | frozen
    limit: 5000000
    used: 1234567
  behavior:
    inquiry_30d, inquiry_90d
    quote_30d, quote_90d
    deal_30d, deal_90d
    conversion_rate_30d, conversion_rate_90d
    avg_ton_per_order
    dso_days
    profit_margin_hist
    categories_dominant: [螺纹钢, 中厚板]
    ghost_score: 0.0~1.0   # 见 5.20
    last_deal_at
    seasonal_pattern: ...  # 选填
```

**5.17.2 自动分类规则（默认 + 可配）**

```
new      ← 首次合作 < 90 天 AND 累计成交单数 < 3
volume   ← 平均订单吨位 > P75 OR 月度采购量稳定且高
profit   ← 历史毛利率 > P60 AND 平均订单吨位 < P75
strategic← 销售或管理员手动标
churn    ← 90 天无成交 AND 历史有成交
```

**5.17.3 更新频率**

- 实时事件驱动：下单/付款/逾期/询价 → 更新对应指标
- 每小时跑一次轻量重分类
- 每天跑一次全量重算
- 销售在 Bot/CRM 里可手动改 type 或 list（带原因记录）

**5.17.4 数据来源**

- ERP / CRM（订单、应收、回款、客户档案）
- Bot 自身（询价、转化、对话）
- 销售手动标注

### 5.18 Pricing Strategy Engine（v6 新增核心）

**5.18.1 设计原则**

- **规则可配置**：策略 = YAML/JSON 配置 + 价格基线（不硬编码）
- **可解释**：每条 QuoteOption 必须能追溯到（基价 → 应用了哪些规则 → 最终价）
- **多策略同时输出**：默认输出主策略 + 1~2 备选
- **风险红线**：低于成本价的报价必须拦截 + 转销售（Bot 永不报赔本价）

**5.18.2 策略库（节选）**

```yaml
strategies:

  - name: new_customer_attractive
    when:
      profile.type: new
      profile.list: [whitelist, normal]
    actions:
      - select_inventory: lowest_attribute_match
      - discount: { fixed: -10, unit: "元/吨" }    # 首单优惠
      - lock_minutes: 360                          # 短锁价 6h（避免被薅）
      - reply_template: new_customer_welcome
      - tag: "首单优惠"

  - name: volume_driven_lowest
    when:
      profile.type: volume
      profile.list: [whitelist, normal]
    actions:
      - select_inventory: best_blended_price       # 拼仓最低
      - ladder:                                    # 量阶梯
          - qty_gte: 100, discount: -20
          - qty_gte: 200, discount: -35
      - lock_minutes: 1440                         # 24h
      - reply_template: volume_focus

  - name: profit_quality_first
    when:
      profile.type: profit
      profile.list: [whitelist, normal]
    actions:
      - select_inventory: best_quality
      - markup_quality_premium: 10
      - lock_minutes: 1440
      - reply_template: quality_focus

  - name: strategic_anchor
    when: { profile.type: strategic }
    actions:
      - use_strategic_price_table: true
      - lock_minutes: 4320                         # 72h
      - reply_template: strategic_partner

  - name: churn_callback
    when:
      profile.type: churn
    actions:
      - discount: { fixed: -30 }                   # 唤回价
      - lock_minutes: 720                          # 12h
      - reply_template: churn_callback
      - trigger_sales_followup: true

  - name: whitelist_discount
    when: { profile.list: whitelist }
    actions:
      - discount: { fixed: -15 }                   # 白名单基础下浮
      - lock_extend: 1.5                           # 锁价延长 50%

  - name: blacklist_uplift
    when: { profile.list: blacklist, profile.blacklist_level: uplift }
    actions:
      - markup: { rate: 0.03 }                     # 上浮 3%
      - require: cash_only

  - name: blacklist_cash_only
    when: { profile.blacklist_level: cash_only }
    actions:
      - markup: { rate: 0.05 }
      - reply_template: blacklist_cash_only

  - name: blacklist_refuse
    when: { profile.blacklist_level: refuse }
    actions:
      - block_quote: true
      - reply_template: blacklist_refuse_polite
      - escalate_to_human: true

  - name: moq_unweighed
    when: { qty_under_moq: true }
    actions:
      - pricing_mode: per_piece
      - markup: { rate: 0.04 }                     # 不过磅成本高
      - reply_template: moq_unweighed
      - tag: "不过磅销售"

  - name: no_qty_indicative
    when: { qty_missing: true }
    actions:
      - quote_type: indicative
      - lock_minutes: 60                           # 极短
      - reply_template: indicative_only
      - tag: "指导价"

  - name: explorer_throttle
    when: { profile.behavior.ghost_score_gte: 0.5 }
    actions:
      - lock_minutes_override: 360                 # 不给长锁价
      - hide_ladder: true                          # 不暴露阶梯
      - trigger_sales_followup: true
      - reply_template: explorer_soft_hint

  - name: repeat_inquiry_quote_reuse
    when: { same_spec_in_24h: true }
    actions:
      - reuse_last_quote: true
      - reply_template: repeat_inquiry_reuse
```

**5.18.3 策略合成**

多条策略可叠加，但有优先级 + 互斥：
- `blacklist_*` 一律最高优先级，且禁止与 `whitelist_*` 共存
- `explorer_throttle` 可与其他策略叠加（覆盖锁价时长）
- 类型策略（`new/volume/profit/...`）互斥，选最匹配一条
- 量级策略（`moq_*` / `no_qty_*`）单独应用

**5.18.4 成本红线**

每个 QuoteOption 都要校验：`final_price >= cost_price * (1 + min_margin)`。
不满足 → 不输出该方案 + 自动转销售人工。
min_margin 在配置里按品类区分。

### 5.19 Inventory Matcher（v6 新增）

**5.19.1 多源库存**
查询输入：(category, grade, spec, qty_demand, dest_city?)
查询输出：候选库存数组（仓库、批次、库存量、属性、基价、距离/运费）

**5.19.2 组合算法**

| 模式 | 适用 |
|---|---|
| **单源最便宜** | 量小 / 客户偏低价 |
| **多源拼单** | 量大 / 单仓不足 / 拼后更便宜 |
| **品质优先** | profit 客户 |
| **就近优先** | 客户偏交付速度 |

输出多个 QuoteOption 时按"客户类型推荐顺序"排列：
- volume → 最低价在前
- profit → 优质品在前
- new → 中等价位 + 标准品在前

**5.19.3 含运费总成本**

报价默认含到达指定目的地的运费（基于仓库距离表）。差异化展示：
- "到武汉江夏含运 ¥3,820"
- "或自提武汉江夏 ¥3,790（省 ¥30 运费）"

### 5.20 Conversion Behavior Tracker（v6 新增）

**5.20.1 漏斗指标**

```
INQUIRY → QUOTED → LOCKED → ORDERED → SHIPPED → PAID
   |         |        |         |          |        |
   v         v        v         v          v        v
inquiry_cnt  quote_cnt lock_cnt deal_cnt   ship_cnt pay_cnt
```

按 7d / 30d / 90d 滚动统计。

**5.20.2 关键派生指标**

| 指标 | 公式 | 用途 |
|---|---|---|
| 询价转化率 | deal / inquiry | 客户健康度 |
| 报价转化率 | deal / quote | 报价吸引力 |
| 锁价转化率 | deal / lock | 锁完不下的占比 |
| **Ghost Score** | 综合得分（见 5.20.3） | 白嫖判定 |
| 平均决策时长 | deal_at - inquiry_at 均值 | 客户犹豫度 |
| 询价波动度 | 月度询价次数的方差 | 客户稳定性 |

**5.20.3 Ghost Score 计算**

```
ghost_score =
  0.4 * (1 - conversion_rate_30d) +
  0.2 * (重复询同规格不成交次数 / 总询价) +
  0.2 * (锁价后未下单率) +
  0.1 * (横向比价显著标识：如询价后 1h 内又问"对方报多少") +
  0.1 * (与同类客户的平均水平相比)
归一化到 0~1
```

阈值：
- < 0.3：正常
- 0.3~0.5：有点活跃比价
- 0.5~0.7：偏白嫖（启动软干预）
- ≥ 0.7：典型白嫖（销售强介入）

**5.20.4 实时事件**
每次状态变化（quoted/locked/expired/ordered/cancelled）打点 → Kafka/Redis Stream → 实时更新指标 + 触发对应策略。

### 5.21 Soft Influence Module（v6 新增：行为干预 / 关系维护）

> 这是处理"白嫖客户"的关键模块。**核心理念**：不抱怨、不指责、靠**差异化体验**让客户自己悟。

**5.21.1 5 层渐进策略**

| 层 | 触发 | 动作 | 客户体感 |
|---|---|---|---|
| **L1 隐性配额** | 24h 内同规格重复询价 | 引用上次报价 + 温和提示 | "原来一样的我刚问过" |
| **L2 锁价缩短** | ghost_score ≥ 0.5 | 锁价从 24h → 6h | "怎么这次给的时间这么短" |
| **L3 阶梯隐藏** | ghost_score ≥ 0.5 | 不展示量阶梯优惠 | "怎么没看到批量价" |
| **L4 销售关怀** | ghost_score ≥ 0.6 OR 报价 5 次未成交 | 销售主动联系，话术：「最近为您报了几次，有什么顾虑可以聊聊」 | "原来销售这么关心我" + 微妙压力 |
| **L5 策略降档** | ghost_score ≥ 0.7 | 从"利型/量型策略"降为标准价；不再给唤回优惠 | "这次报价怎么没那么有竞争力了" |

**5.21.2 配套话术（极简，绝不抱怨）**

| 场景 | 话术（差异化版本） |
|---|---|
| L1 重复询价 | "📌 此规格 24h 内已为您报价 ¥3,792，剩余有效 4h。如需新方案请回复『重新报价』；如需调整请回复『改条件 XX』" |
| L4 销售关怀（销售视角脚本） | "张总最近询了几次咱们家的螺纹，是不是有些方案上还想再对比？我看看哪里能帮您再调整下" |
| L5 沉默降档 | （不主动告知，靠回执差异让客户感知；如客户问"价格怎么涨了"→销售解释市场波动） |

**5.21.3 反向激励（让"勤下单客户"明显更爽）**

| 客户 | 体感差异 |
|---|---|
| 高转化客户 | 报价更快、锁价更长、阶梯更优、销售优先排队、可见"VIP 标识" |
| Ghost 客户 | 上述全部反向 |

**关键**：把"白嫖代价"和"忠诚红利"做成**可对比的体验差**，而不是"惩罚"。客户感受到的不是被指责，而是"原来勤下单的客户被这样对待"。

**5.21.4 客情防火墙**

- L1~L3 全自动，话术不带任何指责字眼
- L4 销售介入前，Bot 先告知销售客户画像 + 建议话术
- L5 永远不向客户公开"ghost_score"或"你被降档了"
- 销售可一键豁免任一层（带原因记录）
- 客户连续 30 天活跃下单 → ghost_score 自动下降 + 恢复优待

### 5.22 Tool Registry（v6 更新）

| 工具 | 入参变化 |
|---|---|
| `parse_inquiry` | 同 v5 |
| `submit_inquiry` | 同 v5 |
| **`compose_quote`** *(v6 新增)* | inquiry_item, customer_id → 返回 QuoteOption[] + strategy_trace；仅在 Auto/Assisted 档位调用 |
| **`confirm_quote`** *(v6 新增)* | option_id（销售确认 Assisted 报价；或客户回"确认"准备下单） |
| `query_inquiry_status` | 同 v5 |
| 其他工具 | 同 v5 |

**QuoteOption v6 结构**：

```json
{
  "option_id": "OPT-...",
  "inquiry_item_id": "...",
  "label": "推荐方案 / 标准方案 / 优质方案",
  "inventory_plan": [
    {"warehouse":"武汉江夏", "qty":30, "base_price":3820, "attrs":"标准"},
    {"warehouse":"襄阳",     "qty":20, "base_price":3750, "attrs":"长锈"}
  ],
  "freight_per_ton": 30,
  "blended_unit_price": 3792,
  "total_amount": 189600,
  "lock_minutes": 360,
  "lock_expires_at": "2026-05-28T22:00:00+08:00",
  "strategy_trace": [
    {"rule":"volume_driven_lowest", "effect":"select_inventory=best_blended_price"},
    {"rule":"explorer_throttle",    "effect":"lock_minutes_override=360"}
  ],
  "tags": ["拼仓","含运","软干预-锁价缩短"],
  "notes_to_customer": "...",
  "notes_to_sales":    "客户 ghost=0.62 偏高，建议人工跟进",
  "moq_ok": true,
  "above_cost_redline": true,
  "tier": "Assisted"
}
```

### 5.23 Storage（v6 新增表）

| 表 | 用途 |
|---|---|
| `customer_profile` | 画像主表 |
| `customer_profile_history` | 画像变更历史（含手动覆盖） |
| `customer_blacklist` / `whitelist` | 名单 + 等级 + 失效时间 |
| `pricing_strategy_config` | 策略规则（YAML/JSON） |
| `pricing_strategy_version` | 策略版本与变更审计 |
| `quote_option` | 生成过的报价方案 |
| `strategy_trace` | 每次报价应用了哪些规则 + 数据 |
| `inventory_snapshot` | 报价时点库存快照（争议追溯） |
| `conversion_event` | 漏斗事件流 |
| `ghost_score_history` | ghost 分数变化 |
| `soft_influence_log` | 触发了哪层干预、客户后续行为 |

---

## 6. 钢铁贸易话术与体验（v6 强化）

### 6.1~6.3 同 v5
略。

### 6.4 客户分层话术模板（v6 新增）

#### 新客（首次询价）
```
🎉 欢迎首次询价！本次为您提供新客特批：
HRB400 螺纹钢 Φ25mm × 50t（沙钢，武汉到货）
¥3,810/吨（含税含运，首单 -10）
有效期 6 小时
如需下单或调整方案，回复『继续』或联系销售小张：13xxxx
```

#### 量型客户
```
📊 您的询价方案：
HRB400 螺纹钢 Φ25mm × 50t（武汉到货）
推荐方案（拼仓）：均价 ¥3,792/吨  总价 ¥189,600
量阶梯：≥100t 再优 ¥20/t；≥200t 再优 ¥35/t
有效 24 小时
回复『下单』或『调整数量』
```

#### 利型客户
```
✨ 为您匹配的优质方案：
HRB400 螺纹钢 Φ25mm × 50t（武汉江夏 优级标准品）
单价 ¥3,820/吨（含税含运）
材质书 + 一炉一证 + 优质短锈
有效 24 小时
```

#### 战略客户
```
🤝 战略合作专属价：
HRB400 螺纹钢 Φ25mm × 50t
¥3,775/吨（战略价表）
锁价 72 小时
如需备货请提前 24h 告知
```

#### 流失客户（唤回）
```
👋 好久不见，本次专为您匹配：
HRB400 螺纹钢 Φ25mm × 50t
¥3,762/吨（唤回价 -30）
12 小时内有效
销售小张稍后会联系您，看看是哪里没合作好
```

#### 黑名单（uplift）
```
您的询价方案：
HRB400 螺纹钢 Φ25mm × 50t
¥3,935/吨（现款现货）
2 小时有效
说明：因账期原因暂按现款，恢复正常后回到标准价
```

#### MOQ 不足（不过磅）
```
您的询价方案：
HRB400 螺纹钢 Φ25mm × 20 支（不足过磅起订量 30t）
¥920/支（按支销售，不过磅）
有效 24 小时
如凑齐 30t 起按吨过磅价 ¥3,820/吨
```

#### 纯询价（无量）
```
当前指导价（仅供参考）：
HRB400 螺纹钢 Φ25mm（沙钢，武汉到货）
约 ¥3,820/吨
实际成交以您提供数量、目的地及提货时为准
有效 1 小时
```

#### 重复询价（L1）
```
📌 此规格 24h 内已为您报价 ¥3,792/吨（剩余有效 4h）
如需新方案：回复『重新报价』
如需续期：回复『续期』（由销售审批）
如需调整：回复『改条件 XX』
```

#### 软干预（L4 销售脚本，仅销售可见）
```
[系统建议销售话术]
客户：张总（量型/普通/ghost=0.62）
建议联系：今日 15:00 后
切入：「张总最近询了几次咱家螺纹，是不是有方案上想再对比一下？
       我看看是规格还是产地能再帮您调一调」
不要说：「您一直询价没下单」「白嫖」
```

---

## 7. 安全与合规（v6 增量）

- **价格策略配置变更**全量审计 + 二人复核（避免改错策略导致赔本）。
- **客户画像、ghost_score、soft_influence_log** 严禁泄漏给客户；销售看到的内容也要按角色权限可见。
- 策略 trace 留存 1 年，纠纷追溯用。
- 黑/白名单变更必须有原因 + 操作人 + 审批。

---

## 8. 可观测性（v6 增量）

- 报价指标：
  - `quote_total{tier}` (Auto/Assisted/Manual)
  - `quote_strategy_applied{rule}`
  - `quote_to_deal_rate{customer_type}`
  - `quote_below_cost_blocked_total`（成本红线拦截）
- 干预指标：
  - `soft_influence_trigger{level}`
  - `ghost_score_distribution`
  - `repeat_inquiry_suppressed_total`
- 销售视角：
  - 一键确认率、平均确认时长、销售调整幅度

---

## 9. 部署 / 10. 里程碑（增量）

里程碑新增（追加到 v5）：

| 里程碑 | 交付物 |
|---|---|
| **M21 Customer Profile Engine** | 客户画像表 + 自动分类规则 + 销售手动覆盖 UI |
| **M22 Conversion Tracker** | 漏斗事件流 + 关键指标 + Ghost Score |
| **M23 Inventory Matcher** | 多源查询 + 组合优化 + MOQ 校验 + 运费 |
| **M24 Pricing Strategy Engine** | 策略库 + 配置后台 + 成本红线 + 策略 trace |
| **M25 三档协作 + Bot UI** | Auto/Assisted/Manual 判定 + 销售一键确认卡片 |
| **M26 Soft Influence** | 5 层渐进式干预 + 差异化话术 + 反向激励 + 客情防火墙 |
| **M27 报价话术模板** | 按客户分层的回执模板 + 销售脚本 |
| **M28 策略灰度上线** | 先 Auto 仅小金额 + 白名单；逐步开 Assisted；全量 |

---

## 11. 风险与对策（v6 增量）

| 风险 | 影响 | 对策 |
|---|---|---|
| Auto 报价赔本 | 资损 | 成本红线必过 + 单笔金额上限 + 异常报价拦截器 |
| 策略配置改错 | 大面积错价 | 二人复核 + 灰度发布 + 自动回滚 |
| Ghost 误伤好客户 | 客情受损 | 评分阈值保守 + 销售可豁免 + 客户重新活跃后自动恢复 |
| 客户察觉"被降档" | 客情危机 | 永远不公开评分；差异通过"市场行情"等中性话术解释 |
| 黑名单滥用 | 销售个人偏好导致客户冤枉 | 黑名单分级 + 强制理由 + 上级审批 + 30 天复盘 |
| 客户偏好与系统画像冲突 | 销售觉得系统不准 | 销售可手动覆盖；学习销售覆盖原因优化模型 |
| 库存数据不实时 | 报价后无货 | 报价附库存快照 + 短锁价 + 自动补货机制 |
| 同一客户多销售争抢 | 内部冲突 | 客户绑定销售 + 报价归属销售 + 销售调整审计 |
| 重复询价压制过度 | 客户不耐烦 | 提供"重新报价"出口 + 限频参数可调 |
| 阶梯优惠泄漏给非目标客户 | 价格穿帮 | 阶梯只在符合条件客户回显；公开渠道不暴露 |
| 软干预触发频繁 | 客户疲劳 | 30 天冷却 + 每客户每周最多触发 N 次 |

---

## 12. 后续演进
- LLM Function Calling 加入 `propose_quote` 工具，让 LLM 在多轮里组织报价话术（但价格仍来自 Pricing Engine）。
- 客户行为预测：用历史数据预测"客户下次下单概率"，提前推送优惠。
- 销售业绩看板 + Bot 自动周报。
- 报价多版本对比（A/B 不同策略）。
- 行业行情联动：钢联/Mysteel 大盘动 → 自动调整基价。
- LoRA 微调 + 客户专属模板。

---

## 13. 版本演进对比

| 维度 | v3 | v4 | v5 | **v6** |
|---|---|---|---|---|
| 询价输入 | 文字 | 文字/图/Excel/PDF | 同 | 同 |
| 询价字段 | 5 | 7 | 9 | 同 |
| NLU 策略 | LLM | LLM | KB + 默认推断 + 字段级溯源 | 同 |
| **报价** | **销售单干** | **销售单干** | **销售单干** | **三档协作（Auto/Assisted/Manual）** |
| **客户画像** | — | — | 偏好画像（询价默认值） | **完整画像引擎（5 类型 + 名单 + 信用 + 行为 + Ghost）** |
| **库存匹配** | — | — | — | **多源 + 组合优化 + 运费 + MOQ** |
| **报价策略** | — | — | — | **可配置策略引擎（11+ 策略 + 成本红线）** |
| **行为干预** | — | — | — | **5 层渐进式 + 差异化话术 + 反向激励** |
| **MOQ / 不过磅** | — | — | — | **品类 MOQ 表 + per_piece 模式** |
| **纯询价** | — | — | — | **指导价 + 极短锁价** |
| **黑白名单** | — | — | — | **分级（上浮/现款/预付/拒报）** |

---

## 14. 附录：报价场景样例（v6 新增）

| 场景 | 客户画像 | 输出策略 | 锁价 | 客户体感 |
|---|---|---|---|---|
| 新客首次询 50t 螺纹 | new + normal + ghost=0.1 | new_customer_attractive | 6h | 欢迎话术 + 首单 -10 |
| 量型老客询 80t | volume + whitelist + ghost=0.2 | whitelist_discount + volume_driven_lowest | 24h*1.5=36h | 拼仓 + 阶梯优惠 + 长锁价 |
| 利型客户询 30t | profit + normal + ghost=0.15 | profit_quality_first | 24h | 优质短锈 + 质保 + 中等价 |
| 黑名单上浮级 | volume + blacklist(uplift) | blacklist_uplift | 2h | 上浮 3% + 现款 |
| 流失客户回归询价 | churn + normal | churn_callback | 12h | 唤回价 -30 + 销售跟进 |
| 频繁询价不下单 | volume + normal + ghost=0.65 | volume_driven_lowest + explorer_throttle | 6h（被覆盖） | 阶梯隐藏 + 短锁价 + 销售关怀 |
| 同规格 24h 内重复 | 任意 + 命中 L1 | repeat_inquiry_quote_reuse | 复用 | "已为您报过" |
| 询价 25t（MOQ 30t） | 任意 | moq_unweighed | 24h | 按支报价 |
| 询价无数量 | 任意 | no_qty_indicative | 1h | 指导价 |
| 超大单 600t | volume + whitelist + 大额触发 Manual | Manual | 销售决定 | "销售小张为您专项跟进" |

---

## 15. 待需求方确认

1. 业务系统能否开发 Inbound Webhook？
2. 业务 API 是否支持创建询价（9 要素 + 批量）、查库存（多源/属性/库位）、提交报价方案？
3. 询价 ERP 报价流：逐 item 还是整单？
4. 报价回推是否附 PDF？
5. 询价 9 要素 ERP 必填/可空？
6. **客户画像主权归属：Bot 维护 vs 同步 CRM？冲突优先级？**（v6 关键）
7. **价格策略配置后台：是否需要？谁来维护（运营/销售总监/老板）？**（v6 关键）
8. **客户分类规则：是否需要按公司业务调整默认阈值（新客 90 天？P75 量？）**
9. **品类 MOQ 表**：每个品类的最低过磅量是多少？
10. **黑名单分级阈值**：上浮多少、什么情况进 cash_only、什么时候 refuse？
11. **战略客户名单**由谁维护？变更审批流？
12. **成本价数据源**：是否能实时从 ERP 拉？还是用每日快照？
13. **Auto 档位金额上限**：单笔多少以下才能 Auto 直接报？
14. **Ghost Score 阈值**和触发软干预的细则，需求方是否需要自己调？
15. 付款凭证 OCR、多 sheet 策略、文件保留、VL 预算、SLA、客户绑定方式、群聊场景（同 v5）
