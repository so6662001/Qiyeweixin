# 模块设计（第 5 章）

> 摘自 `DESIGN.md` 拆分。返回 [总览](../DESIGN.md)。

## 5. 模块设计（v7 增量）

### 5.1~5.23 同 v6
略。

### 5.24 Negotiation Session Manager（v7 新增）

**5.24.1 状态机**

```
                         ┌───────────────┐
                         │   OPENING     │  Bot 已发首报价，等客户响应
                         └───────┬───────┘
                                 │ 客户接受
                                 ▼
                         ┌───────────────┐
                         │   AGREED      │  进入下单流程
                         └───────────────┘
                                 ▲
                                 │ 客户接受
   ┌───────────────┐ 客户出价     │
   │  COUNTERED    │──────────────┤
   │  客户已反报价  │              │
   └───────┬───────┘              │
           │ 我方回让步             │
           ▼                       │
   ┌───────────────┐              │
   │   CONCEDED    │──────────────┤  客户接受
   │  我方已让步    │
   └───────┬───────┘
           │  客户继续砍 / 超阈值
           ▼
   ┌───────────────┐     ┌───────────────┐
   │   IMPASSE     │     │  ESCALATED    │
   │   僵持        │     │  转人工        │
   └───────────────┘     └───────────────┘
```

**5.24.2 数据结构**

```json
NegotiationSession {
  session_id, topic_id, inquiry_id,
  customer_id, sales_id,
  state: OPENING/COUNTERED/CONCEDED/AGREED/IMPASSE/ESCALATED,
  rounds: [
    {
      round_no: 1,
      direction: "system_offer" | "customer_offer",
      timestamp,
      payload: { item_id, price, qty, terms, raw_msg },
      strategy_applied: ["volume_driven_lowest"],
      concession_amount: 0 | 15 | ...,
      generated_options: [...],         // 当时给销售的方案
      sales_chosen: "A",                // 销售选了哪个
      llm_reply_template: "...",
      delta_from_last_round: -15
    },
    ...
  ],
  budget: {
    item_id, max_concession, total_spent, remaining
  },
  competitor_mentions: [
    { round, name, price, tone, verified: bool }
  ],
  emotion_track: ["neutral","strong","irritated"],
  result: AGREED/IMPASSE/ESCALATED,
  closed_at
}
```

**5.24.3 会话生命周期**

- 创建：Bot 发出首报价时创建（OPENING）
- 推进：每次 customer 或 system 出价 → 追加 round
- 关闭：AGREED（转下单）/ IMPASSE（保留 24h 复活）/ ESCALATED（销售接管）
- 跨天恢复：跟 Topic 一致，跨天议价合法（行情未变时）

### 5.25 Concession Strategy Engine（v7 新增）

**5.25.1 让步预算**

每个 QuoteOption 创建时由 Pricing Engine 分配：

```
budget = current_price - max(cost_price * (1 + min_margin), price_floor_per_category)
budget_pace = 客户类型决定（求利型 0.5/0.25/0.1/0.05；求量型主要靠量；战略型一次性 0.7）
```

**5.25.2 让步曲线（按客户类型）**

```yaml
concession_curves:

  profit_seeker:           # 求利型
    description: 慢让、递减、末端价值替代
    rounds:
      - { round: 1, type: price, ratio: 0.50 }
      - { round: 2, type: price, ratio: 0.25 }
      - { round: 3, type: price, ratio: 0.10 }
      - { round: 4, type: value_add, ratio: 0 }   # 转赠送/质量
      - { round: 5, type: escalate }

  volume_seeker:           # 求量型
    description: 量价绑定为主，价让步小
    rounds:
      - { round: 1, type: qty_bundle, hint: "加 20t 再让 ¥20" }
      - { round: 2, type: price, ratio: 0.20 }
      - { round: 3, type: qty_bundle + price, ratio: 0.20 }
      - { round: 4, type: escalate }

  strategic:               # 战略型
    description: 一次给到，换长期承诺
    rounds:
      - { round: 1, type: price, ratio: 0.70 }
      - { round: 2, type: time + term }            # 锁价 + 账期
      - { round: 3, type: escalate }

  new_customer:            # 新客
    description: 首报价已含优惠，不缠斗
    rounds:
      - { round: 1, type: gift }                   # 赠送材质书/优先发
      - { round: 2, type: escalate }

  churn:                   # 流失客户
    description: 一次给到底
    rounds:
      - { round: 1, type: price, ratio: 0.80, hint: "唤回价已包含" }
      - { round: 2, type: escalate }
```

**5.25.3 关键原则**

- **让步递减**：每轮让步幅度递减，避免被无限砍
- **末端价值替代**：让到一半后转价值替代（赠/时/质）
- **不破红线**：成本 + 最低毛利 = 硬底线，永不破
- **不轻易显示底牌**：剩余预算只给销售看，不告诉客户

### 5.26 Order-level Profit Optimizer（v7 新增）

**5.26.1 整单视图**

```json
OrderProfitView {
  order_draft_id,
  items: [
    { item_id, qty, price, cost, margin_per_t, headroom },
    ...
  ],
  total_revenue, total_cost, total_margin, blended_margin_rate,
  bottleneck_item_id: "毛利最薄的 item",
  fattest_item_id:    "毛利最厚的 item"
}
```

**5.26.2 跨项让步分配算法**

输入：客户砍价 X 元在 item-K 上
输出：1~2 个分配方案

```
方案 A "单点让"：
  item-K 让 X1 (X1 = 让步预算允许的最大，但 ≤ X)
  其他 item 保价
  适合：求利型，客户视觉聚焦

方案 B "转嫁让"：
  item-K 保价（headroom 已薄）
  fattest_item 让 Y，让 item-K 客户得到的整单优惠 ≈ X*qty
  适合：item-K 毛利已薄；客户接受"整单看"

方案 C "分散让"：
  每个 item 都让 1~5 块
  整单累计 ≈ X*qty
  适合：求量型（也表现"都让了"）

方案 D "整单一口价"：
  直接给整单总价 -¥Y
  Y = X*qty - 一点
  适合：战略客户
```

**5.26.3 整单底线（v14 按客户 segment 差异化）**

整单加权毛利率 / 每吨毛利 < min_blended_margin → 拒绝继续让步 → 转销售。

```yaml
# v14 客户细分驱动的整单毛利底线（可配置）
order_profit_floor:
  by_segment:
    dealer:       # 下级经销商（以挂牌价为成本核算）
      min_margin_per_ton: 10
      cost_basis: list_price       # 用挂牌价做成本基线
    terminal:     # 终端企业
      min_margin_per_ton: 200
      cost_basis: actual_cost      # 用实际进货成本
  per_customer_override:           # 个别客户可单独配
    cust_xxx: { min_margin_per_ton: 50 }
  per_category_modifier:           # 品类微调（如不锈钢更高底线）
    不锈钢: × 1.5
```

整单 Optimizer 流程调整：
1. 取客户 segment（dealer / terminal）
2. 取 cost_basis（list_price 或 actual_cost）
3. 算 min_margin = floor(segment) × category_modifier
4. 整单加权毛利 < min_margin → REJECT
5. 否则按 5.26.2 跨项让步分配

### 5.27 Counter-offer Generator（v7 新增）

**5.27.1 8 维让步货币定义**

```yaml
concession_currencies:

  price:
    cost_per_unit: 1.0         # 等价系数
    customer_sensitivity: 1.0
    unit: 元/吨
    constraint: 不破成本红线

  qty:
    cost_per_unit: -0.3        # 加量边际增本低
    customer_sensitivity: 0.8
    unit: 吨
    constraint: 库存允许 + 客户能接

  time:
    cost_per_unit: 0.05/小时   # 行情风险折算
    customer_sensitivity: 0.4
    unit: 小时

  freight:
    cost_per_unit: 1.0         # 按实际运费等价
    customer_sensitivity: 0.6
    unit: 元/吨
    constraint: 配送范围内

  term:                        # 账期
    cost_per_unit: 0.03/天     # 资金成本
    customer_sensitivity: 0.5
    unit: 天
    constraint: 信用额度允许

  quality_upgrade:
    cost_per_unit: 10~50       # 升级到优级品
    customer_sensitivity: 0.3
    unit: 元/吨
    constraint: 库存有

  gift:                        # 赠送（材质书/优先发车/优先供应）
    cost_per_unit: ~0
    customer_sensitivity: 0.2
    constraint: 不滥用

  bundle:                      # 整单打包
    cost_per_unit: 看具体方案
    customer_sensitivity: 0.7
    unit: 元/单
```

**5.27.2 等价组合搜索**

```
目标：客户砍 ¥30/t × 50t = ¥1,500 让步

枚举 1~3 维组合（避免方案太复杂客户算不过来）：
  - {price: -15} → 等价 ¥750（够吗？看客户类型）
  - {price: -10, time: +24h} → 实际让 ¥10*50 + 时间风险
  - {price: -10, freight: -20} → 实际 ¥1,500
  - {qty: +20, price_on_total: -20} → 量价绑定
  - {bundle: -1500} → 整单一口让

按客户类型 + sensitivity 排序，取 Top-2~3 给销售
```

**5.27.3 输出格式**

每个 counter-offer 含：
- 摘要："让 ¥10/t + 免运费 ≈ 等价让 ¥30/t"
- 详细成本（销售看的，客户看不到）
- 客户视角话术草稿
- 估计的客户接受概率（基于历史）

### 5.28 Bargaining LLM Layer（v7 新增）

**5.28.1 Prompt 上下文**

```
System: 你是钢铁贸易资深销售助手，正在帮销售小张回复客户议价。
       客户：张总，求利型，普通，半年合作 5 次。
       本次议价：第 2 轮。
       已让步：¥15/t。剩余预算：¥30/t（不告诉客户）。
       本次销售选定方案：让 ¥10/t + 锁价延长 24h。

要求：
  - 礼貌、不卑不亢
  - 不要直接说"还有让步空间"或"老板还能批"（暴露底牌）
  - 把价值替代（锁价延长）说出价值
  - 留下"如果加量还能进一步谈"的钩子（求利型 + 量绑定预热）
  - 不超过 120 字
  - 不出现"白嫖"等不当词

Few-shot: 3 个真实议价话术正面/反面例子

User: 客户最新原话："3780 我才下"
```

**5.28.2 输出示例（典型话术）**

```
"张总，听您的，沙钢 HRB400 25mm 这批我帮您再让 10 块，到 ¥3,805/吨，
锁价延长到 48 小时给您慢慢决定。这批长沙钢厂直发，发货也优先。
如果方便加到 80 吨，我再帮您和老板沟通一下。"
```

**5.28.3 安全约束**

- LLM **绝不输出** 价格、库存、订单号等关键数字（这些从 Counter-offer 结构里直接渲染，LLM 只填话术包装）
- LLM 绝不"许愿"（不能说"我跟老板申请下"暗示有让步空间）除非销售明确批
- 后置 DLP 扫敏感词（如"成本"、"底价"、"老板"、"亏本"）

