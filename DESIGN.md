# 企业微信机器人 — 设计文档（v11）

> 状态：设计阶段（尚未开发）
> 行业：**钢铁贸易**
> 目标：搭建一个企业微信智能机器人，对接 8 项后端能力，覆盖内部员工和外部微信客户，接入 LLM（DeepSeek / 通义千问，含 VL 视觉模型）。
> **v11 核心**：新增**替代料推荐能力**——客户询的规格无货/不足/即使有货但有更优替代时，主动推荐 7 类替代关系（同档异厂/异定尺/品质/向上向下/规格相近/国标外标等价）；客户知情决策；推荐时机三场景控制（完全有货默认不推、部分有货推拼方案、完全无货替代主推）；客户接受度三维度过滤（类型/历史/用途）。
> **v10 核心**：统一转人工路由 + 销售竞争评分 + SLA-ACK 转派。
> **v9 微调**：OCR/视觉统一通义千问 Qwen-VL。
> **v8 核心**：报价生命周期状态机 + 改量五档决策。

---

## 0. 需求方已确认的关键约束

| # | 答复 | 影响 |
|---|---|---|
| 1 | 内 + 外双通道 | 自建应用 + 微信客服 |
| 2 | 业务 API 不可改 | ACL |
| 3 | 必须绑定 | 数据级权限 |
| 4 | 多轮、跨天、多意图 | Topic + Task |
| 5 | 公网云 | 云原生 |
| 6 | 钢铁贸易 | 强权限 + DLP + 审计 |
| 7 | DeepSeek + 通义千问 | LLM 双路由（含 VL） |
| 8 | 8 项能力 | 工单 + 反向回调 + 文件管道 |
| 9 | 询价 4 形态 | 多模态解析 |
| 10 | 询价 9 要素口语化 | KB + 默认推断 + 字段级溯源 |
| 11 | 多库存 + 客户特性 + MOQ + 黑白名单 + 白嫖客户 | 三档报价 + 画像引擎 + 策略引擎 + 5 层干预 |
| **12（v7 新）** | **多轮讨价还价 + 整单利润 + 求利/求量差异化议价** | **议价状态机 + 让步策略 + 整单优化 + 多维让步 + 议价 LLM** |
| **13（v8 新）** | **报价后改量场景（含跌穿 MOQ / 超库存 / 改量频繁 / 锁后改量等）** | **Quote Amendment Engine + 报价生命周期状态机（版本化）+ 四档决策（自动/销售/议价/人工）+ 改量频次防护** |
| **14（v9 新）** | **OCR / 视觉理解统一到通义千问** | **Qwen-VL 系列承接所有视觉任务**；移除 PaddleOCR / 阿里云 OCR / 腾讯云 OCR 等其他视觉服务；架构简化但需明确单供应商故障降级链 |
| **15（v10 新）** | **无库存 / 未定价 → 自动转人工 + 销售选派规则**（绑定优先 / 团队池竞争抽签 / 优秀销售优先 / 不让马太效应） | **Lead Routing Engine + Sales Performance Score + Sales State Manager + SLA-ACK 超时转派 + 配额限制** |
| **16（v11 新）** | **替代料推荐（提升竞争力）**：无货/不足/即使有货时主动推同档/异厂/异定尺/品质/向上向下/规格相近/国标外标等价的替代方案 | **Substitute Knowledge Base + Substitute Match Engine + Recommendation Strategy + 客户知情决策 + 用途场景识别 + 客户接受度过滤** |

---

## 1. 需求与目标

### 1.1~1.5 同 v6
略。

### 1.6 议价业务本质（v7 核心认知）

#### 1.6.1 客户砍价的 6 种姿势

| 姿势 | 客户话术 | 应对核心 |
|---|---|---|
| 直接砍价 | "便宜点 3750 吧" | 让步曲线 + 价值替代 |
| 竞品施压 | "X 家给我报 3780" | 真假验证 + 差异化论证 |
| 条件交换 | "再降 10 我就下" | 抓成交意图 + 锁条件 |
| 量价互换 | "加 20t 再降 5" | 量阶梯激励 |
| 多维要求 | "送货 + 账期 + 降价" | 拆分让步、分维度处理 |
| 整单打包 | "三品种一起来个底价" | 整单利润优化 |

#### 1.6.2 让步的 8 种货币（不只价格）

| 维度 | 成本 | 客户敏感度 | 备注 |
|---|---|---|---|
| 价 | 直接侵蚀毛利 | ★★★★★ | 最敏感，谨慎用 |
| 量 | 用量补毛利 | ★★★★ | 量阶梯绑定 |
| 时 | 时间/行情风险 | ★★★ | 锁价延长 |
| 运 | 运费成本 | ★★★ | 免/补运费 |
| 期 | 资金占用 | ★★★ | 账期延长（须信用支持） |
| 质 | 库存损耗 | ★★ | 升优级品同价 |
| 赠 | 接近零成本 | ★ | 材质书、优先发车、优先供应 |
| 组 | 整体优化 | ★★★ | 整单打包 |

**老销售经验**：保毛利的关键是把"降价"换成"价值替代"。Bot 必须把这 8 种让步当工具箱，不只盯着第一格。

#### 1.6.3 客户类型 × 议价节奏

| 客户类型 | 议价偏好 | 推荐让步货币 | 让步节奏 |
|---|---|---|---|
| 求利型（旧称"利型"在 v6） | 每分钱都要 | 价 + 赠 + 组 | 慢让、递减、末端价值替代 |
| 求量型（旧称"量型"在 v6） | 愿意加量换价 | 量 + 价绑定 + 阶梯 | 用量解锁价格 |
| 战略型 | 要稳定供应 | 时 + 期 + 组 | 一次给痛快 + 框架协议 |
| 新客 | 看诚意又怕被宰 | 价（首单明牌）+ 赠 | 一次报到位，不缠斗 |
| 流失型 | 关心是否被重视 | 价 + 赠 + 销售关怀 | 唤回价 + 限时 |

> v7 把 v6 的 "profit/volume" 客户类型重命名为更直观的 **求利型 / 求量型**，与中文业务语境一致。

#### 1.6.4 整单利润视角（v7 重点）

客户询单 = N 个 item，每个 item 有自己的毛利空间。议价时不能只看当前 item，要看**整单毛利**：

```
示例：客户要 3 个品种
  螺纹  50t  毛利 ¥150/t（薄）
  工字钢 30t  毛利 ¥320/t（厚）
  中板  20t  毛利 ¥220/t（中）
整单毛利总额 ¥21,500，毛利率 5.6%

客户砍："螺纹再降 30"

5 种应对（让步分配方案）：
  A. 螺纹直接降 30 → 螺纹毛利薄到 ¥120/t（求利型可接受）
  B. 螺纹保价，中板让 50 → 整单让 ¥1,000，客户视觉上让得更多
  C. 螺纹 10 + 工字 20 + 中板 10 → 整单让 ¥1,400，各项都让
  D. 整单一口价 ¥382,500 → 让 ¥1,500（求战略客户喜欢）
  E. 量价绑定："螺纹加到 70t 再让 20" → 求量型激励

Order-level Profit Optimizer 按 (客户类型 + 议价轮次 + 各项 headroom) 推荐 1~2 个方案给销售
```

---

### 1.7 报价后改量场景（v8 核心）

#### 1.7.1 改量的 8 种典型情景与连锁反应

| # | 改量类型 | 触发因素 | 连锁影响 | 默认处理路径 |
|---|---|---|---|---|
| 1 | 加量到下一阶梯 | 客户："加到 100t" | 单价应享受量阶梯优惠 | **自动调价** |
| 2 | 加量超库存 | 客户："加到 500t" | 单仓不足，需拼仓/调货 | **升档 Assisted / 拆批** |
| 3 | 加量超 Auto 上限 | 单笔金额 > 阈值 | 风险变高 | **升档 Assisted/Manual** |
| 4 | 减量但仍 ≥ MOQ | 50t → 35t | 可能跌出量阶梯 → 单价上浮 | **自动调价** |
| 5 | 减量跌穿 MOQ | 50t → 20t（MOQ=30t） | 不能正常过磅 | **给客户 2 选项**（加量/切不过磅）|
| 6 | 减量到 0 | "算了不要了" | 实际是取消 | **走撤单流程** |
| 7 | 议价中改量 | 让步姿势"加 20t 再让 5" | 让步预算按新量重算 | **议价引擎处理** |
| 8 | 锁价后改量 | 已锁价 12h 后"改 80t" | **违约性请求** | **必须销售明确批准** |

#### 1.7.2 改量幅度档位

| 幅度 | 处理 |
|---|---|
| ±10% 内 + 库存充足 + 不破规则 | **自动重算 + 展示对比 + 客户确认** |
| ±10~30% / 阶梯档位变化 / 拼仓变化 | **重算 + 销售确认（Assisted）** |
| > 30% / 跌穿 MOQ / 库存不足 / 超 Auto 上限 | **特殊处理（客户选项 OR 转销售）** |
| 同 Quote 改量 ≥ 3 次 | **转销售 + ghost_score↑ + 触发软干预 L4** |

#### 1.7.3 锁价后改量的特殊规则

锁价是双方承诺。客户锁后改量属于违约性请求：

| 操作 | 处理 |
|---|---|
| 锁后**加量** | 通常允许；新增部分按当前价（不一定享受新量阶梯，销售决定） |
| 锁后**减量** | 等于违反承诺；单价保持原档（不享受减量退价）；销售可豁免 |
| 锁后**改规格/目的地/产地** | 本质是新单；原锁价作废，作为新询价处理 |

锁后改量必经销售审批（Bot 不自动处理）。

---

### 1.8 转人工触发场景与销售路由（v10 核心）

#### 1.8.1 转人工触发场景全集（v1~v10 累积）