**5.28.4 客户情绪检测**

LLM 顺带做情绪标注（neutral/strong/irritated/threatening）：
- irritated/threatening → 立即转销售
- strong → 提示销售关注
- neutral → 继续 Bot/Assisted

### 5.29 Tool Registry（v7 增量）

| 工具 | 入参 | 说明 |
|---|---|---|
| `negotiate_round` | session_id, customer_offer | 推进一轮议价，返回推荐 counter-offers |
| `accept_offer` | option_id | 客户/销售接受 |
| `escalate_negotiation` | session_id, reason | 转人工 |
| `propose_order_bundle` | inquiry_id, customer_id | 整单方案生成 |
| `get_negotiation_history` | session_id | 查议价历史 |

### 5.30 Storage（v7 新增表）

| 表 | 用途 |
|---|---|
| `negotiation_session` | 议价会话主表 |
| `negotiation_round` | 每轮 offer 详情 |
| `negotiation_budget` | 让步预算与消耗 |
| `concession_curve_config` | 让步曲线配置（YAML） |
| `competitor_mention` | 客户提的竞品（用于判断真假） |
| `bargaining_prompt_log` | 议价 LLM 调用日志 |
| `order_profit_view_snapshot` | 整单视图快照（议价时点） |

### 5.31 Quote Lifecycle State Machine（v8 新增）

**5.31.1 状态机**

```
                  ┌─────────┐
                  │  DRAFT  │  Pricing Engine 刚生成
                  └────┬────┘
                       │ submit
                       ▼
                  ┌─────────┐
                  │ ACTIVE  │  Bot 已发客户
                  └──┬─┬─┬──┘
                     │ │ │
       amend ────────┘ │ └──── lock
          ▼            │       ▼
   ┌──────────┐        │  ┌─────────┐
   │AMENDING  │        │  │ LOCKED  │
   └────┬─────┘        │  └────┬────┘
        │ rescore      │       │ amend (须销售批)
        ▼              │       ▼
   ┌──────────┐        │  ┌─────────────────┐
   │RESCORED  │        │  │ LOCKED+AMENDING │
   └────┬─────┘        │  └─────────────────┘
        │ accept       │
        ▼              │  expire
   ┌──────────┐        ▼
   │ACTIVE v+1│   ┌─────────┐
   └──────────┘   │EXPIRED  │
                  └─────────┘

任何状态 + supersede ─→ SUPERSEDED（被新版本替代）
任何状态 + close     ─→ CANCELLED / AGREED
```

**5.31.2 版本化原则**
- **不覆盖**：每次改量生成新 version（v1/v2/v3…），原版本保留为 SUPERSEDED
- **可对比**：客户始终能看到「原方案 vs 新方案」差异
- **可回滚**：销售在异常时可手动让客户回到上一版本（带原因）
- **可追溯**：所有版本与变更原因落 `quote_version_log`

**5.31.3 锁价后子状态**
LOCKED + AMENDING 必经销售审批：
- 销售批准 → LOCKED v+1
- 销售拒绝 → 退回 LOCKED v1
- 解锁后重谈 → 原锁价作废

### 5.32 Quote Amendment Engine（v8 新增）

**5.32.1 输入 / 输出**
```
Input:
  quote_id, version
  amendment_request:
    field, value_from, value_to, attached_terms, source
  context: NegotiationSession?, OrderProfitView?, profile, history

Output: AmendmentDecision {
  decision: AUTO | ASSISTED | CUSTOMER_CHOICE | MANUAL | REJECT
  new_quote_version: QuoteOption
  diff_view, customer_choice_options?, sales_options?, reasons
  side_effects: profile_update, workflow_action, audit_log
}
```

**5.32.2 决策树（v14 细化锁价后规则 + 减量退价分档）**
```
═══ 锁价前 / 普通改量 ═══
amend.delta_pct
  ≤ 10%
     ↓ + 合规全过 + 非锁价 + 非议价
     → AUTO
  10~30%
     ↓ 阶梯档位变 / 拼仓变 / 库存吃紧 / 整单接近底线
     任一 → ASSISTED ; 都未 → AUTO
  > 30%
     ↓
     new_qty < MOQ          → CUSTOMER_CHOICE
     new_qty 超库存上限     → MANUAL（含转人工拆批确认 / 整批二选一）
     new_qty 超 Auto 上限   → ASSISTED 或 MANUAL（看金额）
     new_qty == 0           → REJECT + 撤单（**v14：需客户额外确认 + 写入 ghost 行为**）
     其余                   → ASSISTED

═══ 锁价后（LOCKED）v14 细化 ═══
单次幅度  ≤ ±5%   AND  累计幅度 ≤ ±10%  → AUTO（小幅放行，无需审批）
单次 ≤ ±5%       AND  累计 > ±10%       → ASSISTED（销售确认）
单次 > ±5%       AND  累计 > ±10%       → MANUAL（主管审批）
改规格/产地/目的地                       → MANUAL（原锁价作废，新询价）

═══ 减量退价档位（v14 新，不一刀切）═══
减量幅度    单价处理
  ≤ 5%      保持原单价（锁价承诺）
  5~15%     回到无量阶梯标准档
  15~30%    回到无量阶梯 + 加 ¥10/t（小批量成本）
  > 30%     必须 ASSISTED（销售判断）+ 重新整单 Optimizer 校验

═══ 修饰 ═══
  议价中改量            → 不走本流程，转 4.8（**不计入议价轮次**）
  改量次数 ≥ 3          → MANUAL + 软干预 L4
  整单底线破            → MANUAL（按 5.26 客户 segment 查底线）
  锁价后 + 任意改量      → 走"锁价后"分支
```

**5.32.3 模块协同**
| 模块 | 协同方式 |
|---|---|
| Pricing Strategy Engine | 重新定价；strategy_trace 追加 amendment 节点 |
| Inventory Matcher | 新量驱动重新组合 |
| Order-level Profit Optimizer | 整单底线再校验 |
| Negotiation Session Manager | 议价中改量 → NegotiationRound offer |
| Customer Profile Engine | 改量频次/接受率写入 behavior |
| Soft Influence | ≥ 3 次改量 → 升 L4 |
| Workflow Engine | 锁后改量、库存不足 → 创建审批工单 |

**5.32.4 频次防护**
```yaml
amendment_guardrail:
  per_quote_max: 3
  per_session_warning: 5
  per_day_per_customer_max: 10
  cooldown_after_reject_minutes: 10
  exception:
    - 议价中改量不计入
    - 销售代客户改量不计入
```

**5.32.5 新增存储**
| 表 | 用途 |
|---|---|
| `quote_version` | 每个 Quote 的多版本 |
| `quote_amendment_log` | 改量请求 + 决策 + 结果 |
| `quote_version_log` | 版本变更原因审计 |
| `lock_amendment_approval` | 锁后改量审批 |

### 5.33 Lead Routing Engine（v10 新增）

**5.33.1 输入输出**
```
HandoffRequest {
  trigger_type, customer_id, inquiry_id?, quote_id?,
  urgency, category, qty, dest_city,
  conversation_excerpt, customer_profile_snapshot,
  required_skills, prefer_bound
}
   ↓
RoutingDecision {
  assigned_sales_id, fallback_chain: [sales_id, ...],
  routing_reason, weights_snapshot, T_used,
  sla_ack_deadline, sla_reply_deadline
}
```

**5.33.2 决策算法**
```python
def route(req):
    # 第一层 绑定销售
    if req.prefer_bound:
        bound = customer.bound_salesperson_id
        if bound and is_available(bound) and has_skill(bound, req.category):
            return assign(bound, reason='bound')
    
    # 第二层 团队池筛选
    pool = filter_pool(req)
    if not pool:
        pool = relax_filter(req)  # 放宽：允许 busy
    if not pool:
        return route_to_team_group(req)  # 推群 + 主管认领
    
    # VIP 强制
    if customer.type in ['strategic', 'vip']:
        pool = [s for s in pool if s.vip_eligible]
    
    # 加权抽签
    weights = softmax([s.score for s in pool], T=config.routing_T)
    selected = weighted_pick(pool, weights)
    selected.daily_quota_used += 1
    selected.workload += 1
    return assign(selected, reason='pool_weighted')
```

**5.33.3 关键配置**
```yaml
routing:
  T: 1.0                         # 温度参数（小=精英拿大头）
  workload_max: 15               # 每销售最大未完成工单数
  daily_new_lead_quota: 8        # 每日新单（未绑定客户）配额
  ack_deadline_minutes: 5
  reply_deadline_minutes: 30
  decline_penalty: -2 (score)
  ack_timeout_penalty: 0 (不罚)
  vip_eligible_min_score: 80
  newcomer_floor: 0.1            # 底部 30% 销售保底概率不低于此
```

**5.33.4 SLA + 转派**
```
T+0    推工单
T+5min 未 ACK → 自动转下一位（原销售不扣分）
T+15min 已 ACK 未回客户 → 提醒
T+30min 仍未回 → 升级主管 + 客户安抚
T+60min 严重超时 → 团队负责人介入
```

**5.33.5 异常路径**
| 情况 | 处理 |
|---|---|
| 池为空 | 放宽筛选 → 推群认领 → 值班销售兜底 |
| 全员忙 | 客户排队 + 预估等待时间 + 安抚话术 |
| 销售离职 | 自动解绑 + 通知 CRM + 客户后续走池 |
| 突发情绪客户 | 跳过抽签直接转资深销售 |
| 客户指定销售 | 优先满足；该销售不可用时反问客户 |

### 5.34 Sales Performance Score（v10 新增）

**5.34.1 评分公式**

```
score = 100 *
        Σ wᵢ * normalize(metricᵢ, polarity)

其中：
  conversion_rate     w=0.25  正向
  monthly_volume      w=0.20  正向
  dso_days            w=0.15  反向（越短越好）
  nps                 w=0.10  正向
  avg_response_time   w=0.15  反向
  decline_rate        w=0.10  反向
  current_workload    w=0.05  反向

权重可在管理后台调整。
normalize: min-max 归一到 [0,1]，按团队分布或全公司分布
```

**5.34.2 更新频率**
- 实时事件：成交/拒单/客户消息触发，更新对应指标
- 每小时增量重算（用过去 30 天滑窗）
- 每天全量重算
- 历史保留 `sales_score_history` 用于复盘

**5.34.3 团队排名**
- 团队内 leaderboard：实时刷新
- 全公司排名（销售可选择是否对自己可见，主管可见全部）
- 周/月/季度排名变化追踪
- 销售可见自己分数 + 排名（激励），**客户不可见**

**5.34.4 防作弊**
- 拒单率超过 X% → 主管复盘
- 异常高的转化率 → 抽查是否有"假绑定"
- 拒接好客户改接坏客户 → 异常告警
- 数据修改全审计 + 主管授权才能改

### 5.35 Sales State Manager（v10 新增）

**5.35.1 状态维度**

```yaml
SalesState:
  online_status: online | busy | away | offline | leave
  workload:
    current_active_workflows: 12
    today_new_leads: 5
    daily_quota_remaining: 3
  schedule:
    on_duty: true
    shift_until: "2026-05-28T18:00"
    out_of_office_until: null
  authorization:
    categories: [螺纹, 中厚板, H 型钢]
    can_handle_blacklist: false
    can_handle_strategic: true
    max_single_order_amount: 500_0000
  vip_eligible: true   # 是否可接战略客户
  level: junior | mid | senior
```

**5.35.2 状态来源**
- 企业微信在线状态 API
- 销售在 Bot 内主动设置（"我在午休"）
- HR 系统的请假/排班数据
- 工单系统的负载实时计算

**5.35.3 配额管理**
- 每销售每日 `daily_new_lead_quota` 可调（按级别差异）
- 用完后该销售退出池抽签
- 主管可临时增加配额（带原因）

**5.35.4 战略客户特权**
- VIP 资格销售名单由主管/总监维护
- 战略客户的工单仅在 VIP 池中抽签
- 即使 VIP 池为空也不降级到普通池（必转主管）

**5.35.5 新增存储**

| 表 | 用途 |
|---|---|
| `sales_profile` | 销售主表（含权重、授权、级别） |
| `sales_state` | 实时状态（频繁更新） |
| `sales_performance_metric` | 各维度指标 |
| `sales_score_history` | 评分历史 |
| `lead_assignment_log` | 工单分派记录（含抽签快照） |
| `lead_decline_log` | 拒单审计 |
| `routing_config` | 路由配置（T、quota、权重） |

### 5.36 Substitute Knowledge Base（v11 新增）

**5.36.1 数据模型**

```yaml
SubstituteRelation:
  rel_id
  original:
    category, grade, spec, origin?, length?, standard?
  substitute:
    category, grade, spec, origin?, length?, standard?
  rel_type: 同档异厂 | 同档异定尺 | 同档品质差 | 向上兼容 | 向下兼容 | 规格相近 | 外标等价
  bidirectional: true | false           # 是否双向可替代
  recommendation_strength: strong | medium | weak | discouraged
  use_case_constraints:
    allowed_in: [非工程, 配件, DIY, 出口...]
    forbidden_in: [抗震建筑, 桥梁, 基础设施...]
  reason_for_customer: "永钢与沙钢均为国标 HRB400，可互换使用"
  reason_for_sales:    "毛利略高 ¥10/t；客户接受度高"
  audit:
    created_by, created_at, last_reviewed_at, examples: [...]
```

**5.36.2 内置规则集（钢铁贸易常见）**

```yaml
# 同档异厂（强推）
- HRB400 沙钢 ↔ HRB400 永钢
- HRB400 沙钢 ↔ HRB400 中天
- HRB400 沙钢 ↔ HRB400 萍钢
- Q235B 鞍钢 ↔ Q235B 首钢

# 同档异定尺（中推）
- 螺纹 12m ↔ 9m + 倍尺
- 板材 开平定尺 ↔ 卷板（开平加工费另计）

# 向上兼容（弱推）
- HRB400 → HRB500（贵 ¥150/t；强度更高）
- Q235 → Q355（贵 ¥200/t；强度更高）

# 向下兼容（谨慎）
- HRB400E → HRB400（去抗震要求；工程禁）
- Q355 → Q235（强度降；几乎所有场景不行）

# 同档品质差
- 标准品 ↔ 长锈轻锈（便宜 ¥80~150/t）
- 标准品 ↔ 优级（贵 ¥50~100/t）

# 规格相近（高风险，工程禁）
- Φ25 ↔ Φ22 (相近)
- Φ20 ↔ Φ18

# 外标等价（出口贸易）
- Q235B ↔ SS400 (JIS) ↔ A36 (ASTM) ↔ S235JR (EN)
- Q355B ↔ S355JR (EN)
```

**5.36.3 维护方式**

- 后台运营 UI：可视化添加/修改/审核替代关系
- 行业标准导入：GB/ASTM/JIS/EN 等价表
- 历史成交回流：销售标记"客户接受/拒绝" → 反哺规则置信度
- LLM 辅助审核：新增关系前 LLM 给出风险评估

**5.36.4 加载与查询**

- 全量加载到 Redis（按 original 组成 key）
- 查询：input (cat,grade,spec,origin,length) → 多条候选关系
- KB 变更走审计 + 二人复核（重要规则改错影响大）

### 5.37 Substitute Match Engine（v11 新增）

**5.37.1 输入 / 输出**

```
Input:
  - original_spec: {category, grade, spec, origin?, length?, standard?}
  - qty_required
  - dest_city
  - customer_profile (含 substitute_preferences + history)
  - use_case: 工程 | 配件 | DIY | 出口 | 未知（LLM 从对话抽）
  - context: in_negotiation? | in_amendment?

Process:
  Step 1: 查 KB 找候选关系列表
  Step 2: 用 use_case 过滤（forbidden_in）
  Step 3: 用 customer_preferences 过滤（rejects_types / 历史拒绝）
  Step 4: 对每个候选查库存 + 价格 + 运费
  Step 5: 计算 acceptance_score = f(用途, 客户类型, 历史, 关系强度)

Output:
  candidates: [
    {
      substitute_spec, relation_type, relation_reason,
      stock_available, base_price, price_diff,
      our_margin_diff, acceptance_score,
      caveat
    }, ...
  ]
  sorted by composite_score
```

**5.37.2 排序权重**

```yaml
ranking_weights:
  customer_acceptance: 0.30    # 接受度
  stock_sufficiency:   0.25    # 库存够不够
  recommendation_strength: 0.20  # 强/中/弱推
  price_advantage:     0.15    # 客户视角价格优势
  our_margin:          0.10    # 我方毛利
```

**5.37.3 接受度计算**

```python
def acceptance_score(rel, customer, use_case):
    s = 0.5  # 基础
    if customer.type == "trader_processor":  s += 0.2
    if customer.type == "construction":      s -= 0.3
    if rel.rel_type == "同档异厂":           s += 0.3
    if rel.rel_type == "规格相近":           s -= 0.4
    if customer.history_accepted(rel.rel_type, last_n=5):
        s += 0.2 * accept_ratio
    if use_case in rel.forbidden_in:         s = -1  # 直接禁
    return clip(s, 0, 1)
```

### 5.38 Substitute Recommendation Strategy（v11 新增）

**5.38.1 推荐时机决策**

```python
def should_recommend(stock_status, customer, in_negotiation, in_amendment):
    if customer.accepts_substitute == False:
        return False
    if stock_status == "OUT_OF_STOCK":
        return True
    if stock_status == "PARTIAL_STOCK":
        return True
    if stock_status == "FULL_STOCK":
        if in_negotiation:
            return True  # 议价让步货币
        if customer.type == "strategic":
            return maybe_proactive()  # 视配置
        return False  # 默认不主动推
```

**5.38.2 档位决策（与报价档位一致）**

| 决策 | 条件 |
|---|---|
| AUTO | 同档异厂 + 客户贸易商/加工厂 + 历史多次接受类似 |
| ASSISTED | 大多数情况，销售一键确认 |
| CUSTOMER_REVIEW | 客户 conditional，推时附"请确认是否接受" |
| MANUAL | 工程类用途 / 战略客户 / 大单 / 客户档案 conditional |
| NO_RECOMMEND | accepts_substitute=false → 直接转人工 |

**5.38.3 与议价的协同**

议价中替代料作为**第 9 维让步货币**（在 v7 的 8 维基础上扩展）：

| 维度 | 等价说明 |
|---|---|
| **substitute**（v11 新） | 不动主价，换更便宜替代料 → 客户得到等价让步；我方可能毛利反增 |

议价场景下：
- 客户砍价 ¥20，Bot 给销售方案：「换永钢同档替代，省 ¥25，无需让单价」
- 销售选择即可

**5.38.4 合同 & 审计**

- 替代方案确认时，订单合同里**单独标注替代关系**
- 客户必须**明确确认**（回复"确认替代"），Bot 不能默认
- 交付时双方签收清单留档
- 事后争议时翻 `substitute_decision_log` 查证据链

**5.38.5 反馈学习**

每次客户接受/拒绝替代 → 更新：
- `customer_profile.substitute_preferences.history`
- `substitute_relation` 表的 `acceptance_rate`
- 影响下次同类推荐的 acceptance_score

**5.38.6 新增存储**

| 表 | 用途 |
|---|---|
| `substitute_relation` | KB 主表 |
| `substitute_relation_version` | KB 版本审计 |
| `substitute_recommendation_log` | 推荐记录（含输入/候选/客户反馈） |
| `substitute_decision_log` | 客户最终决策 |
| `customer_substitute_preference` | 客户偏好画像（扩展 customer_profile） |

### 5.39 Tool Registry（v11 增量）

| 工具 | 入参 | 说明 |
|---|---|---|
| `recommend_substitutes` | inquiry_item, customer_id, use_case? | 调 Match + Strategy，返回 Top-3 候选 |
| `record_substitute_decision` | recommendation_id, action: accept/reject, chosen_substitute? | 落库 + 更新偏好 |

### 5.40 Customer Pattern Miner（客户行为模式挖掘 - v12 新增）

**5.40.1 输入与输出**
```
Input: conversation_event 表的滚动窗口（30/90/180 天）
Output: customer_profile 扩展字段
```

**5.40.2 挖掘的指标**

```yaml
CustomerLearnedProfile:
  default_values_distribution:
    length:   {12m: 7, 9m: 1}  # 30 天 8 次询价的统计
    origin:   {沙钢: 5, 永钢: 3}
    grade:    {HRB400: 8}
    standard: {GB: 8}
  default_modes:               # 众数
    length: {value: "12m", confidence: 0.88, sample_size: 8}
    origin: {value: "沙钢", confidence: 0.625, sample_size: 8}
  typical_order:
    qty_p50: 80
    qty_p90: 150
    monthly_orders: 4.2
  negotiation_pattern:
    avg_rounds: 2.5
    accept_after_concession_pct: {round1: 0.3, round2: 0.5, round3: 0.7}
    cited_competitor_credibility: 0.4  # 历史虚价比例
  decision_time:
    p50_hours: 22
    p90_hours: 60
  substitute_acceptance:
    同档异厂: 0.73
    同档异定尺: 0.40
    向下兼容: 0.0  # 历史从未接受
  time_pattern:
    inquiry_peak_hour: 14    # 多在下午询
    avg_response_minutes: 5
  channel:
    primary: wecom_kf
    secondary: phone
```

**5.40.3 关键计算原则**
- 滚动窗口：默认 30 天，可配
- 样本数 < 10 → 不输出（冷启动）
- 每个指标都标 confidence + sample_size + last_updated
- 计算资源：异步批处理（不阻塞实时报价）