| 来源 | 触发条件 | 引入版本 |
|---|---|---|
| **无库存** | 仓库都没货，需调货或等待 | **v10** |
| **未定价** | 系统里没有该规格价格，无法机器报价 | **v10** |
| 黑名单 cash_only / refuse | 客户名单等级 | v6 |
| 超 Auto 金额上限 | 单笔过大 | v6 |
| 大单 / 非标 / 复杂组合 | 风险高 | v6 |
| 议价让步预算耗尽 | 议价 5 轮以上 | v7 |
| 议价中客户情绪激动 | LLM 情绪检测 | v7 |
| 锁价后改量 | 违约性请求 | v8 |
| 改量超库存 / 拼仓拆批 | 库存不足 | v8 |
| 改量频次超限 ≥ 3 次 | 防薅 | v8 |
| 客户主动要求 | "找小张" | 各版本 |
| 询价解析失败 / 置信度低 | 多模态识别不清 | v4 |
| Qwen-VL 故障 | 视觉服务降级失败 | v9 |
| 软干预 L4 销售关怀 | ghost_score 高 | v6 |

**统一入口**：所有触发都进 Lead Routing Engine，由它决定转给谁。

#### 1.8.2 销售路由两层逻辑

**第一层（默认）：绑定销售优先**

| 情况 | 处理 |
|---|---|
| 客户已绑定销售 + 销售在线 + 未超载 + 有该品类授权 | 直接转 |
| 客户已绑定销售 + 销售离线/请假/超载 | 走第二层（团队池） |
| 销售已离职 | 走第二层 + 通知 CRM 解绑提醒 |
| 战略/VIP 客户 | 强制走资深销售（可绑也可池中筛选） |

**第二层：团队池 + 竞争规则（加权抽签）**

- 不是"分数第一直接拿"，避免马太效应
- 用 softmax + 温度参数 T 控制差异：
  - T 小（0.5）：精英拿大头（激励）
  - T 大（2.0）：更平均（公平）
  - 公司可调 T 参数
- 加配额限制：每销售每天 N 个未绑定客户新单上限
- 加品类授权：销售必须有该品类销售许可才能进候选池

#### 1.8.3 销售评分（多维综合）

| 维度 | 权重示例 | 含义 |
|---|---|---|
| 成交转化率 | 25% | 询价→下单成功率 |
| 月成交吨位/金额 | 20% | 业绩规模 |
| 回款 DSO（反向） | 15% | 越短越好 |
| 客户 NPS / 满意度 | 10% | 客户体验 |
| 平均响应时长（反向） | 15% | 接单快慢 |
| 拒单率（反向） | 10% | 不挑客户 |
| 当前负载（反向） | 5% | 别累垮 |

加权综合分（0~100），周期更新（每小时轻量、每天全量）。

#### 1.8.4 SLA + 拒单代价

| 行为 | 后果 |
|---|---|
| 收到工单 ≤ X 分钟内 ACK | 正常 |
| ACK 超时 | 自动转下一位（不影响评分） |
| 主动拒接 | 计入 `decline_rate` → 评分降 |
| 接单后及时回客户 | 响应时长指标提升 → 评分↑ |
| 接单后长时间不回 | 客户满意度↓ + 工单超时 → 评分↓ |

形成正向激励：**愿意接 + 接得好 → 评分高 → 优质单更多 → 业绩更好**。

#### 1.8.5 防失控机制

- **优秀销售爆单防护**：单日配额 + 超配额自动溢出到次优销售
- **新人保底**：每天保证 N 个工单分给底部 30% 销售（学习机会）
- **客户绑定保护**：绑定关系一经建立不被池随机覆盖；除非客户主动要求换销售或销售离职
- **抢单防火墙**：销售之间不能互相"截胡"；池抽签结果不可手动改（除非主管授权）
- **战略客户特权**：必转 VIP-eligible 销售，不走概率抽签

---

### 1.9 替代料推荐场景与决策（v11 核心）

#### 1.9.1 替代关系 7 种类型

| 类型 | 例子 | 推荐力度 | 风险 |
|---|---|---|---|
| **同档异厂** | 沙钢 HRB400 Φ25 ↔ 永钢 HRB400 Φ25 | 强推（无风险） | 仅个别工程要求指定钢厂 |
| **同档异定尺** | 12 米 ↔ 9 米 ↔ 倍尺料 | 中推 | 切割损耗、客户加工设备适配 |
| **同档品质差** | 标准品 ↔ 长锈轻锈 ↔ 优级 | 中推 | 价差大但外观/锈蚀差异，需客户接受 |
| **向上兼容** | HRB400 → HRB500 / Q235 → Q355 | 弱推 | 更贵；仅工程必要 |
| **向下兼容** | HRB400E → HRB400（去抗震要求） | 谨慎推 | 抗震/工程项目通常不允许 |
| **规格相近** | Φ25 ↔ Φ22（小幅差异） | 谨慎推 | 工程禁；DIY/装修可 |
| **国标/外标等价** | Q235B ↔ SS400 ↔ A36 / S235JR | 中推 | 出口贸易常用；客户验收标准 |

#### 1.9.2 推荐时机 3 种场景

| 客户原 spec 状态 | 默认策略 | 例外 |
|---|---|---|
| **完全有货** | **不主动推**（避免暴露"还有更便宜的"砸自己价） | 议价/砍价时可作"价值替代"使用；战略客户主动推建立信任 |
| **部分有货** | **推拼方案**（原 spec 30t + 替代 20t） | 客户偏好同档时，全替代也可考虑 |
| **完全无货** | **替代是主推**（替代不成才转人工） | 客户类型 / 历史 / 用途否决时直接转人工 |

#### 1.9.3 客户接受度 3 维过滤

| 维度 | 含义 | 推荐影响 |
|---|---|---|
| **客户类型** | 工程方/施工方 vs 贸易商/加工厂 | 工程方默认不推向下/相近；贸易商可推 |
| **历史接受** | 客户档案里历次替代接受/拒绝记录 | 历史拒绝 ≥ 3 次 → 该类型替代永不推 |
| **用途场景** | LLM 从对话抽取（"建筑工地"/"做配件"/"出口"） | 工程类用途 → 严格模式；非工程用途 → 灵活 |

客户档案新增字段：
```yaml
substitute_preferences:
  accepts_substitute: true | false | conditional
  accepts_types: [同档异厂, 同档异定尺, ...]
  rejects_types: [向下兼容, 规格相近]
  history:
    - {date, original, substitute, action: accepted|rejected, reason}
```

#### 1.9.4 三个潜在大坑与规避

| 坑 | 后果 | 规避 |
|---|---|---|
| **偷偷换货** | 客户拿到货发现不是他要的 → 投诉/拒收 | 推荐时必须**明示**"替代料+替代关系+为什么可替"；合同条款单独标注；客户必须明确确认 |
| **工程方按图纸拒收** | 替代料被工程方退货 | 客户档案标 `accepts_substitute=false` 永不推；用途含"工程/桥梁/基建" → 谨慎模式 |
| **销售当原料卖** | 合规风险 | Bot 内永远显示"替代方案 + 原品对比"；合同条款单独条款；交付时双方签收清单审计 |

---

## 2. 通道选型 / 3. 总体架构（v6 基础 + v7 新模块）

```
   ...（v1~v6 模块同前）...

   ─────────── v7 新增议价模块 ───────────

   ⑭ Negotiation Session Manager（议价会话管理器）
      - 状态机：OPENING → COUNTERED → CONCEDED → AGREED / IMPASSE / ESCALATED
      - 每轮 offer 历史（双方）
      - 各 item 让步累计
      - 触发议价/退出议价的判定

   ⑮ Concession Strategy Engine（让步策略引擎）
      - 按客户类型 × 当前轮次 → 让步曲线
      - 让步预算管理（max_concession = current_margin - min_margin）
      - 让步幅度递减原则
      - 让步节奏（求利型慢让，求量型量绑定，战略型一次到位）

   ⑯ Order-level Profit Optimizer（整单利润优化器）
      - 视角：整单 vs 单 item
      - 跨 item 让步分配（保护薄毛利项，让厚毛利项让）
      - 整单底线 + 整单加权毛利率
      - 推荐 1~2 个让步方案给销售

   ⑰ Counter-offer Generator（多维反报价生成器）
      - 8 种让步货币（价/量/时/运/期/质/赠/组）
      - 组合搜索（在让步预算内找等价组合）
      - 按客户敏感度排序：求利型先价，求量型先量，战略型先时/期/组

   ⑱ Bargaining LLM Layer（议价话术 LLM）
      - 上下文：议价历史 + 客户画像 + 让步预算 + 推荐方案
      - 输出话术：礼貌、留余地、不卑不亢、不漏底
      - 按客户类型切换语气（求利型聊性价比，求量型聊增量收益，战略型聊长期）

   ─────────── 与 v6 的衔接 ───────────
   议价模块在 v6「报价档位 (Auto/Assisted/Manual)」之后激活：
     Auto    → 客户砍价 → 直接跳 Assisted（销售介入）
     Assisted → 议价引擎为销售出方案 → 销售确认
     Manual  → 销售全程，Bot 仅记录议价历史 + 提示
```

   ─────────── v8 新增改量模块 ───────────

   ⑲ Quote Lifecycle State Machine（报价生命周期状态机）
      - DRAFT → ACTIVE → AMENDING → RESCORED → ACTIVE'(v+1)
      - ACTIVE → LOCKED → (改量需销售审批)
      - ACTIVE / LOCKED → EXPIRED / SUPERSEDED
      - 每次改量生成新版本，不覆盖旧版（审计 + 对比）

   ⑳ Quote Amendment Engine（改量引擎）
      - 解析客户改量意图（加/减/置换）
      - 合规校验（MOQ / 库存 / Auto 上限 / 整单底线）
      - 触发重新定价（量阶梯 / 库存拼仓 / 整单 Optimizer）
      - 与议价引擎协同（议价中的改量作为 offer）
      - 频次防护（同 Quote 改量 ≥ N 次→转销售 + ghost↑）

   ─────────── v10 新增转人工路由模块 ───────────

   ㉑ Lead Routing Engine（转人工 + 销售分派引擎）
      - 统一接管所有转人工触发场景
      - 两层路由：绑定销售优先 → 团队池加权抽签
      - 加权抽签：softmax + 温度参数 + 配额限制
      - SLA + ACK 超时自动转派
      - 战略/VIP 客户强制资深销售

   ㉒ Sales Performance Score（销售竞争评分）
      - 多维综合分（转化率/吨位/回款/NPS/响应/拒单/负载）
      - 团队内排名（团队 leaderboard）
      - 周期更新（小时增量 + 日全量）
      - 销售可见自己分数（激励），客户不可见

   ㉓ Sales State Manager（销售状态管理）
      - 在线/忙/离开/离线/请假
      - 当前未完成工单数（负载）
      - 每日配额使用情况
      - 品类销售授权
      - VIP 资格标记


   ─────────── v11 新增替代料模块 ───────────

   ㉔ Substitute Knowledge Base（替代料知识库）
      - 7 种替代关系（同档异厂/异定尺/品质/向上向下/相近/外标等价）
      - 替代规则与兼容性约束
      - 行业标准等价表（GB ↔ ASTM ↔ JIS ↔ EN）
      - 历史成交学习（哪些客户接受过哪些替代）
      - 后台运营可维护

   ㉕ Substitute Match Engine（替代料匹配引擎）
      - 输入：客户原 spec + 用途场景 + 客户接受度
      - 查 KB → 与库存匹配 → 价格对比 → 接受度过滤
      - 输出：排序候选 + 推荐理由

   ㉖ Substitute Recommendation Strategy（推荐策略引擎）
      - 决定推不推：客户原 spec 状态（全/部分/无）
      - 决定推什么：客户类型/历史/用途
      - 决定怎么推：明示替代关系 + 价格对比 + 接受度
      - 与议价引擎协同（议价中替代料作为让步货币）