**5.40.4 应用**
- Default Resolver 优先用 customer learned defaults
- Concession Strategy 微调让步幅度（接受率高的客户可少让）
- Lock duration 用客户决策中位数
- Substitute Strategy 用客户接受率

### 5.41 Strategy Effectiveness Tracker（策略效果追踪 - v12 扩展 v6）

**5.41.1 跟踪维度**

```yaml
strategy_metrics:
  per_strategy:
    new_customer_attractive:
      conversion_rate: 0.42
      avg_concession_used: 8       # 让步使用幅度
      avg_decision_time: 18h
      customer_type_breakdown: {new: 0.42, ...}
    volume_driven_lowest:
      conversion_rate: 0.61
      ...
  per_counter_offer:
    "let_price+lock_extend":
      accept_rate: 0.55
      next_round_concession_required: 0.30
    "保价+换替代":
      accept_rate: 0.48
  per_substitute_relation:
    同档异厂:
      accept_rate: 0.68
      profit_margin_change_avg: +5
  per_soft_influence:
    L4_sales_followup:
      回归率: 0.28      # ghost 客户回到正常采购的比例
      avg_recovery_days: 12
```

**5.41.2 输出 → 影响**

- Pricing Engine 策略推荐顺序：转化率高的策略被推荐先
- Counter-offer Generator Top-K：接受率高的组合优先
- Substitute Recommendation：接受率高的关系类型优先
- Soft Influence：回归率低的干预层被人工 review

### 5.42 Self-tuning Parameter Engine（参数自调引擎 - v12 新增）

**5.42.1 调参范围（白名单）**

只允许调以下参数，其他参数不可自动调：

```yaml
self_tunable_params:
  default_values:
    customer.{field}.default   # 各客户的默认值
  lock_minutes:
    by_customer_decision_time   # 按客户决策中位数
  concession_curve:
    fine_tune_ratio: ±0.05     # 让步比例微调
  substitute_threshold:
    customer.acceptance_score   # 客户接受度门槛
  T_routing:
    auto_adjust_per_team        # 销售路由温度参数
forbidden_params:
  - cost_redline
  - margin_floor
  - blacklist_uplift_rate
  - blacklist_classification
  - customer_type_classification
```

**5.42.2 应用条件（5 道门槛逐一通过）**

```
1. 数据质量门槛  ← 样本数 ≥ 10 AND 数据完整性 ≥ 95%
2. 置信度门槛   ← confidence ≥ 0.7
3. Drift 校验   ← 变化幅度 ≤ ±20%
4. A/B 灰度    ← 5% → 20% → 100%
5. 数字溯源    ← 每次应用必落 trace
```

任何一道未通过 → 不应用（保留人工默认）

**5.42.3 应用频率**

- 实时事件：仅更新统计指标，**不立即应用到规则**
- 日批：通过 drift 检查的指标进入 A/B 候选
- 周批：A/B 验证通过的进入下一档灰度
- 月度：完成全量灰度的参数固化到主版本

### 5.43 Drift Monitor（漂移监控 - v12 新增）

**5.43.1 监控指标**

```yaml
drift_alerts:
  customer_pattern:
    threshold_pct: 20            # 变化超 20% 告警
    metrics: [default_modes, negotiation_pattern, decision_time]
  strategy_effectiveness:
    threshold_pct: 15
    metrics: [conversion_rate, accept_rate]
  data_quality:
    sample_size_drop_pct: 50    # 样本量骤降告警
    completeness_drop_pct: 10
```

**5.43.2 告警动作**

| 级别 | 触发 | 动作 |
|---|---|---|
| WARN | 变化 ±10~20% | Slack/邮件 告警 + 继续应用 |
| HIGH | 变化 ±20~40% | **暂停该指标自动应用** + 人工 review |
| CRITICAL | 变化 ±40%+ / 数据流故障 | **全量暂停学习** + 触发应急 + 主管介入 |

**5.43.3 数据健康监控**

- conversion_event 表 lag 检测
- 客户档案同步与 CRM 一致性
- 策略 trace 完整性
- 学习指标的统计分布

### 5.44 Hallucination Guard（幻觉防护层 - v12 新增）

**5.44.1 LLM 输出三道扫描**

```
Step 1 - 占位符强制：
  LLM 必须用 {{PLACEHOLDER}} 表示所有数字
  例：「为您锁价 {{LOCK_HOURS}} 小时」
       不接受：「为您锁价 23 小时」（即使数字正确）

Step 2 - 后置 DLP 数字扫描：
  扫描 LLM 输出文本：
    - 含未占位符的纯数字 → REJECT
    - 含未占位符的钱币符号 → REJECT
    - 含未占位符的吨数/根数 → REJECT
    - 含禁词（成本/老板/亏本/底价/批价等）→ REJECT
  
Step 3 - 占位符渲染：
  把 {{PLACEHOLDER}} 替换为来自结构化方案的实际数字
  方案数据来源 = Pricing Engine / Concession Strategy / Inventory Matcher
  绝不是 LLM 编造
```

**5.44.2 学习层硬白名单防护**

```python
def can_learn(target_param):
    if target_param in FORBIDDEN_LEARNING_TARGETS:
        return False
    if target_param.startswith("cost_") or target_param.startswith("redline_"):
        return False
    return True

FORBIDDEN_LEARNING_TARGETS = [
    "quote_final_price",
    "concession_step_value",
    "cost_redline",
    "margin_floor",
    "customer_blacklist",
    "customer_type_classification",
    "blacklist_uplift_rate",
    "auto_quote_amount_limit"  # Auto 档位上限不允许学习自动调
]
```

**5.44.3 应用前置审计**

每次 Self-tuning Parameter Engine 应用前：
- 检查 target_param 是否在白名单
- 检查 5 道门槛通过
- 落 parameter_application_log 审计

### 5.45 Storage（v12 新增表）

| 表 | 用途 |
|---|---|
| `customer_learned_profile` | 客户行为模式挖掘结果 |
| `strategy_effectiveness_metric` | 策略效果指标 |
| `parameter_tuning_history` | 参数自调历史 |
| `parameter_application_log` | 应用审计 |
| `learning_trace` | 每次报价的学习足迹 |
| `drift_alert_log` | 漂移告警 |
| `ab_experiment` | A/B 灰度实验 |
| `forbidden_learning_targets` | 不学习目标白名单（运营维护） |

### 5.46 Reservation Engine（留货引擎 - v13 新增）

**5.46.1 状态机**

```
                  ┌──────────┐
                  │  DRAFT   │ 客户已表达意图，未确认
                  └────┬─────┘
                       │ confirm
                       ▼
                  ┌──────────┐         deposit.timeout
   submit ───────▶│PENDING_  │──────────────────────▶ CANCELLED_NO_DEPOSIT
                  │DEPOSIT   │
                  └────┬─────┘
                       │ deposit.paid
                       ▼
                  ┌──────────┐         expire
                  │ ACTIVE   │──────────────────────▶ EXPIRED
                  └────┬──┬──┘
                       │  │ amend / extend / negotiate
        convert ───────┘  │
           ▼              ▼
      ┌─────────┐    （各子流程）
      │CONVERTED│
      └─────────┘
```

**5.46.2 关键能力**

- 资格校验（5 项）
- 定金 / 期限计算（按规则表）
- 业务 API 调用（submit_reservation / convert_to_order / extend / cancel）
- 与 Amendment / Negotiation 协同
- Expiry Workflow 调度

**5.46.3 五项资格校验**

```python
def check_eligibility(customer, inquiry, quote):
    if customer.list_status in [blacklist, frozen]:
        return Reject("黑名单/冻结")
    if not has_credit_room(customer, deposit_amount):
        return Reject("信用额度不足")
    if open_reservation_count(customer) >= MAX_OPEN_RES:
        return Reject(f"未结清留货已达 {MAX_OPEN_RES}")
    if available_stock < qty:
        return Reject("库存不足")
    if quote.status not in [ACTIVE, LOCKED]:
        return Reject("报价已过期，请重报")
    return Approve
```

### 5.47 Inventory Allocation Tracker（库存独占追踪 - v13 新增）

**5.47.1 库存状态模型**

```yaml
StockSnapshot:
  total                # 物理库存
  available            # 可报价 / 可留货
  reserved             # 被留货独占
  sold                 # 转正式订单待发货
  在途/调拨/损耗       # 来自 ERP 同步
constraint: available + reserved + sold + 其他 = total
```

**5.47.2 状态迁移规则**

| 事件 | 迁移 |
|---|---|
| 留货创建 | `available -= qty; reserved += qty` |
| 转正式订单 | `reserved -= qty; sold += qty` |
| 留货过期 | `reserved -= qty; available += qty` |
| 留货取消 | `reserved -= qty; available += qty` |
| 留货延期 | 无变化 |
| ERP 库存同步 | 重算 total，按比例调整 available（reserved/sold 不动） |

**5.47.3 并发控制**

- 留货创建走分布式锁（按 sku_id）
- ACL 调业务 API 时携带库存快照版本号
- 业务 API 侧库存检查 + 扣减原子化

**5.47.4 对账**

- 每分钟与业务 ERP 库存同步检查
- 不一致：告警 + 自动重对（按 ERP 为准）
- 严重不一致（差异 > 5%）：暂停留货功能 + 人工介入

### 5.48 Deposit Manager（定金管理 - v13 新增）

**5.48.1 定金生命周期**

```
GENERATED → PAID → REFUNDED / FORFEITED / OFFSET
```

| 状态 | 含义 |
|---|---|
| GENERATED | 已生成支付链接，未付 |
| PAID | 客户已付 |
| REFUNDED | 转单完成自动退还（实际上余款付清，定金作为前款） |
| FORFEITED | 客户违约 → 扣违约金 |
| OFFSET | 违约金扣除后剩余 → 冲下次留货 |

**5.48.2 定金计算函数**

```python
def calculate_deposit(category, customer, total_price):
    base_rate = DEPOSIT_RULES[category]
    
    # 客户类型乘数
    base_rate *= TYPE_MULTIPLIER[customer.type]
    
    # 名单乘数
    if customer.list_status == "whitelist":
        base_rate *= 0.9
    
    # 战略客户特权（可能为 0）
    if customer.type == "strategic":
        if not customer.deposit_waiver_eligible:
            base_rate *= 0.5
        else:
            base_rate = 0  # 免定金
    
    deposit = total_price * base_rate
    
    # 大单地板规则
    if total_price > 1_000_000 and deposit < total_price * 0.1:
        deposit = total_price * 0.1
    
    return deposit, base_rate
```

**5.48.3 与业务系统财务集成**

- 调财务 API 生成支付链接（含订单号 / 客户号 / 金额 / 回调 URL）
- 财务 webhook：deposit.paid / deposit.timeout / deposit.refunded
- 留货过期触发 deposit.forfeit + deposit.offset
- 全程审计 deposit_log