---

## 4. 关键流程（v7 新增议价流程）

### 4.2 询价 → 报价（v6 流程不变）

略。

### 4.8 议价（v7 新增）

```
═══ 触发议价 ═══════════════════════════════════════════════
客户对 Bot 已发出的报价做出非接受响应：
  - "便宜点"  / "再降 X" / "X 家 3780"
  - "再加 20t 行不行" / "送货吗" / "锁久点"
   ↓
LLM 意图识别 → 进入 NEGOTIATION
   ↓
Negotiation Session 状态：OPENING → COUNTERED

═══ 阶段 1：解析客户 offer ════════════════════════════════
LLM 抽取议价要素：
  - 砍价幅度（绝对 / 相对）
  - 砍价目标 item（默认当前 item，或客户明确指定）
  - 附加条件（量、时、运、期、质等）
  - 提及的竞品（如有）→ 记录到 leads
   ↓
归一化为 CustomerOffer 结构：
  {
    round: 2,
    target_item_id: ITEM-001,
    proposed_price: 3780,
    proposed_qty: 50 (不变),
    requested_terms: ["送货","账期30"],
    competitor_mentioned: "X家 3780",
    tone: 强硬 | 试探 | 友好
  }

═══ 阶段 2：客户画像 + 议价历史 ════════════════════════════
Customer Profile + Negotiation Session 提供：
  - 客户类型（求利/求量/战略/新客/流失）
  - 历史议价转化率
  - 当前 Topic 中议价轮次（第几轮）
  - 已让步累计 vs 让步预算
  - 整单 items 及各项 headroom

═══ 阶段 3：让步策略选择 ═════════════════════════════════
Concession Strategy Engine 算：

  1. 检查让步预算
     remaining = max_concession - total_spent
     若 remaining ≤ 0 → 不让步，转价值替代或转人工

  2. 计算本轮建议让步幅度（递减原则）
     round=1: 0.50 × remaining
     round=2: 0.25 × remaining
     round=3: 0.10 × remaining
     round=4: 0.05 × remaining
     round≥5: 不让步，转人工

  3. 按客户类型调整
     求利型: 单价让步占主，60%
     求量型: 量绑定占主，价让步小
     战略型: 价 + 时 + 期组合
     新客:   保持首报价（除非 ≤2 元小让）
     流失:   一次性给到位

═══ 阶段 4：整单视角（Order-level Optimizer） ════════════
若整单含多 item：
  - 评估每 item 的 margin headroom
  - 跨 item 重新分配让步
  - 输出 1~2 个推荐分配方案
  
示例：客户砍螺纹 ¥30
  方案 A：螺纹直让 ¥20（不直接给到客户要的 30）
  方案 B：螺纹保价 + 中板让 ¥40（中板毛利厚） → 整单让 ¥800 ≈ 螺纹 ¥16

═══ 阶段 5：多维反报价生成 ════════════════════════════════
Counter-offer Generator 搜索：

  对于"客户砍 ¥30"，生成多个等价反报价：
    A. 让 ¥15/t（一半价让步）
    B. 让 ¥10/t + 锁价延长 24h
    C. 让 ¥10/t + 免运费（按当前距离 ≈ ¥20/t 等价）
    D. 保价但加 20t 再让 ¥20（量绑定）
    E. 保价 + 整单从其他 item 让步等价
    F. 保价但赠送材质书 + 优先发车

  按客户类型筛选 Top-2

═══ 阶段 6：档位决策 + 销售确认 ═══════════════════════════
档位 = Auto：
  - 客户砍价 → 自动升档到 Assisted（不让 Bot 自己议价）
  - 议价始终需要销售点头

档位 = Assisted：
  - Bot 推荐方案给销售（textcard + 一键确认）
    «客户张总（求利型/普通）议价（第 2 轮）
     原报价 ¥3,820
     客户出价 ¥3,780（砍 ¥40）
     ──────
     方案 A（推荐）：让 ¥15 → ¥3,805，附 24h 锁价
     方案 B：让 ¥10 + 免运 ≈ 让 ¥30
     方案 C：保价但加 20t 再让 ¥20
     ──────
     让步预算：剩 ¥45/t
     竞品：X 家 ¥3,780（系统标注：该客户半年内 3 次提同样 X 家但未走 X 家）
     [发方案 A] [发方案 B] [发方案 C] [自定义] [转人工]»
  - 销售 30 秒内选 → Bot 发话术给客户
  - 销售可"自定义" → 弹简易表单填新方案

档位 = Manual：
  - Bot 不出方案，只把议价上下文 + 历史给销售
  - 销售自己回

═══ 阶段 7：客户回复 → 进入下一轮 OR 收口 ════════════════
客户回："3795 我就下"
  → 解析为 round=3 的 CustomerOffer
  → 回到阶段 2，循环

客户回："好"
  → 状态 AGREED
  → 询问"是否锁价并安排下单"，进入下单流程

客户长时间不回 / "再考虑"：
  → IMPASSE 状态，记录但保留议价 session 24h
  → Soft Influence: 锁价过期前提醒一次

═══ 阶段 8：退出议价 ═══════════════════════════════════════
退出条件：
  - AGREED 客户接受 → 进入下单/锁价
  - 触达成本红线 → 转销售人工
  - 议价轮次 ≥ 5 → 转人工
  - 客户多次提虚假竞品 → 转人工
  - 客户暴怒/施压（LLM 情绪检测） → 转人工 + 提示销售
  - 销售主动接管

═══ 全程审计 ══════════════════════════════════════════════
所有 offer 双向落 negotiation_log：
  - 每轮 customer_offer / system_response / chosen_strategy / sales_action
  - 让步预算使用情况
  - 最终结果（AGREED/IMPASSE/ESCALATED）
用途：纠纷追溯 + 策略复盘 + 销售培训
```

### 4.9 整单议价（多 item 同时谈）

```
客户："这三个品种加起来给个底价"
   ↓
Order-level Profit Optimizer：
  - 计算整单当前总价、总毛利、各项 headroom
  - 按客户类型选分配模式：
      求利型: 主项小让 + 配项多让
      求量型: 量阶梯整体激活
      战略型: 整单一口价
  - 输出整单方案 1~2 个
   ↓
Bot/销售发整单方案，客户对整单回价
   ↓
若客户仍逐项砍 → 进入 4.8 单 item 议价循环
若客户接受整单 → 整单 AGREED → 一次下单
```

### 4.10 改量流程（v8 新增）