**5.48.4 违约金处理（公司可选模式）**

```yaml
forfeit_policy:
  default: hold_offset                # 扣 50% + 冲下次
  options:
    full_forfeit: 全扣
    hold_offset:  扣 50% + 剩余冲下次
    full_refund:  全退（高端服务/战略客户）
  per_customer_type_override:
    strategic: full_refund
    blacklist: full_forfeit
```

### 5.49 Reservation Expiry Workflow（过期工单 - v13 新增）

**5.49.1 提醒时序**

```yaml
reservation_reminders:
  T-24h:
    to: customer
    template: reservation_remind_24h
  T-12h:
    to: customer
    template: reservation_remind_12h
    buttons: [转单, 延期, 取消]
  T-1h:
    to: customer + bound_sales
    template: reservation_remind_1h_critical
    sales_action: 主动联系客户
  T+0:
    action: auto_expire
    to: customer + sales
    template: reservation_expired
```

**5.49.2 过期自动动作**

```python
def auto_expire(reservation):
    reservation.status = EXPIRED
    InventoryTracker.release(reservation.sku, reservation.qty)
    DepositManager.forfeit(reservation.deposit_id)
    Notifier.send(customer, EXPIRED_TEMPLATE)
    Notifier.send(sales, EXPIRED_SALES_TEMPLATE)
    AuditLog.append(reservation_id, "auto_expired")
```

**5.49.3 延期审批工单**

延期请求 → 创建审批工单 → 销售/主管审批 → 通过则更新 expire_at + 通知客户

**5.49.4 集成 Lead Routing（过期高危）**

过期前 1h 自动触发 Lead Routing 推绑定销售（不是池抽签，直接绑定优先）；如绑定销售不可用，按 v10 池规则升级。

### 5.50 Tool Registry（v13 增量）

| 工具 | 入参 | 说明 |
|---|---|---|
| `propose_reservation` | quote_id, qty | 评估资格 + 计算定金/期限，返回 ReservationOffer |
| `submit_reservation` | offer_id, customer_confirmed | 创建留货 + 调业务 API |
| `convert_reservation_to_order` | reservation_id | 转正式订单 |
| `extend_reservation` | reservation_id, days | 延期申请 |
| `cancel_reservation` | reservation_id, reason | 取消（按合同处理定金） |
| `query_my_reservations` | customer_id, status? | 客户查自己的留货 |

### 5.51 Storage（v13 新增表）

| 表 | 用途 |
|---|---|
| `reservation` | 留货主表 |
| `reservation_event` | 状态迁移流水 |
| `deposit_transaction` | 定金交易 |
| `inventory_allocation_log` | 库存独占变化 |
| `reservation_expiry_workflow` | 过期工单 |
| `reservation_extension_approval` | 延期审批 |
| `deposit_rules_config` | 定金规则（可后台维护） |
| `expiry_rules_config` | 期限规则 |

### 5.52 Quote Lifecycle 扩展（v13 微调 v8）

v8 报价生命周期扩展新状态：

```
ACTIVE / LOCKED  ──── reserve ────▶  RESERVED   ──── convert ────▶  CONVERTED (成订单)
                                       │
                                       └── expire ────▶ EXPIRED + 释放库存
                                       └── cancel ────▶ CANCELLED + 按合同处理定金
                                       └── amend_qty/extend/spec_change → 子流程
```

`Quote.status = RESERVED` 时：
- Pricing Engine 不重新定价
- Negotiation Engine 默认拒绝议价
- Amendment Engine 走 v13 的留货改量子流程（4.15 阶段 9）
- 库存查询时该 sku 已扣除留货量

### 5.53 Configuration Center（企业配置中心 - v14 核心）

**5.53.1 设计目标**

承载本次确认的 25+ 个"可配置项"。多租户 / 多类型 / 版本化 / 审计 / 灰度。

**5.53.2 分类与示例**

```yaml
# === PRICING ===
pricing.auto_quote_amount_limit: 500_0000      # Auto 档金额上限
pricing.cost_basis_default: actual_cost

# === ORDER_PROFIT_FLOOR ===
profit_floor.dealer.min_margin_per_ton: 10
profit_floor.terminal.min_margin_per_ton: 200
profit_floor.category_modifier.不锈钢: 1.5

# === NEGOTIATION ===
negotiation.max_rounds: 5
negotiation.concession_curve.profit_seeker: [0.5, 0.25, 0.1, 0.05]
negotiation.term_extension_requires_credit_approval: true

# === AMENDMENT ===
amendment.locked.single_pct_threshold: 5     # 单次幅度
amendment.locked.cumulative_pct_threshold: 10 # 累计幅度
amendment.per_quote_max: 3
amendment.per_day_per_customer_max: 10
amendment.decrease_pricing_tiers:
  - { max_pct: 5,   policy: keep_locked_price }
  - { max_pct: 15,  policy: standard_tier }
  - { max_pct: 30,  policy: standard_tier_plus_10 }
  - { max_pct: 999, policy: manual }

# === ROUTING ===
routing.binding_source: crm | bot           # 客户绑定销售来源
routing.T: 1.0
routing.ack_deadline_minutes: 5
routing.reply_deadline_minutes: 20          # v14: 30 → 20
routing.decline_rate_threshold: 0.10        # 拒单率 10%
routing.daily_quota:
  junior: 5
  mid:    8
  senior: 12
routing.newcomer_floor:
  onboarding_le_3m: 0.15
  onboarding_le_6m: 0.10
  onboarding_le_12m: 0.05
  else: 0.0
routing.sales_swap_allowed: true             # 销售互调
routing.show_pick_probability_to_sales: true # 中签概率透明

# === SUBSTITUTE ===
substitute.full_stock_proactive_when:
  - same_grade_diff_length_better_price       # 同档异长度更优可推
substitute.acceptance_strategy_by_segment:
  dealer:    aggressive
  terminal:  conservative
substitute.caveat_default_show: true
substitute.savings_share_ratio: 0.5          # 让利分配（让 50% 给客户）

# === LEARNING ===
learning.window_short_days: 7
learning.window_long_days: 90
learning.cold_start_sample: 60               # v14: 10 → 60
learning.confidence_threshold: 0.7
learning.drift_threshold_warn: 0.20
learning.drift_threshold_critical: 0.40
learning.ab_period_days: 30                  # v14: 7 → 30
learning.ab_phase_ratios: [0.05, 0.20, 1.0]
learning.trace_visibility: supervisor_only   # 仅主管看

# === RESERVATION ===
reservation.deposit_rate_by_category:
  螺纹钢: 0.10
  普通板材: 0.10
  卷板: 0.15
  中厚板: 0.15
  不锈钢: 0.30
  管材: 0.15
reservation.duration_days_default:
  strategic: 5
  whitelist: 3
  normal:    3
  new:       1
  warning:   1
reservation.max_open_per_customer: 3
reservation.forfeit_policy: hold_offset      # 默认扣50%冲下次
reservation.strategic_waiver_authorized_roles: [sales_director, ceo]
reservation.large_order_floor:
  amount_threshold: 1_000_000
  min_rate: 0.10
reservation.market_drop_threshold_for_renegotiation: 0.03  # 行情跌 3% 可特批
reservation.extension_approval_levels: [sales_lead]         # 一级
reservation.expiry_1h_sales_kpi: true        # 销售必须接触客户

# === SLA ===
sla.routing_reply_minutes: 20
```

**5.53.3 配置类型**

| 类型 | 含义 | 修改权限 |
|---|---|---|
| **system_param** | 阈值/比例（如 `negotiation.max_rounds`） | 业务管理员 |
| **policy_rule** | 策略规则（如减量退价档位） | 销售总监 + 二人复核 |
| **business_floor** | 业务红线（如毛利底线） | 老板 + 强制审计 |
| **rate_table** | 费率表（如定金率） | 业务管理员 |
| **whitelist_blacklist** | 白名单/黑名单 | 销售主管 |

**5.53.4 灰度发布与回滚**

```
新配置 → 5% 流量 → 7 天观察 → 20% → 30 天观察 → 100%
任何阶段异常 → 一键回滚
business_floor 类型必须双人复核才能上线
```

**5.53.5 配置审计**

```
config_change_log {
  config_key, old_value, new_value,
  changed_by, changed_at, reason,
  rollout_phase, approval_chain[]
}
```

**5.53.6 配置加载**

- 启动加载到 Redis（按 tenant + key 分桶）
- 修改后发布事件，各服务订阅 → 热加载
- 客户端总有 fallback 默认值（防止配置丢失致服务崩溃）

### 5.54 Sales Style Profile（销售话术风格 - v14 新增）

**5.54.1 数据模型**

```yaml
SalesStyleProfile:
  sales_id
  tone: 严谨 | 亲切 | 幽默 | 简洁 | 详细
  intensity: 0~10        # 语气强度（0=平和，10=强势）
  signature_phrases: ["听您的", "帮您争取", "兄弟你给个机会"]
  forbidden_words: [...] # 个人禁用词
  greeting: "张总好～"   # 个人开场白
  closing: "您看下哈"    # 个人结束语
  examples:              # few-shot 实例（议价话术）
    - input: "客户砍 ¥30"
      output: "张总，沙钢这批拿货成本就在那儿，我帮您让 ¥15 到 ¥3,805，锁价再帮您拉到 48h"
  updated_at, reviewed_by
```

**5.54.2 与议价 LLM 集成**

议价 LLM Prompt 注入：
```
System: 你是销售小张的助理 AI。
你的语言风格：亲切（强度 6）
常用语：「听您的」「帮您争取」「咱们这批」
开场：「张总好～」
结束：「您看下哈」
... 标准议价 prompt ...

User: [客户最新消息 + 议价上下文]
```

**5.54.3 风格学习**

学习层（不学决策仅学话术风格）：
- 从销售历史已确认话术中提取 signature_phrases
- 销售可在管理后台编辑
- 每月由销售本人 review

**5.54.4 仍走幻觉防护**

风格定制不绕过数字占位符强制。任何 LLM 输出仍需 DLP 扫描。

### 5.55 Competitor Credibility System（竞品可信度 - v14 新增）

**5.55.1 评分模型**

```yaml
CompetitorCredibility:
  customer_id × competitor_name
  score: 0~100   # 默认 50（中性）
  history:
    - {date, claimed_price, our_price, action_taken, eventual_outcome}
  category: trusted | neutral | suspicious | blacklist
```

**5.55.2 扣分 / 加分规则**

```yaml
score_rules:
  claim_competitor_but_buy_from_us:        # 提了竞品但买了我们
    score_delta: -5
    note: 可能虚价施压
  claim_competitor_and_buy_from_competitor: # 提了竞品且真的去了
    score_delta: +5
    note: 可信
  claim_competitor_no_follow_up:           # 提完没下文
    score_delta: -2
    note: 试探性
  sales_reported_obvious_fake:             # 销售标记明显虚假
    score_delta: -20
    requires: sales_report_with_evidence
```