```
═══ 阶段 0：触发改量 ═══════════════════════════════════════
客户已收到 Bot 报价 Q-001 v1（50t @ ¥3,820 锁 24h）
客户："改成 80t 吧" / "改成 20t" / "改成 500t" / "改到 0"
   ↓ LLM 意图识别 = AMEND_QUOTE
取出当前活跃 Quote 版本 → 进入 Quote Amendment Engine

═══ 阶段 1：状态合规检查 ═══════════════════════════════════
当前 Quote 状态:
  ACTIVE         → 允许改量
  LOCKED         → 触发"锁后改量"特殊流程（销售审批）
  EXPIRED        → 报价已过期，提示客户重新询价
  IN_NEGOTIATION → 改量作为议价的 offer（走 4.8）
  AGREED / 已下单 → 拒绝；引导走"订单变更"工单

═══ 阶段 2：解析与归一 ═════════════════════════════════════
amendment = {
  quote_id, version_from: 1,
  field: qty, value_from: 50, value_to: 80,
  delta: +30, delta_pct: 0.6,
  attached_terms: [], source: customer
}

═══ 阶段 3：合规校验（5 项）═══════════════════════════════
1. MOQ：new_qty ≥ 品类 MOQ？
2. 库存：new_qty ≤ 当前可拼上限？
3. Auto 上限：new_qty × price ≤ Auto 金额上限？
4. 整单底线：属整单时，新量是否破整单加权毛利？
5. 改量频次：同 Quote 改量次数 ≥ 阈值？

═══ 阶段 4：重新定价 ═══════════════════════════════════════
Pricing Engine 用新量重跑：
  - 量阶梯档位变化（50t @ 3820 → 80t @ 3805）
  - 库存匹配方案变化（单仓 → 拼仓）
  - 运费变化（大量单车装不下）
  - 整单 Optimizer 重算（若属整单）
  - 议价让步预算重算（若议价中）

═══ 阶段 5：四档决策 ═════════════════════════════════════
AUTO（自动调价）：
  幅度 ≤ ±10% + 5 项合规全过 + 非议价中 + 非锁价后

ASSISTED（销售确认）：
  幅度 10~30% / 阶梯档位变化 / 拼仓变化 / 整单接近底线 / 库存吃紧

CUSTOMER_CHOICE（给客户选项）：
  跌穿 MOQ

MANUAL（转销售人工）：
  库存不足 / 超 Auto 上限 / 锁后改量 / 频次超限 / 整单破底

REJECT（拒绝并撤单）：
  new_qty == 0

═══ 阶段 6：生成新 Quote 版本 ═════════════════════════════
Q-001 v1 → AMENDING
生成 Q-001 v2 (DRAFT) → 新量、新单价、新组合、新锁价、变更原因

客户接受 → v1 状态 SUPERSEDED, v2 状态 ACTIVE
客户拒绝 → v2 状态 CANCELLED, v1 保持 ACTIVE
全程留 quote_amendment_log

═══ 阶段 7：客户回执（差异对比） ══════════════════════════
"📋 已为您调整报价：
 ┌─ 原方案 v1
 │  HRB400 螺纹钢 Φ25mm 沙钢 武汉 50t × ¥3,820 = ¥191,000
 │  锁价剩 18h
 ├─ 新方案 v2
 │  80t × ¥3,805 = ¥304,400（量阶梯 -¥15/t 省 ¥1,200）
 │  锁价继承剩余 18h
 │  库存：拼仓 江夏 50t + 襄阳 30t（含倒短 ¥30/t）
 └─
 回复『确认』锁价；『取消』保持原方案；『修改 XX』继续调整"

═══ 阶段 8：议价中改量（与 4.8 协同）══════════════════════
Quote 在议价中（NegotiationSession 活跃）：
  - 客户改量解析为 NegotiationRound 的 customer_offer
  - 让步预算按新量重算
  - Counter-offer Generator 量绑定方案优先级提升
  - 不走 4.10 自动重算，融入 4.8

═══ 阶段 9：跌穿 MOQ ═══════════════════════════════════════
new_qty=20t < MOQ=30t
Bot 给 3 选项：
  A. 加到 30t 起按吨过磅 ¥3,820/t
  B. 按 20 支不过磅 ¥920/支
  C. 取消调整，保持原 50t

═══ 阶段 10：库存不足 ═══════════════════════════════════════
new_qty=500t，可拼仓上限 200t → 转 Assisted
销售看到拆批方案 A/B/C，一键选择

═══ 阶段 11：锁价后改量 ═══════════════════════════════════
LOCKED 状态改量：
  - 直接转 Assisted（销售决定批/拒/部分批）
  - 加量：销售决定新增部分是否享受量阶梯
  - 减量：默认不享受退价；销售可豁免
  - 改规格/目的地/产地：原锁价作废，作为新询价
  - 所有锁后改量留 override 原因审计

═══ 阶段 12：频次防护 ═══════════════════════════════════════
同 Quote 改量 v1→v2→v3:
  ≥ 3 次：Bot 提示"已多次调整，让小张帮您理一下"，转人工
  ≥ 5 次：ghost_score↑，触发软干预 L4
  改量后立即下单 vs 继续问：写入 customer_profile.behavior
```

### 4.11 转人工 + 销售分派流程（v10 新增）

```
═══ 阶段 0：触发转人工 ═══════════════════════════════════════
来自 13 种触发场景之一（见 1.8.1）：
  无库存 / 未定价 / 黑名单 / 大单 / 议价超限 / 锁后改量 / ...
   ↓
统一入口：Lead Routing Engine.route(handoff_request)

═══ 阶段 1：构造 HandoffRequest ═══════════════════════════════
{
  trigger_type: NO_INVENTORY | NO_PRICING | BLACKLIST | LARGE_ORDER |
                NEGOTIATION_EXHAUSTED | EMOTION_ALERT | LOCKED_AMENDMENT |
                AMEND_FREQ_EXCEEDED | CUSTOMER_REQUEST | PARSE_FAIL | ...,
  customer_id, inquiry_id?, quote_id?,
  urgency: HIGH | NORMAL | LOW,
  category, qty, dest_city,
  conversation_excerpt: "最近 5 条消息",
  customer_profile_snapshot,
  required_skills: [螺纹, 大单, VIP, ...],
  prefer_bound: true | false
}

═══ 阶段 2：第一层 — 绑定销售优先 ════════════════════════════
查 customer_profile.bound_salesperson_id
   ↓
若有绑定 + 在线 + 未超载 + 有品类授权:
   → 直接 assign(bound_sales)
   → 跳到阶段 4 推送
   
若有绑定但不可用（离线/请假/超载/无授权）:
   → 走第二层；通知该销售"客户被转给团队"

若无绑定:
   → 走第二层

═══ 阶段 3：第二层 — 团队池加权抽签 ═══════════════════════════
候选池筛选：
  pool = team_members
    .filter(in_team(customer.team_or_region))
    .filter(category_authorized(inquiry.category))
    .filter(online OR partially_available)
    .filter(workload < limit)
    .filter(daily_quota_remaining > 0)
    .filter_if(customer.type in [strategic,vip], vip_eligible)

若 pool 为空:
  - 放宽筛选条件（如允许 busy 状态）再试
  - 仍空 → 推工单到团队群 + 主管认领
  - 极端：值班销售兜底

抽签:
  scores = [s.performance_score for s in pool]
  # softmax with temperature
  weights = softmax(scores / T)
  selected = weighted_pick(pool, weights)
  
  selected.daily_quota_used += 1
  selected.workload += 1

═══ 阶段 4：推送到选定销售 ═══════════════════════════════════
Bot → 销售（企业微信 textcard）：
  «客户询价转您处理
   客户：张总（武汉 / 量型 / 普通 / 信用正常）
   触发：无库存（螺纹 HRB400 25mm 沙钢 200t）
   原话："要 200t 螺四 25 沙钢的"
   候选库存：江夏 30t + 江岸 20t + 襄阳 150t，余 0
   建议：1) 调货 5-7 天 2) 改产地 3) 拆批
   SLA：5 分钟内 ACK，30 分钟内回客户
   [我接单] [转给同事 XX] [拒接（请填原因）]»
   ↓
Bot → 客户：
  "您的询价已转销售小张为您处理，
   小张会在 30 分钟内联系您～"

═══ 阶段 5：SLA + ACK 监控 ═══════════════════════════════════
T+0    工单推送
T+5min 销售未 ACK → 自动转池下一位
                     原销售不扣分（系统认为"没看到"）
T+15min 已 ACK 但未回客户 → 内部提醒
T+30min 仍未回客户 → 升级到销售主管 + 客户安抚消息
T+60min 客户体验严重受损 → 团队负责人介入

═══ 阶段 6：销售响应 ═══════════════════════════════════════
销售点 [我接单] →
  - 工单 status = ASSIGNED
  - 销售可在 Bot 里 /reply 给客户
  - 后续对话由 Bot 转发（销售↔客户）

销售点 [拒接 + 原因] →
  - decline_rate += 1
  - 工单回池重新抽签（排除该销售）
  - 拒接原因落审计

销售点 [转给同事 XX] →
  - 须主管授权（防止互相甩单）
  - 二级转派审计

═══ 阶段 7：闭环 ═══════════════════════════════════════════
工单完成（成交 / 客户放弃 / 转其他销售）：
  - 工单 status = CLOSED
  - 销售负载 -= 1
  - 性能指标更新（成交+1 / 响应时长记录 / NPS 收集）
  - performance_score 异步重算

═══ 阶段 8：监控与复盘 ═══════════════════════════════════════
管理后台：
  - 团队 leaderboard（销售排名实时刷新）
  - 工单分布热力图（看是否被几个销售垄断）
  - 拒单率 / 超时率 / NPS 看板
  - 抽签温度参数 T 可视化调整
```

### 4.12 替代料推荐流程（v11 新增）