**5.55.3 应用**

| 分数 | 类别 | 议价场景动作 |
|---|---|---|
| 80+ | trusted | 正常考虑竞品报价 |
| 50~79 | neutral | 系统标注，销售判断 |
| 20~49 | suspicious | **系统警告销售**："该客户此前 N 次提竞品价值得怀疑" |
| < 20 | **专项黑名单** | 销售可提报 → 主管审批 → 进入**专项黑名单**（与普通黑名单分离） |

**5.55.4 专项黑名单**

- 与普通黑名单**分离**（普通黑名单影响留货/账期等；专项黑名单仅影响议价竞品采信）
- 入名单必须由销售提报 + 销售主管审批 + 提供**完整证据日志**
- 强制审计：`competitor_blacklist_log`（含证据链、审批人、有效期）
- 默认 6 个月失效，可续

**5.55.5 与议价 LLM 协同**

议价 prompt 注入：
```
该客户对竞品 X家 的可信度评分：35（suspicious）
历史：4 次提同价位未走 → 销售可适当强硬，但话术仍温和
注意：不要直接说"您虚价"，用市场话术化解
```

### 5.56 Sales Level System（销售级别体系 - v14 新增）

**5.56.1 级别定义**

```yaml
SalesLevel:
  junior:   入职 ≤ 6 月 OR 业绩 P25 以下
  mid:      入职 7~24 月 AND 业绩 P25~P75
  senior:   入职 > 24 月 OR 业绩 P75 以上
```

**5.56.2 配额差异化**

```yaml
daily_new_lead_quota:
  junior: 5
  mid:    8
  senior: 12
```

**5.56.3 入职时间衰减保底**

```yaml
newcomer_floor:
  onboarding ≤ 3m:  0.15  # 必须 ≥ 15% 概率拿到工单
  onboarding ≤ 6m:  0.10
  onboarding ≤ 12m: 0.05
  > 12m:            0.0   # 完全凭实力
```

**5.56.4 客户类型授权**

```yaml
customer_type_authorization:
  junior:  [new, churn]                    # 新人优先练手
  mid:     [new, volume, profit, churn]
  senior:  全部 + strategic + vip
```

**5.56.5 升降级**

- 季度自动评估业绩 + 入职时间 → 推荐升级（销售主管确认）
- 降级慎重：连续 2 季度业绩 P25 以下 → 主管 review
- 升降级落档案审计

### 5.57 Machine-quotable SKU Whitelist（v14 新增）

**5.57.1 数据模型**

```yaml
SKUWhitelistEntry:
  sku_id (category + grade + spec + origin? + length?)
  status: active | review_pending | retired
  added_by, added_at, last_review_at
  notes: "常态品种"
  market_volatility: low | medium | high   # 行情波动度
```

**5.57.2 流程**

```
parse_inquiry 抽取 sku → 查 whitelist:
  hit  → 走标准 Pricing 流程
  miss → 触发 v10 NO_PRICING 转人工
         附建议："该规格未在 SKU 白名单内，建议销售先报价 + 提报添加"
```

**5.57.3 维护**

- 销售 / 业务部可提报"加入白名单"
- 销售主管审批
- 6 个月 review 一次（清理停售品种）
- 行情极不稳定的 SKU 可临时移出 + 自动恢复定时

**5.57.4 与替代料协同**

替代料候选必须在白名单内（否则也无定价）；候选不在白名单的也走人工。

### 5.58 Tool Registry（v14 增量）

| 工具 | 入参 | 说明 |
|---|---|---|
| `get_config` | tenant_id, key | 读配置 |
| `update_config` | tenant_id, key, new_value, reason | 改配置（带审计） |
| `report_fake_competitor` | customer_id, competitor, evidence | 销售提报虚假竞品 |
| `propose_customer_type_change` | customer_id, suggested_type, reason | 系统建议销售考虑改客户类型 |
| `query_sku_whitelist` | sku_id | 查白名单 |
| `add_sku_to_whitelist` | sku_id, justification | 提报加入白名单 |

### 5.59 Storage（v14 新增表）

| 表 | 用途 |
|---|---|
| `config_entry` | 配置主表（按 tenant + key） |
| `config_change_log` | 配置变更审计 |
| `customer_segment_history` | 客户细分变更 |
| `sales_style_profile` | 销售话术风格 |
| `competitor_credibility` | 竞品可信度（customer × competitor） |
| `competitor_credibility_event` | 评分事件流 |
| `competitor_blacklist_special` | 专项黑名单（议价层用） |
| `sales_level_history` | 销售级别变更 |
| `sku_whitelist` | 可机器报价 SKU 白名单 |
| `customer_type_suggestion` | 系统给销售的客户类型建议 |
| `ghost_score_amend_to_zero` | 改到 0 的客户行为记录（影响下次询价待遇） |

### 5.60 Self-built Capability Layer（自建能力层 - v15 核心）

> 4 个能力 ERP 不支持，由本系统自建。这部分不是"调 ERP"，而是 Bot 系统的**自有业务子系统**。

**5.60.1 创建询价单（Inquiry Order）**

ERP 无询价单概念，Bot 自建：

```yaml
InquiryOrder:
  inquiry_id, customer_id, sales_id, created_at, channel
  items: [InquiryItem × N]    # v5 的 9 要素结构
  status: DRAFT | SUBMITTED | QUOTING | QUOTED | CLOSED
  source: text | image | excel | pdf
  attachments: [原始文件 OSS 引用]
  quote_refs: [Quote × N]
```

- 支持批量（一份 Excel N 行 → N 个 item）
- 与 ERP 关系：询价单是 Bot 侧的"前置单据"；客户下单转单时才在 ERP 创建正式订单
- 报价完成（销售在 Bot 报价）后 → quote.ready 不需要 ERP webhook（因为询价在 Bot 侧）
- 但若 ERP 也要看询价数据 → Bot 反向推送 ERP（可选）

**5.60.2 材质书 PDF 生成与管理（Material Cert）**

ERP 无材质书电子化，Bot 自建：

```yaml
MaterialCert:
  cert_id, batch_no（炉批号）, order_id?, customer_id
  source: generated（Bot 按模板生成）| uploaded（上传扫描件）
  pdf_oss_url, generated_at
  fields: { 牌号, 规格, 炉号, 化学成分, 力学性能, 标准号, ... }
```

- 两种来源：① 按模板 + ERP/质检数据生成 PDF；② 人工上传扫描件
- 客户索要 → 查 cert → Media Pipeline 发送
- 与 ERP 关系：化学成分/力学性能数据从 ERP 或质检系统拉（若有）；否则人工录入

**5.60.3 付款凭证管理（Payment Voucher）**

ERP 无凭证接收，Bot 自建（v3 已设计 Media Pipeline 入站，v15 明确为自建）：

```yaml
PaymentVoucher:
  voucher_id, customer_id, order_ids[], amount, paid_at
  oss_url, ocr_result（Qwen-VL）
  status: PENDING_VERIFY | VERIFIED | REJECTED
  verified_by（财务）, verified_at
```

- 客户上传 → OSS + Qwen-VL OCR → 关联订单 → 财务审核工单
- 与 ERP 关系：财务确认后，Bot 把核销信息**写回 ERP**（若 ERP 有应收核销 API）或人工录入

**5.60.4 发货催办（Shipment Urge）**

ERP 无催办单，Bot 自建（v3 工单基础上明确）：

```yaml
ShipmentUrge:
  urge_id, order_id, customer_id, sales_id
  message, status: OPEN | ACKED | RESOLVED
  sla, created_at, resolved_at
```

- 客户催 → 创建催办工单 → 通知销售 → 销售 Bot 内 /reply
- 与 ERP 关系：查订单发货状态从 ERP 读；催办本身是 Bot 侧工单

**5.60.5 自建能力与 ERP 的数据同步原则**

| 数据流向 | 机制 |
|---|---|
| ERP → Bot（读） | ACL 调 ERP API（订单/库存/欠款/装车重量/结算单） |
| Bot → ERP（写回） | 转正式订单 / 付款核销 → 调 ERP 写 API 或人工 |
| Bot 自有数据 | 询价单 / 材质书 / 付款凭证 / 催办工单（Bot 数据库） |
| 对账 | 每日 Bot 自有单据与 ERP 订单对账，发现遗漏告警 |

### 5.61 Interactive Quote Editor（交互式报价编辑 - v15）

**5.61.1 报价回推（含 PDF）**

```
报价生成 → 三种形式同时回推：
  1. 微信消息内 Markdown 表格（逐行展示，直观）
  2. PDF 报价单（正式，可转发/存档）
  3. 交互式编辑入口（卡片按钮 或 H5 链接）
```

**5.61.2 整单 + 逐行展示双模式**

```
整单报价（销售一次报完）但展示逐行：
┌─────────────────────────────────────
│ 报价单 Q-001（共 5 行，总价 ¥958,000）
├─ 1. 螺纹 HRB400 Φ25 沙钢 50t  ¥3,820  [改]
├─ 2. 螺纹 HRB400 Φ22 沙钢 30t  ¥3,810  [改]
├─ 3. 中板 Q235B 12mm 100t     ¥4,050  [改]
├─ 4. 工字钢 14# 20t           ¥4,200  [改]
├─ 5. 角钢 ∠50×5 10t          ¥4,100  [改]
├─────────────────────────────────────
│ [全部接受] [下载 PDF] [整单议价] [逐行改]
└─────────────────────────────────────
```

**5.61.3 客户快速修改（追求方便快捷）**

| 修改方式 | 实现 |
|---|---|
| 点某行 [改] | 弹卡片：改数量 / 删除该行 / 单独议价 |
| 文字快捷 | "第 3 行改 80 吨" / "删除第 5 行" / "1 和 2 行各加 20t" |
| H5 编辑页 | 复杂多行修改 → H5 表格批量编辑 → 提交即重算 |
| 语音 | 客户发语音 → Qwen 转写 → 解析修改意图 |

**5.61.4 修改即时重算联动**

```
客户改第 3 行数量 100t → 80t
   ↓
Interactive Quote Editor:
  - 调 Pricing Engine 重算第 3 行（量阶梯可能变）
  - 调 Inventory Matcher 重新匹配库存
  - 调 Order Optimizer 重算整单毛利（校验 segment 底线）
  - 走 Amendment Engine 决策（v8/v14：AUTO/ASSISTED/...）
   ↓
生成 Quote 新版本（v8 版本化）
   ↓
回推更新后的逐行展示 + 新 PDF
```

**5.61.5 PDF 报价单**

- 模板化（含公司抬头 / 客户信息 / 逐行明细 / 总价 / 有效期 / 条款）
- 出站 Media Pipeline 发送
- 每个 Quote 版本一份 PDF（版本号水印）
- 客户改完重新生成

### 5.62 品类要素必填配置（v15）