```
═══ 阶段 0：进入条件 ════════════════════════════════════════
parse_inquiry 完成 → InquiryItem 含 (category, grade, spec, origin, length, ...)
当前在 报价 / 议价 / 改量 流程中
   ↓
进入 Substitute Recommendation Strategy 评估

═══ 阶段 1：先查原 spec 库存状态 ═══════════════════════════════
Inventory Matcher 查原 spec:
  - FULL_STOCK   完全有货
  - PARTIAL_STOCK 部分有货
  - OUT_OF_STOCK 完全无货

═══ 阶段 2：决定要不要推替代 ════════════════════════════════
FULL_STOCK + 非议价/非砍价 + 非战略客户:
  → 不推（避免砸自己价）
FULL_STOCK + 在议价中（让步货币）:
  → 可推（让 quality_upgrade 货币使用）
PARTIAL_STOCK:
  → 推"拼方案"（原 spec 30t + 替代 20t）
OUT_OF_STOCK:
  → 替代是主推
战略客户 + FULL_STOCK + 替代价更优:
  → 主动推（建立长期信任）

═══ 阶段 3：客户接受度过滤 ════════════════════════════════════
查 customer_profile.substitute_preferences:
  - accepts_substitute == false → 不推；走转人工流程
  - rejects_types 列表 → 这些替代类型不推
  - 历史拒绝 ≥ 3 次同类替代 → 该类不推
   ↓
LLM 从近期对话抽取"用途":
  - 关键词：建筑/工地/桥梁/基建/出口/检验/图纸/工程
  → 严格模式：仅推同档异厂、外标等价；不推向下/相近
  - 关键词：装修/配件/加工/DIY/自用
  → 灵活模式：所有 7 类都可推

═══ 阶段 4：替代候选生成 ════════════════════════════════════
Substitute Match Engine:
  查 KB 找候选 → 按用途/客户接受度过滤 → 与库存匹配
  对每个候选：
    candidate = {
      sub_category, sub_grade, sub_spec, sub_origin, sub_length,
      relation_type: 同档异厂/异定尺/品质/向上/向下/相近/外标等价,
      relation_reason: "Q235B 与 SS400 国标日标等价",
      stock_available: 50t,
      base_price: 3795,
      price_diff_vs_original: -25,  // 比原 spec 便宜 25
      our_margin_diff: +10,          // 我方毛利差异（销售看的）
      acceptance_score: 0.85,        // 综合接受度（用途+客户类型+历史）
      caveat: "标准品转长锈轻锈，外观有锈迹但结构无影响"
    }

═══ 阶段 5：候选排序 ══════════════════════════════════════════
排序权重：
  - 客户接受度 0.30
  - 库存充足度 0.25
  - 替代等级（强/中/弱推）0.20
  - 价格优势 0.15
  - 我方毛利 0.10

输出 Top-3 候选

═══ 阶段 6：组装推荐方案 ════════════════════════════════════════
按场景生成 RecommendationPlan:

  PARTIAL_STOCK:
    plan = [
      原 spec 30t @ ¥3,820,
      替代 Y 20t @ ¥3,795（同档异厂 永钢），
      总 50t 均价 ¥3,810
    ]

  OUT_OF_STOCK:
    plan_A = [全部替代 永钢 HRB400 Φ25 50t @ ¥3,795]
    plan_B = [全部替代 沙钢 HRB400E Φ25 50t @ ¥3,850 抗震]
    plan_C = [全部替代 沙钢 长锈轻锈 50t @ ¥3,720]

  议价中（让步货币）:
    "保持原价不变，但产品升级到优级品（同价升级）"
    或 "保持原价，换更便宜替代 → 总价省 ¥500"

═══ 阶段 7：四档决策（与 v6 报价档位一致）═══════════════════════
推荐方案的执行档位：
  AUTO:
    - 同档异厂 + 客户类型贸易商/加工厂 + 历史接受类似 → Bot 直接推
  ASSISTED:
    - 大多数情况 → Bot 出方案 → 销售一键确认 → 发客户
  CUSTOMER_REVIEW:
    - 客户档案 conditional → 推时附 "请确认是否接受"
  MANUAL:
    - 工程类用途 / 战略客户 / 大单 → 转销售人工推
  NO_RECOMMEND:
    - accepts_substitute=false → 转人工 + 不带替代方案
    - 替代候选库存也不足 → 转人工调货

═══ 阶段 8：客户知情决策 ═══════════════════════════════════════
Bot 发客户（明示替代关系）：
"📋 您要的螺纹 HRB400 Φ25 沙钢 50t 库存不足（仅 30t）
余 20t 为您匹配：
  ┌──────────────────────────────
  │ 方案 A（推荐）拼方案
  │   原 spec 30t × ¥3,820 + 永钢 HRB400 Φ25（同档异厂）20t × ¥3,795
  │   总 50t 均价 ¥3,810 总价 ¥190,500
  │   永钢与沙钢均为国标 HRB400，可互换使用
  ├──────────────────────────────
  │ 方案 B 全替代 永钢
  │   永钢 HRB400 Φ25 50t × ¥3,795 = ¥189,750
  │   省 ¥1,250
  ├──────────────────────────────
  │ 方案 C 标准 + 长锈轻锈
  │   原 spec 30t × ¥3,820 + 沙钢 同规格长锈轻锈 20t × ¥3,720
  │   总 50t 均价 ¥3,780 总价 ¥189,000
  │   长锈轻锈不影响结构性能，外观有锈迹
  └──────────────────────────────
  回复『方案 X』选择；或『不要替代』转销售调货
"

═══ 阶段 9：客户响应记录到偏好 ═══════════════════════════════════
客户选 X / 拒绝 / 提问 → 记录到 customer_profile.substitute_preferences.history
  → 偏好库长期学习
  → 下次同类替代的 acceptance_score 自动调整

═══ 阶段 10：合同/审计 ════════════════════════════════════════
客户确认 → 生成订单
合同条款里**单独标注替代关系**："本订单含替代料：永钢 HRB400 Φ25 替代沙钢 HRB400 Φ25，
                              客户已知情并同意，两者均为国标 HRB400 同档替代"
交付时双方签收清单留档；事后投诉时有据可查
```

---

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

**5.26.3 整单底线**

整单加权毛利率 < min_blended_margin → 拒绝继续让步 → 转销售。

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

**5.32.2 决策树**
```
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
     new_qty 超库存上限     → MANUAL
     new_qty 超 Auto 上限   → ASSISTED 或 MANUAL（看金额）
     new_qty == 0           → REJECT + 撤单
     其余                   → ASSISTED

修饰：
  锁价后                → MANUAL（销售审批）
  议价中                → 不走本流程，转 4.8
  改量次数 ≥ 3          → MANUAL + 软干预
  整单底线破            → MANUAL
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

---

## 6. 钢铁贸易话术与体验（v7 增量）

### 6.5 议价话术模板

#### 求利型 - 第 1 轮
```
张总好，您说的价格我跟仓库核了下，咱们这批沙钢 HRB400 25mm 的拿货成本就在那儿。
我尽力帮您争取，让 ¥15/t 到 ¥3,805，锁价也帮您从 24h 拉到 48h，您慢慢看。
现在这价位上下游都在挺，长不了。
```

#### 求利型 - 第 2 轮（递减让步）
```
张总，前面已经帮您让到 ¥3,805 了。这次再让 ¥5 到 ¥3,800，
另外材质书我帮您准备好，到货当天就给您；发车也排前面，48 小时到武汉。
您看这样行不行？
```

#### 求利型 - 第 3 轮（价值替代）
```
张总，价上确实没再多空间了。这样，我看您是要送到江夏对吧，运费这块儿
我帮您兜下来（约 ¥20/t），相当于您实际拿到 ¥3,780 的价位。
咱们价直接到这了，下午能锁吗？
```

#### 求量型 - 量价绑定
```
张总，单价上让多了仓库这边就紧了。换个思路：
您把这批从 50t 加到 70t，单价我帮您压到 ¥3,795，多出来这 20t 也按同价，
等于均价直接到位。库里正好有，您看？
```

#### 战略型 - 一次到位
```
李总，您是咱们老合作伙伴，这次不绕弯子，直接给到 ¥3,775，
锁价 72 小时；如果您能签个本月 500t 的量保，我再帮您锁个全月。
```

#### 新客 - 不缠斗
```
王总，您首单的价 ¥3,810 已经是给您新客户专批的，
材质书优先开、首车优先发，这些我都给您安排上。
价上恕我帮不上更多了，您看方便先来 30t 试试合作吗？
```

#### 流失客户 - 唤回
```
赵总，好久没听到您声音了！这次给您直接到唤回价 ¥3,762，
锁 12 小时，您看下今天能定吗？回头我跟小张专门盯下您这单。
```

#### 黑名单 cash_only 客户施压
```
张总，咱们这边规则您也清楚，这单需要现款，我能给的就是 ¥3,935 这条线。
您方便资金到位的话，我马上锁价给您发。
```

#### 客户提竞品（系统记录："X家半年内 3 次同样话术未走 X 家"）
```
张总，X 家 ¥3,780 这个我了解。咱们家这批是沙钢直发，
材质书一炉一证、24h 内发车、48h 内到武汉江夏。
真要按 ¥3,780 走，我得跟老板汇报一下，您容我半小时回您可以吗？
```
> 备注：这种话术给销售一个"缓冲窗"，让客户冷却 + 销售判断是否真给批价。

#### 转人工
```
张总，您这个方案咱们得让销售小张当面跟您细聊一下。
他十分钟内联系您，您先等下。
```

### 6.6 改量话术模板（v8 新增）

#### 加量到下一阶梯（AUTO）
```
📋 已为您调整报价：
原方案：HRB400 螺纹钢 Φ25mm 沙钢 50t × ¥3,820 = ¥191,000
新方案：HRB400 螺纹钢 Φ25mm 沙钢 80t × ¥3,805 = ¥304,400
       （量阶梯 -¥15/t，省 ¥1,200）
锁价：继承剩余 18 小时
回复『确认』或『修改 XX』继续调整
```

#### 加量超库存（MANUAL）
```
您要加到 500t，目前即时可发 200t（江夏 30 + 江岸 20 + 襄阳 150）
余 300t 需从沙钢厂调货，约 5~7 天到货
销售小张稍后联系您，看是拆批先发还是整批等货
```

#### 减量但仍 ≥ MOQ（AUTO 上浮）
```
📋 已为您调整：
原方案：50t × ¥3,805（含量阶梯）
新方案：35t × ¥3,820（量阶梯档位变化）
锁价：继承剩余 18 小时
说明：因量下调跌出 50t+ 阶梯，单价回到标准档
回复『确认』或『修改』
```

#### 跌穿 MOQ（CUSTOMER_CHOICE）
```
您调整到 20t，低于螺纹起订量 30t。两个选项：
A. 加到 30t 起，按吨过磅 ¥3,820/t
B. 按 20 支不过磅销售 ¥920/支（含不过磅成本）
C. 取消调整，保持原 50t 方案
```

#### 改到 0（REJECT + 撤单）
```
好的，已为您撤销询价 Q-001。
原报价 80t × ¥3,805 已失效。
如后续有需要欢迎再次询价～
```

#### 锁价后改量（MANUAL）
```
您当前 Quote 已在 12:00 锁价（剩 6h）。
锁后调量需要销售小张确认，已为您转告
小张稍后联系您
```

#### 频繁改量（≥ 3 次，MANUAL）
```
张总，本单已为您调整 3 次了，
让销售小张直接帮您把方案理一下，10 分钟内联系您
```

#### 议价中改量（融入议价）
```
听您的，把量加到 80t 重算：
新均价 ¥3,795/t × 80t = ¥303,600
锁价同时延长到 48 小时
您看这样可以吗？
```

#### 改量后立刻下单（积极信号）
```
好的 80t × ¥3,805 = ¥304,400 已为您锁定
请提供：
  - 收货地址（默认武汉江夏，确认请回『确认』）
  - 收货联系人电话
  - 是否需要送货上门
我帮您发起下单工单，2 小时内出合同
```

### 6.7 转人工话术模板（v10 新增）

#### 无库存
```
张总，您要的螺纹钢 HRB400 Φ25mm 200t（沙钢，武汉到货）
目前库存匹配如下：
  江夏 30t + 江岸 20t + 襄阳 150t = 200t 即时可发 ✅

抱歉刚仔细查了下，沙钢这批暂时余量不够。
已为您转销售小张，他会看是否能：
  1. 调货 5-7 天到 
  2. 改其他产地（永钢/中天）
  3. 拆批先发可用量
小张 30 分钟内联系您～
```

#### 未定价（公司没该规格挂牌价）
```
张总，您询的"无缝管 Φ159×8 304 不锈钢"是非标品种，
我系统里暂没标准报价。
已为您转销售小张专项询价，预计 30 分钟内给您方案。
小张电话 13xxxxxxxxx
```

#### 客户主动要求人工
```
好的张总，已为您转销售小张
小张 5 分钟内联系您
```

#### 绑定销售离线，转池备用销售
```
张总，您的销售小张今天休假，
今天由小李为您临时服务（已熟悉您的合作历史）
小李 30 分钟内联系您～
```

#### 战略客户（必转资深）
```
李总，您的询价已为您转**金牌销售小王**专项处理
（5 年钢贸经验、负责您所在区域 VIP 客户）
小王 15 分钟内联系您
```

#### 池抽签结果通知销售
```
[转给销售（textcard）]
客户：张总（武汉/量型/普通/信用正常）
触发：无库存（螺纹 HRB400 25mm 沙钢 200t）
原话："要 200t 螺四 25 沙钢的"
当前库存：江夏 30 + 江岸 20 + 襄阳 150，余 0
建议：1) 调货 2) 改产地 3) 拆批
SLA：5 分钟 ACK，30 分钟回客户
您的接单概率：35%（团队池抽签结果）

[我接单] [转同事 XX] [拒接（请填原因）]
```

#### ACK 超时客户安抚（自动转下一位时）
```
张总抱歉让您久等了，
正在为您匹配最合适的销售，1 分钟内回您～
```

#### 销售接单后第一句（销售可一键发送）
```
张总好，我是小张（13xxxxxxxxx），
您刚询的螺纹 HRB400 Φ25mm × 200t 我看到了，
正在为您查最优方案，5 分钟内给您具体回复～
```

#### 排队等待（团队全员忙）
```
张总，目前销售都在跟其他客户，预计 10 分钟内有人为您服务
是否需要我先记下您的联系电话，让小张稍后回拨？
```

### 6.8 替代料推荐话术模板（v11 新增）

#### 部分有货 - 拼方案（贸易商客户）
```
张总，您要的螺纹 HRB400 Φ25 沙钢 50t
当前沙钢库存仅 30t，余 20t 为您拼方案：

  📌 推荐方案（拼仓）：
   • 原 spec：沙钢 HRB400 Φ25 30t × ¥3,820
   • 替代：永钢 HRB400 Φ25（同档异厂）20t × ¥3,795
   • 总 50t 均价 ¥3,810 总价 ¥190,500
   
  ⚖️ 替代说明：永钢与沙钢均为国标 HRB400，
                可互换使用；价格略低 ¥25/吨

回复『确认』锁价；『不要替代』转销售调货沙钢全量
```

#### 完全无货 - 多方案对比
```
张总，您要的螺纹 HRB400 Φ25 沙钢 50t
不巧沙钢这批暂时无货，为您匹配了几个方案：

  方案 A（推荐）永钢同档
   • 永钢 HRB400 Φ25 50t × ¥3,795 = ¥189,750
   • 永钢与沙钢均为国标 HRB400，性能一致
   
  方案 B 同厂升级抗震
   • 沙钢 HRB400E Φ25 50t × ¥3,850 = ¥192,500
   • HRB400E = HRB400 + 抗震性能（贵 ¥30/t）
   • 适合工程项目
   
  方案 C 沙钢长锈轻锈
   • 沙钢 同规格 长锈轻锈 50t × ¥3,720 = ¥186,000
   • 价格优惠 ¥100/t；不影响结构性能但外观有锈迹
   • 适合用作隐蔽工程/非外露件

  方案 D 等沙钢调货
   • 5~7 天到货，价随行情，可锁价
   • 销售小张协助安排

回复『方案 X』选择；或『再看看』暂存
```

#### 战略客户主动推（FULL_STOCK 但替代更优）
```
李总，您要的螺纹 HRB400 Φ25 沙钢 50t 库存充足，
作为战略客户额外为您匹配优选：

  📌 当前方案 沙钢 HRB400 Φ25 50t × ¥3,820 = ¥191,000

  💡 备选 永钢 HRB400 Φ25 50t × ¥3,795 = ¥189,750（省 ¥1,250）
        永钢与沙钢同档可换，下次合作可考虑～

按原方案确认请回『确认』；如换永钢请回『换永钢』
```

#### 议价中作为让步货币
```
张总，价上确实让到边了。换个思路：

  📌 保持原价 ¥3,820 不变
     但产品从标准品升级到优级品（同价升级）
     材质书更好开、客户验收更顺

  📌 或换永钢同档 ¥3,795，省 ¥1,250

您看哪个方便？
```

#### 工程类用途 - 严格模式（不推规格相近/向下）
```
张总，了解到您这批用于桥梁工程，
按图纸要求咱这边严格不推规格相近或向下替代，
只考虑同档异厂或同档异定尺替代：

  方案 A 永钢同档（推荐）
   永钢 HRB400 Φ25 50t × ¥3,795
   ↑ 国标同档，工程通用
   
  方案 B 同档 9 米定尺
   沙钢 HRB400 Φ25 9 米 50t × ¥3,805
   ↑ 比 12 米便宜 ¥15/t；适用切割使用场景

回复『方案 X』或『其他方案』
```

#### 客户拒绝替代时
```
好的张总，已记录您要原 spec 不接受替代。
沙钢 HRB400 Φ25 50t 已为您转销售小张调货，
预计 5-7 天，是否锁价？
```

#### 替代决策合同条款（销售可见的提示）
```
[销售确认替代方案前 - 系统提示]
本订单含替代料：
  原 spec：沙钢 HRB400 Φ25 12 米
  替代为：永钢 HRB400 Φ25 12 米
  替代类型：同档异厂

合同需单独标注此替代关系；
客户已通过 Bot 明确确认；
交付时签收清单留档；
争议时可查 substitute_decision_log。