**5.62.1 按品类配置必填/可空**

```yaml
category_field_requirements:
  螺纹钢:
    required: [category, grade, spec, qty, dest_city]
    optional: [origin, length, standard, delivery_date]
    # length/standard 有默认（12m / GB），可空走默认
  中厚板:
    required: [category, grade, spec(厚度), qty, dest_city]
    optional: [origin, length(定尺), standard, surface]
  无缝管:
    required: [category, grade, spec(外径×壁厚), qty, dest_city]
    optional: [origin, length, standard]
  不锈钢:
    required: [category, grade(必须明确如304/316L), spec, qty, dest_city]
    optional: [origin, surface, edge]
```

**5.62.2 与 Inquiry Parser / Default Resolver 协同**

- 解析后按品类查必填项
- 必填项缺失 → critical_missing → 反问（v5）
- 可空项缺失 → 走 Default Resolver 默认推断（v5）
- 配置在 Configuration Center（企业可调）

**5.62.3 与转人工协同**

- required 中 `category` + `spec` 同时缺失 → v12 品类规格全空转人工
- 其余 required 缺失 → 反问 2 轮 → 补不齐转人工

### 5.63 Billing & Quota（计费与配额 - v15）

**5.63.1 SaaS 套餐体系**

```yaml
plans:
  basic:
    monthly_fee: ...
    vl_calls_per_month: 1000
    file_retention_months: 12
    seats: 5
  standard:
    vl_calls_per_month: 5000
    file_retention_months: 12
    seats: 20
  premium:
    vl_calls_per_month: 20000
    file_retention_months: 24
    seats: 50
  enterprise:
    vl_calls_per_month: custom
    file_retention_months: custom
    seats: custom
```

**5.63.2 Qwen-VL 配额**

- 按套餐月度配额
- 用量计量：每次 VL 调用计 token + 次数
- 超额：① 限流（拒绝并提示升级）② 或按量计费（套餐外单价）
- 配额告警：80% / 95% / 100%
- file hash 缓存命中不计费（v9 优化）

**5.63.3 文件保留计费**

- 默认 1 年免费
- 超期保留：按存储量/月计费
- 财务相关文件（付款凭证）强制保留 5 年（合规，单独计费或包含）
- 客户可选"延长保留"增值服务

**5.63.4 用量计量与账单**

```yaml
usage_metering:
  - vl_calls
  - storage_gb_months
  - active_seats
  - message_volume（可选）
billing_cycle: monthly
quota_enforcement: soft（提示）| hard（限流）  # 可配
```

**5.63.5 配额与功能降级**

- VL 配额耗尽 → 询价图片/PDF 解析降级为"请用文字描述或联系销售"
- 不影响核心文字询价 / 报价 / 议价（这些不耗 VL）
- 提示企业升级套餐

### 5.64 Tool Registry（v15 增量）

| 工具 | 入参 | 说明 |
|---|---|---|
| `create_inquiry_order` | items[], customer_id | 自建询价单 |
| `generate_material_cert` | batch_no/order_id | 生成材质书 PDF |
| `submit_payment_voucher_selfbuilt` | image, order_ids | 自建付款凭证（替代原 ERP 调用） |
| `create_shipment_urge` | order_id, message | 自建发货催办 |
| `edit_quote_line` | quote_id, line_no, changes | 交互式逐行改 |
| `generate_quote_pdf` | quote_id | 生成报价单 PDF |
| `check_quota` | tenant_id, resource | 查配额 |

### 5.65 Storage（v15 新增表）

| 表 | 用途 |
|---|---|
| `inquiry_order` | 自建询价单 |
| `material_cert` | 材质书 |
| `payment_voucher` | 付款凭证（自建） |
| `shipment_urge` | 发货催办工单 |
| `quote_pdf` | 报价单 PDF 版本 |
| `category_field_config` | 品类要素必填配置 |
| `saas_plan` | 套餐定义 |
| `tenant_subscription` | 企业订阅 |
| `usage_metering` | 用量计量 |
| `quota_alert_log` | 配额告警 |
| `erp_sync_log` | 自建能力与 ERP 同步/对账 |

### 5.66 Price Composition Engine（报价构成引擎 - v16 核心）

**5.66.1 报价构成数据模型**

```yaml
PriceComposition:
  quote_line_id, sku
  qty_ton

  # 1) 基准货款
  base:
    unit_price: 3650            # 基准货款单价
    cost_basis: actual_cost | list_price   # 按 segment（v14）
    tax_rate: 0.13

  # 2) 加工费
  processing:
    items:
      - { type: 剪切,  rate_per_ton: 30, cost_per_ton: 18 }
      - { type: 开平,  rate_per_ton: 50, cost_per_ton: 32 }
    tax_rate_one_invoice: 0.13
    tax_rate_two_invoice: 0.13   # 或 0.06（看加工服务定性）

  # 3) 运输
  delivery:
    mode: 自提 | 送货上门 | 代送
    freight_per_ton: 30          # 自提=0
    freight_cost_per_ton: 22     # 我方实际成本
    agent_service_fee: 0         # 代送服务费
    tax_rate: 0.09

  # 4) 结算/票据
  settlement:
    method: 现金 | 电汇 | 银行承兑 | 商业承兑
    bill_term_months: 6
    discount_rate_annual: 0.03   # 贴现率
    discount_cost: ...           # = 票面 × rate × term

  # 5) 账期资金成本
  credit:
    days: 30
    finance_rate_annual: 0.08    # 资金成本率
    finance_cost: ...            # = 货值 × rate × days/365

  # 6) 开票方式
  invoice:
    mode: one_invoice | two_invoice
    tax_diff_adjustment: ...     # 一票制税差补偿

  # 输出
  result:
    pretax_subtotal: ...
    tax_total: ...
    total_with_tax: ...
    unit_price_final: ...
    full_cost: ...               # 全成本（成本红线用）
    breakdown: [...]             # 逐项明细
```

**5.66.2 计算流程**

```
1. 基准货款 = base.unit_price × qty
2. 加工费   = Σ processing.items.rate × qty
3. 运费     = delivery.freight_per_ton × qty（自提=0）+ 代送服务费
4. 贴现成本 = 票面 × discount_rate × (term_months/12)  （现金=0）
5. 资金成本 = 货值 × finance_rate × (credit_days/365)   （现款=0）
6. 税务计算：
   - 一票制：全部并入 13% → 算税差补偿
   - 两票制：货款 13% + 运费 9% + 加工 13%/6% 分别计税
7. 全成本 = 货款成本 + 加工成本 + 运输成本 + 贴现成本 + 资金成本
8. 报价单价 = (各项加总 + 税) / qty
9. 校验：报价单价 ≥ 全成本 × (1 + min_margin_by_segment)
```

**5.66.3 一票/两票制税务计算**

```python
def calc_tax(comp):
    if comp.invoice.mode == "one_invoice":
        # 全部按 13% 开票
        taxable = base + processing + freight
        tax = taxable * 0.13
        # 我方运费进项仅 9%，存在税差损失 → 补偿
        freight_tax_loss = freight * (0.13 - 0.09)
        tax_diff_adjustment = freight_tax_loss
    else:  # two_invoice
        tax = base*0.13 + freight*0.09 + processing*proc_rate
        tax_diff_adjustment = 0
    return tax, tax_diff_adjustment
```

**5.66.4 构成项作为让步货币（13 维）**

| 让步维度 | 实现 |
|---|---|
| 加工费减免 | processing.rate 调减（不破加工成本红线） |
| 现金价 | settlement 切现金 → discount_cost=0，等价让利 |
| 账期优惠 | credit.days 调减 → finance_cost 减（注意与"给账期"反向） |
| 一票转两票 | invoice.mode 切换 → 帮客户优化税务 |

Counter-offer Generator（5.27）扩展这 4 维，按客户类型推荐。

**5.66.5 配置项（Configuration Center）**

```yaml
price_composition:
  processing_rates:        # 加工费率（按工序）
    剪切: { rate_per_ton: 30, cost_per_ton: 18 }
    开平: { rate_per_ton: 50, cost_per_ton: 32 }
    纵剪: { rate_per_ton: 40, cost_per_ton: 25 }
    折弯: { rate_per_process: 5 }
  freight_table:           # 运费表（按距离/目的地）
    武汉江夏: { per_ton: 30, cost: 22 }
  discount_rates:          # 贴现率
    银行承兑: 0.03
    商业承兑: 0.05
  finance_rate_annual: 0.08   # 资金成本率
  tax_rates:
    货物: 0.13
    运输: 0.09
    加工服务: 0.13
  one_invoice_tax_compensation: true   # 一票制是否补偿税差
```

**5.66.6 与其他模块联动**

| 模块 | 联动 |
|---|---|
| Pricing Strategy（5.18） | 策略作用于"基准货款"，构成引擎叠加其他项 |
| Order Profit Optimizer（5.26） | 整单毛利按全成本核算（含加工/运输/贴现/资金） |
| Counter-offer Generator（5.27） | 让步货币扩展 4 维 |
| Interactive Quote Editor（5.61） | 报价单逐项展示构成 + 改构成即时重算 |
| Customer Profile（5.17） | 学习客户常用：开票方式/结算方式/交付方式/常加工工序 |
| Configuration Center（5.53） | 全部费率/税率/贴现率可配 |

**5.66.7 客户偏好学习（与 v12 自我学习联动）**

学（事实）：客户常用开票方式（一票/两票）、常用结算（现金/承兑）、常用交付（自提/送货）、常需加工工序
不学：费率/税率/贴现率/资金成本率（这些是配置项，列入不学习白名单）

**5.66.8 新增存储**

| 表 | 用途 |
|---|---|
| `price_composition` | 每个报价行的构成明细 |
| `processing_rate_config` | 加工费率配置 |
| `freight_table_config` | 运费表 |
| `discount_rate_config` | 贴现率配置 |
| `customer_settlement_preference` | 客户结算/开票/交付偏好（学习） |

### 5.67 Quote Delivery & Interaction（报价投递与交互 - v17 核心）

**5.67.1 投递编排**

```
报价生成（Pricing + Composition）
   ↓
Quote Delivery 编排，按渠道能力选择投递组合：

微信客服（外部客户）:
  1. 发文字摘要（纯文本，空格对齐 + emoji）
  2. 发 PDF 报价单（file 消息）
  3. 发菜单消息 msgmenu（轻交互选项）
  4. 发 H5/小程序链接（重交互，可选）

企业微信（内部销售）:
  1. 发模板卡片 template_card（带按钮）
  2. 附 PDF
```

**5.67.2 文字摘要渲染（微信不支持 Markdown）**

```python
def render_text_summary(quote):
    # 微信纯文本，用空格对齐 + emoji，不用 Markdown 表格
    lines = ["📋 您的报价已出（{} 行，含税合计 ¥{}）".format(n, total)]
    for i, line in enumerate(quote.lines, 1):
        lines.append(f"{i}. {line.sku} {line.qty}吨  ¥{line.unit_price}/吨")
    lines.append(f"结算：{settlement} | 交付：{delivery} | 有效期 {valid}h")
    return "\n".join(lines)
```