[继续提交订单] [让客户再确认一次]
```

---

## 7. 安全与合规（v7 增量）

- 议价话术 LLM 输出必经 DLP（禁词：成本/老板/亏本/底价/批价等暴露内部的字眼）。
- 让步预算永远不在客户回复里露面。
- 销售 override 议价决策（如手动放穿底价）→ 强制留原因 + 主管审批 + 审计。
- 议价日志保留 1 年，纠纷追溯。

---

## 8. 可观测性（v7 增量）

- 议价指标：
  - `negotiation_rounds_distribution`
  - `negotiation_outcome{result}`（AGREED/IMPASSE/ESCALATED）
  - `concession_total_amount{customer_type}`
  - `counter_offer_acceptance_rate{type}`
  - `bargaining_llm_safety_blocked_total`（DLP 拦截）
- 销售视角：
  - 销售选 Top-3 方案接受率（评估 Bot 建议质量）
  - 销售 override 频率
  - 议价成单率
- 客户视角：
  - 议价后转化率
  - 议价拉长导致超时未成交率
  - 客户情绪分布

---

## 9. 部署 / 10. 里程碑（v7 增量）

| 里程碑 | 交付物 |
|---|---|
| **M29 Negotiation Session Manager** | 议价状态机 + 会话存储 + 跨天恢复 |
| **M30 Concession + Counter-offer** | 让步预算 + 让步曲线 + 8 维让步生成 |
| **M31 Order-level Profit Optimizer** | 整单视图 + 跨项让步分配 + 整单底线 |
| **M32 Bargaining LLM** | 议价 Prompt + few-shot + 情绪检测 + DLP |
| **M33 议价话术模板** | 8 套话术模板 + 销售脚本 |
| **M34 议价灰度** | 先 Assisted（强销售确认）→ 部分 Auto 升级议价 → 整单议价 |
| **M35 改量能力（v8）** | Quote 生命周期状态机（版本化）+ Amendment Engine + 四档决策 + 改量话术 + 频次防护 + 锁后改量审批工单 |
| **M36 Lead Routing + Sales Score（v10）** | Lead Routing Engine + Sales Performance Score + Sales State Manager + SLA-ACK 转派 + 团队 leaderboard + 抽签温度参数后台 |
| **M37 替代料推荐（v11）** | Substitute KB 基础数据 + Match Engine + Strategy Engine + 客户偏好画像扩展 + 替代话术 + 合同/审计 + 议价让步货币集成（第 9 维） + 后台运营 UI（KB 维护） |

---

## 11. 风险与对策（v7 增量）

| 风险 | 影响 | 对策 |
|---|---|---|
| 让步预算泄漏 | 客户摸清底牌后无限砍 | 预算只在系统内；LLM 输出经 DLP 扫禁词 |
| 议价 LLM 编造让步条件 | 销售背锅 | LLM 不出价/不许愿；所有数字从结构化方案直接渲染 |
| 客户用虚假竞品施压 | 错失或赔本 | 系统记录历次竞品提及；多次提同价未走，可信度↓；销售自行判断 |
| 整单优化算错跨项让步 | 整单赔本 | 整单加权毛利底线校验；算法决策走审计 |
| 求利型陷入无限议价 | 时间成本 | 5 轮上限；末端强制转人工 |
| 求量型客户被骗加量 | 客户信任损失 | 量绑定方案必须诚实标注新均价；不能玩文字游戏 |
| 议价 + 软干预冲突 | 体验混乱 | 议价中暂停软干预降档；议价结束后恢复 |
| 销售在 Bot 外口头给低价 | 系统价格失控 | 销售口头 offer 必须 30 分钟内在 Bot 里登记；超时报警 |
| 跨天议价行情变了 | 报价过期 | 跨天恢复时校验当前底价；变了自动提示销售重报 |
| 客户情绪激动被 Bot 顶撞 | 客情危机 | 情绪检测 ≥ strong 立即转人工 |
| 议价话术机械化 | 客户察觉 AI | LLM + 销售微调 + 自然多样话术库 |
| 自动让步过快被薅 | 毛利失守 | 让步节奏由曲线控制；销售不能"加速" |
| **改量后客户反悔回旧版**（v8） | 操作混乱 | 版本化 + 销售可手动回滚（带原因） |
| **客户反复改量薅羊毛**（v8） | 浪费销售/系统资源 | 频次防护 + 转人工 + ghost_score↑ + 软干预 |
| **锁后改量销售随意批**（v8） | 锁价失去意义 | LOCKED+AMENDING 必走审批工单 + 主管复核 + 30 天复盘 |
| **改量跌穿 MOQ 未提示**（v8） | 客户体验差 | CUSTOMER_CHOICE 强制走选项卡片，不自动按"不过磅"处理 |
| **改量超库存自动拼仓错误**（v8） | 实际发货困难 | 库存校验 + 销售确认拆批方案 |
| **整单改量破整单底线**（v8） | 整单赔本 | Optimizer 严格校验；破线即转 MANUAL |
| **议价中改量预算计算错**（v8） | 让步预算失控 | 改量进议价引擎，预算按新量重算 + 上限保护 |
| **抽签算法导致马太效应**（v10） | 优秀销售爆单/底部销售掉队 | 温度参数 T 可调；新人保底概率 ≥10%；优秀销售每日配额限制 |
| **销售互相抢单或甩单**（v10） | 内部冲突 | 绑定关系受保护；池抽签不可手动改；二级转派须主管授权；甩单率监控 |
| **销售故意拒接好客户**（v10） | 评分扭曲 | decline_rate 入分；异常拒单审计；主管复盘 |
| **优秀销售工单挤压**（v10） | 服务质量下降 | 配额限制；超配额自动溢出；workload 监控 |
| **绑定销售离职后客户体验中断**（v10） | 客户信任受损 | 自动解绑 + 历史交接给池新接单销售 + 客户安抚话术 |
| **VIP 客户被分到非资深销售**（v10） | 客户流失风险 | VIP 池强制 + vip_eligible 主管维护 + VIP 池空时直转主管 |
| **池为空 SLA 严重超时**（v10） | 客户体验崩盘 | 放宽筛选 → 团队群认领 → 值班销售兜底 → 主管介入 |
| **无库存被 Bot 错判为有库存**（v10） | 报价后无货可发 | 库存查询附 timestamp；报价附库存快照；锁价时长 ≤ 库存确定性 |
| **未定价规格被 LLM 编造价格**（v10） | 商业事故 | LLM 严禁出价；未定价路径强制转销售 |
| **替代料偷换被客户发现**（v11） | 投诉/拒收/索赔 | 推荐必须明示替代关系；客户必须明确确认；合同单独条款；交付签收清单 |
| **工程方按图纸拒收替代料**（v11） | 退货/工程纠纷 | 客户档案 accepts_substitute 字段 + 用途场景 LLM 抽取 + 工程类严格模式 |
| **销售把替代当原料卖**（v11） | 合规风险 | Bot 永远显示"替代方案 + 原品对比"；合同条款单独标注；审计追溯 |
| **替代料 KB 规则错误**（v11） | 大面积推错 | KB 变更二人复核 + 灰度发布；行业标准导入校验；历史成交回流验证 |
| **替代推荐砸自己原品价格**（v11） | 老客户感觉被宰 | FULL_STOCK 时默认不主动推；战略客户主动推也仅作"信任建立"非"降价" |
| **客户接受度画像冷启动**（v11） | 新客户体验不准 | 用客户类型 + 用途场景兜底；首次推保守模式 |
| **同档异厂规格细微差异**（v11） | 客户加工设备不适配 | 规则中记录"加工差异提示"；推荐时 caveat 字段透出 |
| **OCR 单供应商风险**（v9） | 通义千问限流/故障时所有视觉能力受影响 | 企业级 SLA 配额；Qwen-VL-Max → Plus 内部降级；解析失败转人工 + 告警；演进项预留 DeepSeek-VL 应急备选 |
| **Qwen-VL OCR 对非标准票据/手写识别下降**（v9） | 付款凭证关联订单错位 | 规则正则二次校验金额/卡号末四位；不唯一时反问客户；财务最终人工确认才落账 |
| **DashScope 计费失控**（v9） | 视觉 token 量大费用飙升 | 文件 hash 缓存（同图不重复识别）；按客户 / 日 配额；图片预先压缩到合理分辨率；非询价/付款凭证场景一律不走 VL |

---

## 12. 后续演进

- **议价 RL（强化学习）**：用历史议价数据训练让步策略（哪个客户哪轮让多少最优）。
- **竞品价格数据库**：销售可登记竞品报价 → 系统判断真假 + 整体行情走向。
- **多客户协同**：同区域客户都在砍同样品种 → 行情分析 → 给销售总监行情提示。
- **议价 A/B**：不同让步曲线在不同客群上跑 A/B，找最优。
- **销售助理小程序**：议价桌面端面板，销售一眼看到所有当前议价 session。
- LLM Function Calling 直接驱动整个议价循环（含 propose_quote/negotiate_round/accept_offer），但**关键决策仍走规则引擎**。

---

## 13. 版本演进对比

| 维度 | v3 | v4 | v5 | v6 | v7 | v8 | v9 | v10 | **v11** |
|---|---|---|---|---|---|---|---|---|---|
| 询价输入 | 文字 | 文字/图/Excel/PDF | 同 | 同 | 同 | 同 | 同 | 同 | 同 |
| 询价字段 | 5 | 7 | 9 | 同 | 同 | 同 | 同 | 同 | 同 |
| NLU 策略 | LLM | LLM | KB+默认+溯源 | 同 | 同 | 同 | 同 | 同 | 同 |
| 报价 | 销售单干 | 同 | 同 | 三档协作 | 同 | 同 | 同 | 同 | 同 |
| 客户画像 | — | — | 偏好 | 完整画像 | 同 | 同 | 同 | 同 | 同 |
| 库存匹配 | — | — | — | 多源组合 | 同 | 同 | 同 | 同 | 同 |
| 报价策略 | — | — | — | 11 策略 | 同 | 同 | 同 | 同 | 同 |
| 行为干预 | — | — | — | 5 层 | 同 | 同 | 同 | 同 | 同 |
| **议价** | — | — | — | — | 多轮状态机 + 让步曲线 | 同 | 同 | 同 | 同 |
| **让步货币** | — | — | — | — | 8 维（价/量/时/运/期/质/赠/组） | 同 | 同 | 同 | 同 |
| **整单优化** | — | — | — | — | 跨项让步分配 + 整单底线 | 同 + 整单改量校验 | 同 | 同 | 同 |
| **议价 LLM** | — | — | — | — | 带预算的议价话术 + 情绪检测 | 同 | 同 | 同 | 同 |
| **议价审计** | — | — | — | — | 全轮 offer + 销售决策落库 | 同 | 同 | 同 | 同 |
| **报价生命周期**（v8 新） | — | — | — | — | — | **DRAFT/ACTIVE/AMENDING/LOCKED/EXPIRED/SUPERSEDED + 版本化** | 同 | 同 | 同 |
| **改量决策**（v8 新） | — | — | — | — | — | **AUTO / ASSISTED / CUSTOMER_CHOICE / MANUAL / REJECT 五档** | 同 | 同 | 同 |
| **跌穿 MOQ**（v8 新） | — | — | — | — | — | **客户选项卡片** | 同 | 同 | 同 |
| **锁后改量**（v8 新） | — | — | — | — | — | **必走审批工单** | 同 | 同 | 同 |
| **改量频次防护**（v8 新） | — | — | — | — | — | **每 Quote / Session / 日多级阈值** | 同 | 同 | 同 |
| **视觉 OCR 供应商**（v9 新） | — | — | — | — | — | — | **统一通义千问 Qwen-VL（移除 PaddleOCR / 阿里 / 腾讯 OCR）** | 同 | 同 |
| **DashScope 集成**（v9 新） | — | — | — | — | — | — | **企业 SLA + 配额 + 子账号 + 计费监控** | 同 | 同 |
| **转人工触发统一**（v10 新） | — | — | — | — | — | — | — | **13 种触发归 Lead Routing Engine** | 同 |
| **销售路由两层**（v10 新） | — | — | — | — | — | — | — | **绑定优先 + 团队池加权抽签** | 同 |
| **销售评分**（v10 新） | — | — | — | — | — | — | — | **7 维综合 + 团队 leaderboard + 周期更新** | 同 |
| **SLA + ACK**（v10 新） | — | — | — | — | — | — | — | **5min ACK / 30min 回客户 / 60min 主管升级** | 同 |
| **抽签温度 T**（v10 新） | — | — | — | — | — | — | — | **可后台调节 精英 vs 公平** | 同 |
| **配额限制**（v10 新） | — | — | — | — | — | — | — | **每销售每日新单配额 + 新人保底 10%** | 同 |
| **替代料推荐**（v11 新） | — | — | — | — | — | — | — | — | **7 类关系 + 3 场景时机 + 3 维客户接受度** |
| **替代料 KB**（v11 新） | — | — | — | — | — | — | — | — | **同档/向上下/相近/外标等价规则集 + 后台维护** |
| **让步货币第 9 维**（v11 新） | — | — | — | — | — | — | — | — | **substitute 作为议价让步货币** |
| **替代料合同**（v11 新） | — | — | — | — | — | — | — | — | **单独条款 + 客户明确确认 + 签收清单审计** |

---

## 14. 附录：议价场景样例（v7 新增）

### 样例 1：求利型客户 3 轮议价
```
[首报价] 螺纹 HRB400 25mm 50t 沙钢武汉 ¥3,820/t 锁 24h
客户："3780 我才下"
[round 1] 让 ¥15 + 锁价延长到 48h → ¥3,805
客户："3795 吧"
[round 2] 让 ¥5 + 材质书优先 → ¥3,800
客户："3790 行不行"
[round 3] 价不动，运费兜底 ¥20 → 等价 ¥3,780（运到货价）
客户："好，定"
[结果] AGREED，让步预算用 60%，毛利保住 ¥110/t
```

### 样例 2：求量型客户量价绑定
```
[首报价] 螺纹 50t ¥3,820
客户："3780 行的话我下"
[round 1] 不直接降价，给量绑定方案：加到 70t 单价 ¥3,795
客户："那 80t 呢"
[round 2] 80t ¥3,790（再让 ¥5）
[结果] AGREED 80t 整单，整单毛利反而比 50t 降 ¥30 更高
```

### 样例 3：整单议价
```
客户：螺纹 50t + 工字钢 30t + 中板 20t
[首报价整单] ¥384,000 毛利率 5.6%
客户："螺纹再降 30 吧"
[round 1 - 跨项分配 方案 B]
  螺纹保 ¥3,820
  中板让 ¥40/t → 整单让 ¥800 + 工字钢锁价延长
客户："那螺纹真不能再让点？"
[round 2 - 方案 D]
  整单一口价 ¥382,300（让 ¥1,700），不再逐项谈
[结果] AGREED 整单
```

### 样例 4：客户用竞品施压 + 系统记录
```
客户："X 家给我报 3750"
系统检测：该客户半年内 4 次提"X 家 3750"但都未走 X 家
Bot 给销售标记：「客户疑似虚价施压，过去 4 次同话术成单于我方」
销售选话术：「咱们家这批沙钢直发，材质书一炉一证。3750 真要走，
            我得汇报老板，您容我半小时回您」
客户半小时后："算了 3795 也行"
[结果] AGREED ¥3,795
```

### 样例 5：求利型客户 5 轮死磕 → 转人工
```
[round 1~4] 让步预算耗尽，客户仍砍
[round 5] 触发转人工：「张总，这价咱们得让小张当面跟您细聊」
[销售接管] 销售判断后给"友情价" -¥10 + 长期合作承诺
[结果] AGREED + override_audit_log
```

### 样例 6：跨天议价行情大跳
```
[Day1 18:00] 首报价 ¥3,820，客户考虑
[Day2 09:00] 客户："我接受 ¥3,820 了"
系统检测：Day2 早盘螺纹大涨 ¥40/t
Bot 给销售提示："客户接受昨日报价，但当前行情已涨 ¥40
                 建议：1) 礼貌按今日 ¥3,860 重报；2) 仍按昨日价
                 您看如何处理？"
销售可选维持承诺（亏 ¥40 守信） 或 重报（要解释）
```

### 样例 7：求利型客户多维要求
```
客户："价让 30 + 送货到厂 + 账期 30 天"
拆分让步：
  价让 30 → 成本 ¥1,500
  送货到厂 → 成本 ¥1,000（运费）
  账期 30 天 → 资金成本 ~¥1,200
合计成本 ¥3,700，远超让步预算 ¥1,500

系统建议方案：
  方案 A：让 ¥15 + 送货 → 总成本 ¥1,750（小超）
  方案 B：让 ¥10 + 账期 15 天 → 总成本 ¥1,100
  方案 C：让 ¥20 不送货不账期 → 总成本 ¥1,000

销售选 B，话术：
  "送货咱们这块儿不行，但账期 15 天没问题，价再让 ¥10。
   您看：¥3,810 自提 + 15 天账期"
```

---

## 15. 待需求方确认

新增 v7 关键问题：

16. **议价是否允许 Auto 档客户直接进议价**？还是任何议价都强制升档到 Assisted？
17. **让步曲线节奏**（求利 0.5/0.25/0.1 / 求量量绑定为主 / 战略 0.7 一次给）是否符合公司业务？
18. **整单议价的整单加权毛利底线**是多少？（影响 Optimizer 决策）
19. **8 种让步货币是否都允许 Bot 用**？账期延长涉及信用是否要走风控审批？
20. **议价话术 LLM 是否需要按销售个人风格定制**？还是公司统一话术池？
21. **议价轮数上限**默认 5 轮，业务方意见？
22. **跨天议价行情变化处理**：自动按今日价 / 按昨日承诺 / 提示销售人工决定？
23. **客户多次提虚假竞品的处理策略**：系统标注后销售如何应对？是否要积分式信用扣减？

v8 新增（改量相关）:

24. **改量频次阈值**：同 Quote 改量上限默认 3 次、单日单客户 10 次，是否符合业务？
25. **锁价后改量是否一律走销售审批**？还是 ±5% 内小幅可放行？
26. **加量超库存的拆批方案**：客户能接受"先发可用 + 后调货"还是必须整批？
27. **减量退价/不退价规则**：减量时单价回到标准档？还是按"原锁价不变"保护毛利？
28. **改量后报价的版本管理需求**：客户能否主动回到上一版本？保留几代版本？
29. **议价中改量是否计入议价轮次**？同一 round 升级还是新 round？
30. **改到 0 的撤单流程**：是否需要客户额外确认？是否影响该客户下次询价待遇？

v10 新增（转人工 / 销售路由相关）：

31. **客户绑定销售关系数据**：从 CRM 同步还是 Bot 自建？冲突时谁说了算？
32. **销售评分 7 个维度的权重**（默认转化率 25%/吨位 20%/DSO 15%/NPS 10%/响应 15%/拒单 10%/负载 5%）是否符合业务？哪些权重要调？
33. **抽签温度参数 T 初始值**：默认 1.0（中等差异化）；偏精英 0.5 / 偏公平 2.0，业务方倾向？
34. **每销售每日新单配额**默认 8 个，是否合适？需按级别（junior/mid/senior）差异化吗？
35. **新人保底概率**默认 ≥10%，是否需要更高？或按入职时间衰减？
36. **VIP 资格名单**谁维护？销售主管 / 销售总监 / 老板？多久复审？
37. **ACK SLA**默认 5 分钟，业务方意见？
38. **回客户 SLA**默认 30 分钟，业务方意见？
39. **拒单率上限**默认未设硬阈值，是否需要超 X% 自动停接新单？
40. **库存数据实时性**：可以做到分钟级实时吗？还是 5 分钟 / 15 分钟快照？影响 "无库存" 判定准确性。
41. **未定价规格列表**：哪些品种属常态、哪些属偶发？是否考虑维护 "可机器报价 SKU 白名单"？
42. **销售互调工单**是否完全禁止？还是允许 + 主管事后审批？
43. **池抽签算法对销售透明吗**？销售能看到自己的中签概率吗？
44. **客户能否主动要求换销售**？流程？
45. **销售离职交接流程**：自动解绑 + 由谁接手？

v11 新增（替代料推荐相关）：

46. **替代料 KB 初始数据**：由谁整理？销售部 + 技术部 + 行业标准？大概多少条规则起步？
47. **客户类型 → 接受度策略**默认：贸易商/加工厂强推、工程方默认不推向下/相近——是否符合公司客户结构？
48. **用途场景识别**：LLM 从对话抽取（工程/配件/出口）是否需要补充行业关键词？
49. **完全有货时是否主动推替代**？默认不推（避免砸自己价）；战略客户除外。业务方意见？
50. **替代料合同条款模板**：法务是否需要起草专门条款？签收清单格式？
51. **同档异厂的细微差异（如永钢和沙钢的螺纹尺寸公差）是否需要在 caveat 中提示**？
52. **替代价更优时的让利分配**：节省的钱完全让给客户、部分让给客户、不让全给公司？
53. **替代料推荐是否需要工程方"现场签字"才能交付**？
54. **替代料 KB 维护后台**谁来维护？销售总监 / 技术部 / 业务部？多久 review 一次？

继续保留 v3~v10 已有的问题（共 54 个累积）。