**5.67.3 菜单消息（msgmenu）**

```json
{
  "msgtype": "msgmenu",
  "msgmenu": {
    "head_content": "请选择您的操作：",
    "list": [
      {"type":"click","click":{"id":"accept","content":"① 接受下单"}},
      {"type":"click","click":{"id":"reserve","content":"② 申请留货"}},
      {"type":"click","click":{"id":"negotiate","content":"③ 我要议价"}},
      {"type":"click","click":{"id":"amend","content":"④ 修改数量"}},
      {"type":"click","click":{"id":"human","content":"⑤ 转人工"}}
    ],
    "tail_content": "或直接回复文字，如『便宜点』『第3行改80吨』"
  }
}
```

客户点击 → 回传 click.id → 路由对应流程。

**5.67.4 对话式意图路由**

```python
INTENT_ROUTES = {
    "accept":     to_order_flow,         # 4.x 下单
    "reserve":    to_reservation_flow,   # 4.15 留货
    "negotiate":  to_negotiation_flow,   # 4.8 议价
    "amend":      to_amendment_flow,     # 4.10 改量
    "human":      to_lead_routing,       # 4.11 转人工
    "composition_change": to_composition_recalc,  # v16 构成调整
}

def handle_customer_reply(msg):
    if msg.is_menu_click:
        return INTENT_ROUTES[msg.click_id](...)
    intent = llm_classify(msg.text)   # "便宜点"→negotiate, "第3行改80吨"→amend
    return INTENT_ROUTES[intent](...)
```

**5.67.5 页面式（H5/小程序）**

```
报价详情页（H5 或小程序）：
  - 顶部全局选项：开票(一票/两票) / 结算(现金/承兑/账期) / 交付(自提/送货/代送)
  - 逐行明细：数量可编辑、删除、单独议价按钮
  - 切换任意选项 → 调后端重算 API → 实时刷新总价
  - 底部：[全部确认下单][申请留货][提交议价][转销售]
  - 提交 → 回调 Bot → 微信对话流确认

技术：
  - H5：企业微信 OAuth 获取身份 + 后端 API
  - 小程序：需开发钢贸小程序（增强版，长期可选）
  - 后端共用 Interactive Quote Editor（5.61）逻辑
```

**5.67.6 降级链**

```
首选：文字摘要 + PDF + H5/小程序链接
  ↓ H5/小程序不可用 / 客户不点
文字摘要 + PDF + 菜单消息
  ↓ 菜单不支持
纯文字摘要 + 文字快捷指令（"回复 1 接受 / 2 留货 / 3 议价"）
  ↓ 文件发送失败
纯文字摘要兜底
```

**5.67.7 与模块联动**

| 模块 | 联动 |
|---|---|
| Interactive Quote Editor（5.61） | H5/小程序前端 + 逐行重算后端 |
| Price Composition（5.66） | 页面全局选项切换 → 重算 |
| 议价/改量/留货 | 双路径均触发同一后端流程 |
| Quote Lifecycle（5.31） | 每次改生成新版本 + 新 PDF |
| Billing & Quota（5.63） | H5/小程序/PDF 生成计入用量（如适用） |

**5.67.8 新增存储**

| 表 | 用途 |
|---|---|
| `quote_delivery_log` | 投递记录（渠道/形式/送达状态） |
| `quote_interaction_event` | 客户交互事件（菜单点击/页面操作/文字指令） |
| `h5_quote_session` | H5/小程序报价会话态 |

### 5.68 Mini Program（微信小程序 - v18 核心）

**5.68.1 形态：平台统一小程序 + 企业隔离**

```
一个平台小程序（SaaS）
  ├ 客户进入 → 绑定所属钢贸企业（tenant_id）
  ├ 数据按 tenant 隔离（复用 v14 多租户）
  ├ 微信支付：平台商户号 + 分账 / 各企业子商户号
  └ 大客户可升级白标小程序（增值）
```

**5.68.2 BFF（Backend for Frontend）**

```
小程序前端
   ↓ HTTPS API
BFF 层（小程序专用聚合接口）
   ↓ 复用
Bot 后端引擎（Pricing / Composition / 议价 / 留货 / 自建能力层 / 配置中心）
```

- BFF 做接口聚合 + 小程序态裁剪，不重复业务逻辑
- 后端引擎与微信对话侧完全共享

**5.68.3 身份打通**

```python
# 小程序登录 → 关联 Bot 客户身份
def mp_login(code):
    session = wx_code2session(code)      # openid + unionid
    binding = find_by_unionid(session.unionid)
    if not binding:
        # 引导绑定（与微信客服 external_userid 关联）
        return need_binding(session)
    return inject_biz_identity(binding.biz_user_id)  # 复用 v5 绑定
```

- 平台小程序 + 微信客服挂同一微信开放平台账号 → unionid 唯一
- external_userid ←→ unionid ←→ biz_user_id 三者打通

**5.68.4 页面与后端映射**

| 页面 | 复用后端 |
|---|---|
| 询价 | Inquiry Parser（5.12）+ 自建询价单（5.60） |
| 报价详情 | Interactive Quote Editor（5.61）+ Price Composition（5.66） |
| 议价 | 议价引擎（4.8 / 5.24） |
| 留货 | Reservation（4.15 / 5.46）+ 微信支付 |
| 订单/磅单/材质书 | ACL 调 ERP + 自建材质书（5.60） |
| 对账/付款凭证 | 结算单 + 付款凭证（5.60）+ 微信支付 |

**5.68.5 微信支付**

| 场景 | 实现 |
|---|---|
| 留货定金 | wx.requestPayment → Deposit Manager（5.48）回调 |
| 余款/付款 | 转单付清 / 对账后付款 |
| 分账 | 平台商户号收 → 分账各企业；或各企业子商户号直收 |

**5.68.6 订阅消息**

| 模板 | 触发 |
|---|---|
| 报价更新 | 议价/改量后新报价 |
| 留货到期 | T-24h/12h/1h（5.49） |
| 订单状态 | shipment.*（Inbound Webhook 5.10） |
| 对账单生成 | settlement.created |

- 一次性订阅，客户每次授权
- 与微信客服 48h 窗口互补（订阅消息可突破窗口）

**5.68.7 降级与三层关系**

```
对话式（5.67 微信客服文字/菜单）= 主入口 + 兜底
  ↑增强
小程序（5.68）= 重交互核心
  ↑降级
H5（5.67）= 小程序不可用/审核期备选
```

**5.68.8 新增存储**

| 表 | 用途 |
|---|---|
| `mp_user_binding` | 小程序 openid/unionid ←→ biz_user_id |
| `mp_subscribe_authorization` | 订阅消息授权记录 |
| `wx_payment_transaction` | 微信支付交易 |
| `wx_payment_settlement` | 分账记录 |
| `mp_tenant_binding` | 客户 ←→ 所属钢贸企业 tenant |

**5.68.9 待决策（见 DESIGN.md 1.17.11）**

形态（平台统一 vs 白标）/ 支付资质 / 开放平台账号 / 销售端 / 上线时机 / 白标增值。

### 5.69 Sales Measurement Method Engine（销售计量方式引擎 - v19 核心）

**5.69.1 计量方式数据模型**

```yaml
SalesMeasurementMethod:
  method: 过磅 | 点支 | 理计 | 平方 | 按米 | 抄牌
  price_unit: 元/吨 | 元/支 | 元/根 | 元/㎡ | 元/米
  quantity_unit: 吨 | 支 | 根 | ㎡ | 米 | 件 | 捆
  weight_basis: actual | theoretical | nameplate | none
  show_ton_equivalent: true | false   # 是否展示折合吨
```

**5.69.2 企业配置矩阵**

```yaml
tenant_category_measurement:
  <tenant_id>:
    <category>:
      allowed: [过磅, 抄牌, ...]
      default: 过磅
      forbidden: [点支, ...]
```

- 存 Configuration Center（5.53），类型 = policy_rule
- 业务部维护；不在 allowed 内的方式 → 系统禁用并提示

**5.69.3 计量 → 价格/数量计算**

```python
def calc_amount(method, qty, unit_price, item):
    if method == "过磅":
        # 下单预估吨位，磅单为准
        return qty_ton_estimate * unit_price, "以实际过磅为准"
    if method == "点支":
        return qty_pieces * unit_price_per_piece, None
    if method == "理计":
        theo_weight = qty * lookup_theoretical_weight(item)
        return theo_weight * unit_price_per_ton, None
    if method == "平方":
        return qty_sqm * unit_price_per_sqm, None
    if method == "按米":
        return qty_meter * unit_price_per_meter, None
    if method == "抄牌":
        nameplate_weight = lookup_nameplate_weight(item, qty_pieces)
        return nameplate_weight * unit_price_per_ton, None
```

**5.69.4 重量表（复用 + 扩展 Steel KB 5.36）**

| 表 | 用途 |
|---|---|
| `theoretical_weight_table` | 理论单重（理计用，5.36 已有） |
| `nameplate_weight_table` | 抄牌标称重量（抄牌用，v19 新增）|

**5.69.5 过磅预估 vs 磅单结算**

```
下单（过磅方式）：按理论/历史估吨位 → 预估金额
发货过磅 → 实际吨位（v3 装车重量 / 磅单）
结算：以磅单为准
磅差处理：企业可配（磅差范围、超差是否复磅、超差责任）
```

**5.69.6 与各模块联动**

| 模块 | 联动 |
|---|---|
| Inquiry Parser（5.12） | 识别 qty 单位 + 按品类默认推断计量方式 |
| Default Resolver（5.14） | 计量方式缺失 → 用企业品类默认 |
| Price Composition（5.66） | 各构成项按计量单位计价 |
| MOQ（5.18/v8） | MOQ 按计量方式（吨/支/㎡…） |
| Inventory（5.47） | 库存计量与销售计量换算 |
| Reservation（5.46） | 留货量按计量方式 |
| Interactive Quote Editor（5.61） | 报价单价格单位 + 改量单位 |
| 自我学习（5.40） | 学客户/品类常用计量方式（事实，可学）|

**5.69.7 校验规则**

- 客户/销售选的计量方式必须在企业该品类 allowed 内
- 不允许的方式 → 提示"本企业 H型钢仅支持抄牌/过磅"
- 计量方式与品类不匹配（如螺纹按平方）→ 拦截 + 反问

**5.69.8 新增存储**

| 表 | 用途 |
|---|---|
| `measurement_method_config` | 企业×品类×计量方式配置 |
| `nameplate_weight_table` | 抄牌标称重量表 |
| `weighbridge_diff_rule` | 磅差处理规则 |
| `customer_measurement_preference` | 客户常用计量方式（学习） |

---

