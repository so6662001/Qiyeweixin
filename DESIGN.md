# 企业微信机器人 — 设计文档（v5）

> 状态：设计阶段（尚未开发）
> 行业：**钢铁贸易**
> 目标：搭建一个企业微信智能机器人，对接公司已有的 8 项后端能力（查留货订单 / 查欠款 / **询价（4 种输入：文字/图片/Excel/PDF）** / 查装车重量 / 要材质书 / 接收或查结算单 / 接收付款凭证 / 提醒销售发货），覆盖内部员工和外部微信客户，接入 LLM（DeepSeek / 通义千问，含 VL 视觉模型）做自然语言理解、多模态解析与多轮对话。
> **v5 核心**：钢铁贸易自然语言询价的语义抽取（含品类/规格/材质/产地/长度/标准/数量/重量/单重 9 要素），引入**钢铁知识库**和**默认值推断引擎**。

---

## 0. 需求方已确认的关键约束

| # | 问题 | 答复 | 对设计的影响 |
|---|---|---|---|
| 1 | 用户范围 | 内部 + 外部微信用户 | 双通道 |
| 2 | 业务 API | 已具备，不可改造 | ACL 适配 |
| 3 | 绑定账号 | 必须 | 强绑定 + 数据级权限 |
| 4 | 多轮对话 | 跨天、多意图 | Topic + Task |
| 5 | 部署 | 公网云，已备案 | 云原生 |
| 6 | 行业 | 钢铁贸易 | 强权限 + DLP + 审计 |
| 7 | LLM | DeepSeek + 通义千问 | 双路由 |
| 8 | 后端能力 | 8 项 | 工单 + 反向回调 + 文件管道 |
| 9 | 询价输入形态 | 文字/图片/Excel/PDF | 多模态解析 + VL |
| 10 | **询价自然语言复杂度（v5）** | **9 要素混排、地域/品类/客户讲法各异、省略=默认** | **新增钢铁知识库 + 默认值推断引擎 + 字段级溯源；扩展 InquiryItem；反问策略升级** |

---

## 1. 需求与目标

### 1.1 8 项能力的形态分类（同 v4）

略，参考 v4。

### 1.2 询价 4 种形态（同 v4）

略。

### 1.3 询价自然语言 9 要素（v5 新增）

| 要素 | 字段 | 例子 | 是否常被省略 |
|---|---|---|---|
| 品类 | category | 螺纹钢 / 盘螺 / 线材 / 中厚板 / 热轧卷 / 冷轧卷 / H 型钢 / 工字钢 / 槽钢 / 角钢 / 无缝管 / 焊管 / 镀锌管 / 圆钢 / 方钢 | 很少省略 |
| 规格 | spec | Φ25 / 25mm / 14# / DN50 / 108×4.5 / 6.0×1500×C / 200×200×8×12 | 很少省略 |
| 材质（牌号） | grade | HRB400 / Q235B / Q355B / 45# / SS400 / SUS304 | **经常默认**（按品类） |
| 产地（钢厂） | origin | 沙钢 / 永钢 / 中天 / 萍钢 / 武钢 / 宝钢 | **经常默认**（按客户偏好） |
| 长度（定尺） | length | 9m / 12m / 定尺 / 倍尺 / 非定尺 | **经常默认**（12m 国标） |
| 标准 | standard | GB/T 1499.2 / GB/T 3091 / GB/T 8162 / GB | **几乎都省略**（默认国标） |
| 数量 | qty + qty_unit | 100 吨 / 50 根 / 30 件 / 5 卷 | 很少省略 |
| 重量 | weight + unit | 由 qty + unit_weight 换算 | 计算字段 |
| 单重 | unit_weight | 来自理论重量表（如 螺纹钢 Φ25 = 3.85 kg/m） | 计算字段 |

**重要原则**：
- "省略 ≠ 没有"，而是"约定俗成的默认"
- 默认值必须 **显式标注** 出来给客户看（标 `🤖默认`），让客户能修正
- 绝不能"偷偷"用默认值进 ERP

### 1.4 同义/异写问题示例（v5 新增）

**品类同义**：
- 螺纹钢 = 螺纹 = 螺纹筋 = 罗纹 = 螺四（HRB400 口语）= 三级螺纹 = 抗震螺纹
- 中厚板 = 中板 = 钢板 = 板材 = 中厚
- H 型钢 = H 钢 = H 型 = 宽翼缘
- 工字钢 = 工钢 = 工 = 普工
- 槽钢 = 槽 = 普槽
- 角钢 = 角铁
- 无缝管 = 无缝
- 焊管 = 焊接钢管 = 直缝焊管 = 黑管
- 镀锌管 = 白管 = 锌管 = 镀锌
- 热轧 = 热轧卷 = 热卷 = 卷板
- 冷轧 = 冷轧卷 = 冷卷

**规格异写**：
- 螺纹 Φ25 = D25 = 25 = 直径25
- 工字钢 14# = 工14 = 14号 = I14
- 槽钢 16# = 槽16
- H 型钢 200×200×8×12 = HW200 = 200H
- 角钢 ∠50×50×5 = L50×5 = 50角
- 无缝管 Φ108×4.5 = 108×4.5 = 108的 4.5壁厚
- 圆钢 Φ20 = 20圆
- 卷板 6.0×1500×C = 6×1500（默认 C 卷）

**材质（牌号）旧/新/口语**：
- 螺纹钢：HRB400 = 三级 = 三级钢 = 抗震
- HRB335 = 二级（已淘汰，要识别）
- HPB300 = 一级 = 圆钢盘条
- 板材：Q235 = 普碳 = 普通碳钢
- Q355 = 16Mn（老标号）= 低合金
- 优质碳素：45# = 45号钢
- 不锈钢：304 / 316L
- 日标：SS400 ≈ Q235

**钢厂别名**：
- 沙钢 = 江苏沙钢
- 中天 = 中天钢铁
- 永钢 = 江苏永钢
- 萍钢 = 江西萍钢
- 武钢 / 宝钢 / 鞍钢 / 首钢 / 河钢 / 安钢 / 马钢 / 莱钢 / 济钢
- 北方常用：河钢、首钢、鞍钢
- 长三角：沙钢、永钢、中天、马钢
- 华南：广钢、韶钢、湘钢

**单位/数量歧义**：
- "100 吨" → qty=100, unit=吨
- "100 根" → qty=100, unit=根 → 需要乘理论单重换算成吨
- "10 个" 在"中板10个"里 = 10mm 厚度（不是数量！）
- "几车" / "一车货" 模糊，按 30~33 吨/车 估算并反问

---

## 2. 通道选型（同 v4）

略。

---

## 3. 总体架构（v5）

```
                    ┌──────────────────────────────────────┐
   普通微信用户  ──▶│  微信客服 (kf_*)                       │
   企业微信员工  ──▶│  自建应用                              │
                    └────────────────┬─────────────────────┘
                                     │ 回调 (加密)
                                     ▼
   ┌────────────────────────────────────────────────────────────────┐
   │                       Bot Gateway                                │
   │  Callback → Channel Adapter → Identity/Binding → Permission     │
   │           → Fast Path Router                                    │
   │              ↓                  ↓                                │
   │   Session State Manager        Command Handler                   │
   │              ↓                                                   │
   │   LLM Orchestrator (DeepSeek / Qwen 文本 + VL)                   │
   │              ↓                                                   │
   │   Tool Registry (Function Calling)                               │
   │              ↓                                                   │
   │   Business API Adapter (ACL) ──▶ 现有业务 API                    │
   │              ↓                                                   │
   │   Reply Composer + DLP ──▶ WeCom API Client                      │
   │                                                                  │
   │  ─────────────── 询价能力栈（v3~v5 累积）─────────────────────  │
   │                                                                  │
   │  ① Workflow Engine        ② Inbound Webhook                     │
   │  ③ Media Pipeline                                                │
   │                                                                  │
   │  ④ Inquiry Parser  (v4)                                          │
   │      文字 / 图片 / Excel / PDF → InquiryDraft                    │
   │      ┌────────────────────────────────────────┐                 │
   │      │  v5 新增子组件                          │                 │
   │      │                                         │                 │
   │      │  ⑤ Steel Knowledge Base                 │                 │
   │      │      品类/材质/规格/钢厂/标准词典       │                 │
   │      │      规格正则、理论重量公式             │                 │
   │      │                                         │                 │
   │      │  ⑥ Default Resolver                     │                 │
   │      │      客户偏好 + 地区默认 + 品类默认     │                 │
   │      │      销售关联默认                       │                 │
   │      │      输出 source 标签                   │                 │
   │      │                                         │                 │
   │      │  ⑦ Field-Level Tracer                   │                 │
   │      │      每字段：值 + 置信度 + 来源        │                 │
   │      │      explicit/inferred/missing          │                 │
   │      │                                         │                 │
   │      │  ⑧ Reask Strategy                       │                 │
   │      │      只问 critical missing               │                 │
   │      │      默认值显式回显让客户确认            │                 │
   │      └────────────────────────────────────────┘                 │
   └────────────────────────────────────────────────────────────────┘

   ┌──────────┐  ┌──────────┐  ┌──────────────┐  ┌────────────────┐
   │  Redis   │  │ Postgres │  │ Async Worker │  │ Audit / DLP    │
   │ token/ctx│  │ 绑定/会话│  │ LLM/解析/    │  │ ES / SLS / OSS │
   │ KB cache │  │ KB/偏好  │  │ 工单/推送    │  │ 解析/推断日志   │
   │ 限流/锁  │  │ 工单/审计│  │              │  │                │
   └──────────┘  └──────────┘  └──────────────┘  └────────────────┘
```

---

## 4. 关键流程（v5 重点：询价文字解析）

### 4.1 同步查询类（同 v4）
略。

### 4.2 询价（文字分支详细流程，v5 重写）

```
═══ 阶段 0：客户输入 ════════════════════════════════════════════
客户 → "要50吨螺四 25 沙钢的"

═══ 阶段 1：Tokenize + 候选标注 ═══════════════════════════════════
拆词 + 行业实体识别（基于 Steel KB）：
  「要」     - 动词忽略
  「50吨」   - 数量 qty=50, unit=吨
  「螺四」   - 同义词命中 → category=螺纹钢, grade=HRB400
  「25」     - 规格候选（孤数字 → 联合上下文判断为直径）
              category=螺纹钢 时 25 → spec=Φ25mm
  「沙钢」   - 钢厂别名 → origin=江苏沙钢
  「的」     - 助词忽略

═══ 阶段 2：LLM 函数调用（带 few-shot + KB 注入）═══════════════════
喂给 LLM 的 prompt：
  - System: 钢铁询价助手人设 + 9 要素 JSON schema
  - KB Snapshot: 同义词、规格正则、单位规则（裁剪后注入）
  - Few-shot: 5~10 个真实询价 → 标准输出样例
  - History: 当前 Topic 历史
  - User: 当前消息 + 候选标注

LLM 输出（带 source/confidence）：
{
  "items": [{
    "row_no": 1,
    "category":   {"value":"螺纹钢", "source":"explicit",      "confidence":0.99},
    "grade":      {"value":"HRB400", "source":"explicit",      "confidence":0.99,
                   "note":"由『螺四』推得"},
    "spec":       {"value":"Φ25mm",  "source":"explicit",      "confidence":0.93},
    "origin":     {"value":"沙钢",   "source":"explicit",      "confidence":0.97},
    "length":     {"value":null,     "source":"missing"},
    "standard":   {"value":null,     "source":"missing"},
    "qty":        {"value":50,       "source":"explicit",      "confidence":0.99},
    "qty_unit":   {"value":"吨",     "source":"explicit",      "confidence":0.99},
    "unit_weight":{"value":null,     "source":"missing"},
    "dest_city":  {"value":null,     "source":"missing"}
  }]
}

═══ 阶段 3：Default Resolver 补齐 ═════════════════════════════════
按优先级：客户偏好 > 销售-客户绑定默认 > 地区默认 > 品类默认

length 缺失:
  - 查客户偏好：客户 张总 历史 12 次询螺纹，11 次定 12m → 推断 12m
  → length = {value:"12m", source:"inferred_customer_pref", confidence:0.85,
              note:"按您历史习惯默认 12米定尺"}

standard 缺失:
  - 品类默认：螺纹钢 → GB/T 1499.2
  → standard = {value:"GB/T 1499.2", source:"inferred_industry_default",
                confidence:0.95, note:"国标"}

unit_weight 缺失:
  - 计算字段：螺纹 Φ25mm 理论重量 = 3.85 kg/m
  → unit_weight = {value:3.85, unit:"kg/m", source:"calculated",
                   confidence:1.0}

dest_city 缺失:
  - 客户偏好：常发武汉 → 推断武汉？
  - 但目的地是高风险字段，**不自动推断**，标 critical_missing
  → dest_city = {value:null, source:"missing", critical:true}

═══ 阶段 4：合理性校验 + 派生计算 ══════════════════════════════════
- 规格在合理范围（Φ6~Φ50）✓
- qty>0 ✓
- 派生 total_weight = qty * (无需，qty 已是吨) 或 qty * unit_weight / 1000

═══ 阶段 5：回显客户（Markdown）═══════════════════════════════════
"📋 已解析您的询价（置信度 91%）：

| 项 | 值 | 来源 |
|---|---|---|
| 品类 | 螺纹钢 | ✏️您说的 |
| 牌号 | HRB400 | ✏️『螺四』推得 |
| 规格 | Φ25mm | ✏️您说的 |
| 产地 | 沙钢 | ✏️您说的 |
| 数量 | 50 吨 | ✏️您说的 |
| 长度 | 12米 | 🤖按您历史默认 |
| 标准 | GB/T 1499.2 国标 | 🤖行业默认 |
| 单重 | 3.85 kg/m | 🧮理论重量 |
| 目的地 | ❓ 缺失 | 请补充 |

回复『武汉』补充目的地后即可提交；
或回复『长度 9米』修改默认；
或『取消』丢弃。"

═══ 阶段 6：反问/确认 → 提交工单 ══════════════════════════════════
客户回 "武汉"
   ↓
Default Resolver 把 dest_city=武汉 写回
   ↓
所有 critical_missing 已补齐 → submit_inquiry → 工单创建 → 销售报价 → 回推
   ↓
parse_log 落库（input/output/source/客户修正）
偏好库更新：dest_city=武汉 计数 +1（下次默认置信度提高）
```

### 4.3~4.7 其余流程（同 v4）
略。

---

## 5. 模块设计（v5）

### 5.1~5.11 模块（同 v4）
略，沿用。

### 5.12 Inquiry Parser（v5 强化）

总流程：

```
输入 → Modality Branch (文字/图片/Excel/PDF) → 候选标注（KB 实体识别）
     → LLM 抽取 (function calling + few-shot + KB 注入)
     → Default Resolver (按优先级补齐缺失)
     → 合理性校验 + 派生计算
     → Field-Level Tracer 打标 (explicit / inferred / calculated / missing)
     → Reask Strategy (只问 critical missing)
     → 回显模板 → 客户确认 → 提交
     → parse_log 落库 → 偏好库更新
```

文字分支的子步骤：

1. **预处理**：去标点、统一全半角、把"廿/卅"等数字大写转阿拉伯。
2. **实体标注（KB 匹配）**：用 Aho-Corasick 多模匹配 KB 里的同义词，把"螺四 / 中板 / 工14 / 沙钢 / DN50"等高置信实体先打上标签。这是 LLM 之前的"硬规则层"，提高准确率和省 token。
3. **LLM 抽取**：输入 = 原文 + 已标注实体 + 历史 Topic + Few-shot。
4. **Default Resolver 补齐**：见 5.14。
5. **合理性校验**：规格落在合理钢标范围、数量 > 0、单位匹配、互斥字段检查（HRB400 不能配"圆钢盘条"）。
6. **派生计算**：理论单重、总重量。
7. **Critical Missing 判定**：哪些缺失必须反问、哪些可走默认。
8. **回显模板**：表格 + 来源标签 + 操作菜单。

### 5.13 Steel Knowledge Base（v5 新增核心模块）

**5.13.1 内容**

| 子库 | 内容 | 维护方式 |
|---|---|---|
| `category` | 品类标准名 + 同义词 + 子类型 | 配置文件 + 后台 |
| `grade` | 材质牌号 + 旧称 + 口语 + 适用品类 | 配置 |
| `spec_pattern` | 各品类的规格正则与归一化规则 | 配置 |
| `origin` | 钢厂标准名 + 别名 + 所属区域 | 配置 |
| `standard` | 国标/行标/外标编号 + 适用品类 | 配置 |
| `unit_weight_table` | 各品类/规格的理论单重（kg/m 或 kg/张） | 配置 |
| `term_glossary` | 行业术语（定尺/倍尺/开平/卷板/横切…） | 配置 |
| `regional_alias` | 各地区对同一品类的常用叫法 | 配置 |

**5.13.2 示例数据结构**

```json
// category
{
  "id": "rebar",
  "canonical_name": "螺纹钢",
  "aliases": ["螺纹", "螺纹筋", "罗纹", "螺四", "三级螺纹", "抗震螺纹"],
  "default_grade": "HRB400",
  "default_length": "12m",
  "default_standard": "GB/T 1499.2",
  "regions_preferring": ["全国"]
}

// spec_pattern (rebar)
{
  "category_id": "rebar",
  "patterns": [
    {"regex": "^[Φφ∅Dd]?(\\d{1,2})(?:mm)?$", "type": "diameter",
     "normalize": "Φ${1}mm", "valid_range": [6, 50]}
  ]
}

// unit_weight_table (rebar)
{ "category_id":"rebar", "spec":"Φ25mm", "unit_weight": 3.85, "unit":"kg/m" }

// origin
{
  "id": "shagang",
  "canonical_name": "江苏沙钢",
  "aliases": ["沙钢", "Shagang"],
  "region": "长三角"
}
```

**5.13.3 加载与缓存**

- 启动时全量加载到 Redis（按 type 分 key）。
- Aho-Corasick 自动机进程内常驻，热更新时重建。
- 后台 UI 增删改 → 触发缓存重建（运营可维护）。

**5.13.4 喂给 LLM 的 KB 上下文**

为节省 token，不把整个 KB 喂给 LLM，而是**先用 Aho-Corasick 识别命中的实体，只把相关条目摘要进 prompt**。例：

```
[KB-Snippet]
- 「螺四」= 螺纹钢 HRB400 (category/grade)
- 「25」+ category=螺纹钢 → 规格 Φ25mm (spec)
- 「沙钢」= 江苏沙钢 (origin)
```

### 5.14 Default Resolver（v5 新增核心模块）

**5.14.1 优先级链**

```
对每个未明确给出的字段，按顺序找默认值：

1. customer_pref           — 该客户历史 N 次询价里该字段众数
2. customer_sales_pref     — 客户的对应销售常用的默认
3. order_history_pref      — 该客户最近成交订单里的默认
4. regional_default        — 客户所在地区的默认（如华东 12m, 华北也 12m）
5. industry_default        — 行业惯例默认（如螺纹标准=GB/T 1499.2）

任一命中即停止；都未命中 → missing
```

**5.14.2 字段级策略矩阵**

| 字段 | 是否自动推断 | 主要来源 | critical |
|---|---|---|---|
| category | 否 | — | 是 |
| grade | 是（按品类） | industry_default | 否 |
| spec | 否 | — | 是 |
| origin | 是 | customer_pref → regional_default | 否 |
| length | 是 | customer_pref → industry_default | 否 |
| standard | 是 | industry_default | 否 |
| qty | 否 | — | 是 |
| qty_unit | 是（默认吨） | industry_default | 否 |
| unit_weight | 是（计算） | unit_weight_table | 否 |
| dest_city | **否** | — | **是**（高风险，错地址=错运费） |
| delivery_date | 否 | — | 否（可空） |

> `dest_city` 即使客户偏好稳定也不自动填，必须客户每次确认（避免发错仓库的事故）。

**5.14.3 客户偏好画像**

- 每次客户确认询价后更新：`customer_pref[field][value] += 1`。
- 默认值来源 = 该字段最近 N 次（如 N=10）的众数。
- 冷启动客户（历史 < 3 次）跳过 customer_pref，直接走 regional/industry。
- 销售可以"覆盖"客户偏好：在销售端为客户设固定默认。

**5.14.4 推断溯源**

每个被推断的字段必须落 `inquiry_inference_log`：

```
{
  draft_id, item_no, field, inferred_value,
  source: "customer_pref"|"regional_default"|...,
  evidence: {hits: 11, samples_count: 12, ...},
  customer_accepted: true|false,
  customer_corrected_to: "..."  // 客户改了的话
}
```

用于：
- 评估默认值准确率
- 事后追责（如果客户说"我没说这个"，调日志看推断来源）
- 持续优化默认逻辑

### 5.15 Tool Registry（v5 微调）

| 工具 | 入参变化 |
|---|---|
| `parse_inquiry` | 返回扩展的 InquiryDraft（含字段级 source 与 confidence） |
| `submit_inquiry` | 入参的 InquiryItem 含明确的"客户已确认的最终值" |
| 其他工具同 v4 | |

**InquiryItem v5 完整结构**：

```json
{
  "row_no": 1,
  "fields": {
    "category":     {"value":"螺纹钢", "source":"explicit",   "confidence":0.99},
    "grade":        {"value":"HRB400", "source":"explicit",   "confidence":0.99},
    "spec":         {"value":"Φ25mm",  "source":"explicit",   "confidence":0.93,
                     "normalized_from":"25"},
    "origin":       {"value":"江苏沙钢", "source":"explicit", "confidence":0.97,
                     "normalized_from":"沙钢"},
    "length":       {"value":"12m",    "source":"inferred_customer_pref",
                     "confidence":0.85, "note":"按您历史习惯"},
    "standard":     {"value":"GB/T 1499.2", "source":"industry_default",
                     "confidence":0.95},
    "qty":          {"value":50,       "source":"explicit",   "confidence":0.99},
    "qty_unit":     {"value":"吨",     "source":"explicit",   "confidence":0.99},
    "unit_weight":  {"value":3.85,     "unit":"kg/m",
                     "source":"calculated", "confidence":1.0},
    "total_weight": {"value":50,       "unit":"吨",
                     "source":"calculated"},
    "dest_city":    {"value":"武汉",   "source":"user_confirmed",
                     "confidence":1.0},
    "delivery_date":{"value":null,     "source":"missing", "optional":true},
    "remark":       {"value":null,     "source":"missing", "optional":true}
  },
  "raw_excerpt": "要50吨螺四 25 沙钢的",
  "overall_confidence": 0.93,
  "critical_missing": [],
  "user_confirmed_at": "2026-05-28T16:00:00Z"
}
```

### 5.16 Reask Strategy（反问策略，v5 新增）

**只在 critical 字段缺失时反问**，其余字段用默认 + 回显让客户被动确认：

| 缺失字段 | 反问话术 |
|---|---|
| category | "您要询哪种钢材？比如螺纹、板材、管材..." |
| spec | "规格是多少？如 Φ25 / 14# / 108×4.5" |
| qty | "数量是多少？比如 50 吨 / 100 根" |
| dest_city | "送到哪个城市/仓库？" |

**最多 2 轮反问**，再不齐转人工。避免对话陷入"机器人催问"。

**批量场景**（Excel 50 行）：把所有 critical missing 集中表达，例如"以下 3 行规格不明确，请逐项补充：第 3 / 7 / 12 行"。

---

## 6. 钢铁贸易话术与体验（v5 强化）

### 6.1 回显模板
统一来源标签：
- ✏️ 您说的（explicit）
- 🤖 默认推断（inferred）+ 简短说明（按您历史 / 行业默认 / 销售为您设的偏好）
- 🧮 自动计算（calculated）
- ✅ 您已确认（user_confirmed）
- ❓ 待补充（missing critical）

### 6.2 客户操作动词
- 「确认」 / 「全部确认」 — 接受当前所有值
- 「确认默认」 — 一键接受所有 🤖 推断
- 「第3行 长度 9米」 — 字段级修正
- 「都按沙钢」 — 批量修正
- 「取消」 / 「重新发」 / 「转销售」

### 6.3 销售视角
工单卡片里展示：
- 客户原话 + 解析结果差异（如果改过）
- 每个字段的来源标签
- 客户偏好画像快照（可帮销售判断异常）
- 一键"按往常报价"/"覆盖客户默认"

---

## 7. 安全与合规

同 v4。**v5 新增**：
- `customer_preference` 表是商业敏感数据，加密 + 审计；销售只看自己 managed_customers 的偏好。
- 知识库变更（运营改字典）→ 审计 + 通知开发评审。
- 解析推断日志保留 1 年，用于纠纷追溯。

---

## 8. 可观测性

同 v4。**v5 新增**：
- `inquiry_field_source{field,source}` — 各字段各来源占比
- `inquiry_inference_accept_rate{field}` — 推断接受率
- `inquiry_correction_total{field}` — 客户修正次数
- `kb_hit_rate{type}` — KB 命中率
- 周报：解析准确率 / 推断接受率 / Top 修正字段 → 反哺 KB 维护

---

## 9. 部署拓扑

同 v4。**v5 新增**：
- 知识库后台管理 UI（运营维护词典 / 默认规则 / 钢厂列表）。
- 内置 KB 配置文件 + 数据库表双写，配置文件作为兜底。

---

## 10. 里程碑（v5 调整）

| 里程碑 | 交付物 |
|---|---|
| M1~M7 | 同 v4 |
| **M8 询价文字（v5 重点）** | KB 基础版（品类/材质/钢厂/规格正则/单重表） + LLM 抽取 + 默认推断 + 回显确认 + parse_log |
| M9 询价图片 | VL 模型；图片询价表识别 |
| M10 询价 Excel | openpyxl + LLM 列头映射；批量 SKU |
| M11 询价 PDF | 文本 + 扫描双分支 |
| **M11.5 客户偏好画像** | customer_preference 落库；推断准确率监控 |
| M12 材质书文件下发 | |
| M13 结算单 | |
| M14 付款凭证 | |
| M15 提醒发货 | |
| M16 微信客服通道 | |
| M17 DLP + 数据级权限完善 | |
| M18 可观测 + 风控 + 对账 | |
| **M19 KB 运营后台** | 词典/默认规则可视化维护 |
| M20 灰度上线 | 内部销售 → 部分外部客户 → 全量 |

---

## 11. 风险与对策（v5 更新）

| 风险 | 影响 | 对策 |
|---|---|---|
| **默认值推断错误，客户没注意就提交了** | 错单 | 所有 🤖 推断字段在回显里高亮；critical 字段不自动推断；客户每次都看到来源标签 |
| **客户偏好画像冷启动** | 新客户体验差 | 用区域 + 行业默认兜底；前 5 次询价不写偏好库 |
| **KB 词典维护滞后** | 新词识别不到 | 解析失败日志聚类 → 运营定期补词；客户修正回流 |
| **同义词太多导致 LLM 误判** | 抽取错位 | KB 实体优先匹配 + 规则强约束；LLM 只在剩余字段上发挥 |
| **规格歧义（"200" 是 H 钢规格还是数量）** | 错单 | 上下文 + 单位线索；不确定时反问 |
| **旧标号客户（HRB335）** | 识别不到 | KB 含历史/淘汰牌号，标 deprecated 但仍可解析 |
| **同一份询价跨多行 NLU 错位** | 串行 | 每行独立解析；交叉信息（地址/日期）允许全局共享 |
| 多模态解析准确率不够 | 错单 | 强制回显 + 人工确认；置信度低转人工 |
| Excel 表头千奇百怪 | 列映射错 | LLM 列头映射；客户专属模板（演进） |
| VL 模型费用 | 成本 | 缓存 + 客户配额 |
| LLM/VL 编造数字 | 商业事故 | 严禁出数字；后置 DLP |
| 业务系统不能开发 Webhook | 异步流程走不通 | 过渡轮询 |
| 微信客服 48h 窗口 | 推送失败 | 引导用户先发起 / SMS 退化 |
| 客户机密外泄 LLM | 合规 | prompt 脱敏；如需要切私有化 |
| 跨用户转发暴露敏感信息 | 合规 | DLP + 数据级权限 |

---

## 12. 后续演进

- **客户专属模板**：识别 Top 客户的固定 Excel 模板，跳过列头映射。
- **LoRA 微调**：用 parse_log 真实语料微调 Qwen 7B/14B，做特定领域专用模型。
- **报价 PDF 出件 → 客户回执 OCR 闭环**。
- **主动智能**：账期/留货/价格变动提醒。
- **销售 Copilot**：基于客户偏好画像 + 历史成交，推荐报价策略。
- **多企业租户 KB 隔离**：每家公司自己的词典/默认偏好。

---

## 13. 版本演进对比

| 维度 | v1 | v2 | v3 | v4 | **v5** |
|---|---|---|---|---|---|
| 用户范围 | 通用 | 内外双 | 同 | 同 | 同 |
| 意图识别 | 关键字 | LLM 双供应商 | 同 | + VL | 同 |
| 多轮 | Redis | Topic/Task | 同 | 同 | 同 |
| 绑定 | 草案 | 三方式 | 同 | 同 | 同 |
| 业务 API | 假设 | ACL | + Webhook | 同 | 同 |
| 行业能力 | 通用 | 假想 | 8 项实际 | 同 + 询价多模态 | 同 |
| 询价输入 | 文字 | 文字 | 文字 | 文字/图/Excel/PDF | 同 |
| 询价字段 | 5 | 5 | 5 | 7 | **9（+ 产地/长度/标准/单重）** |
| **NLU 策略** | 关键字 | LLM | LLM | LLM | **KB + AC自动机 + LLM + 默认推断** |
| **默认值** | — | — | — | — | **客户/销售/地区/行业 4 级默认 + 字段级溯源** |
| **同义词处理** | — | — | — | — | **专门 KB 子库 + Aho-Corasick** |
| **规格归一** | — | — | — | 基础 | **正则模板 + 品类分组** |
| **理论单重** | — | — | — | — | **内置公式表 + 计算字段** |
| **偏好画像** | — | — | — | — | **customer_preference + 销售可覆盖** |
| **审计** | 基础 | 增强 | 同 | + 解析日志 | **+ 推断日志（含证据链）** |

---

## 14. 附录：钢铁贸易自然语言解析样例（v5）

| 客户原话 | 解析结果（关键字段） |
|---|---|
| "要50吨螺四 25 沙钢的" | 品类=螺纹钢，牌号=HRB400(✏️『螺四』)，规格=Φ25mm，产地=沙钢，数量=50t，长度=12m(🤖)，标准=GB/T 1499.2(🤖)，目的地=❓ |
| "中板10个100吨" | 品类=中厚板，规格=10mm(✏️『10个』=厚度)，数量=100t，牌号=Q235B(🤖)，定尺=开平定尺(🤖)，目的地=❓ |
| "工14三十吨" | 品类=工字钢，规格=14#(✏️『工14』)，数量=30t，牌号=Q235(🤖)，标准=GB(🤖)，长度=12m(🤖) |
| "108的无缝 4.5壁厚 100根" | 品类=无缝管，规格=Φ108×4.5，数量=100根→需换算吨(单重 11.49 kg/m × 长度)，牌号=20#(🤖)，长度=❓ |
| "弄点H钢200的 5吨" | 品类=H型钢，规格=HW200×200×8×12(🤖 200默认 HW 系列)，数量=5t，牌号=Q235B(🤖)，⚠️ 规格推断置信度 0.6，建议反问 |
| "二级18 50吨 GB" | 品类=螺纹钢(🤖 二级18 暗示)，牌号=HRB335(✏️『二级』, deprecated 提示)，规格=Φ18mm，标准=GB/T 1499.2，数量=50t |
| "螺纹 18 22 25 各50吨 沙钢12米送武汉" | **3 条 item**：螺纹钢/HRB400/Φ18/50t/沙钢/12m/武汉；Φ22/50t/...；Φ25/50t/... |
| "要点6.0热卷沙钢的 50吨" | 品类=热轧卷，规格=6.0mm 厚(✏️『6.0』)，宽度=1500mm(🤖) / ❓，产地=沙钢，数量=50t，目的地=❓ |
| "DN50 镀锌 100米" | 品类=镀锌管，规格=DN50(✏️)，长度=100米(数量字段)，需换算根数；标准=GB/T 3091(🤖)，材质=Q195/Q235(🤖) |
| "304板2.0*1500*L 5吨" | 品类=不锈钢板，材质=304(✏️)，规格=2.0×1500，L=定尺(✏️)，数量=5t |

---

## 15. 待需求方确认

1. **业务系统能否配合开发 Inbound Webhook**（quote.ready / settlement.created / payment.* / shipment.*）？
2. **业务 API 是否支持以下操作**？
   - 按客户+时间查留货订单
   - 按客户查欠款汇总/明细
   - **创建询价单（支持 v5 的 9 要素完整结构 + 批量）**
   - 按订单/车牌查装车重量
   - 按炉号/订单查材质书 PDF
   - 按客户+月份查结算单 PDF
   - 提交付款凭证（写）
   - 提交发货催办（写）
3. 询价 ERP 报价流：销售逐 item 报价还是整单？
4. 报价回推是否需要同时附 PDF 报价单？
5. **询价 9 要素中，业务 API 接收时哪些必填？哪些可空？**
6. **客户偏好画像是 Bot 侧维护还是同步 CRM**？冲突时以谁为准？
7. **运营是否需要 KB 维护后台**？谁来维护词典（销售运营 / 技术）？
8. 付款凭证是否需要 OCR？OCR 服务选阿里 / 腾讯 / 自建？
9. 多 sheet Excel 询价处理：默认询第一个 sheet？反问？
10. 询价文件保留 1 年合规吗？
11. VL 模型预算？
12. 销售在 Bot 里 /reply 还是 ERP 里操作？
13. 询价/催发货/付款核对 SLA 各是多少？
14. 客户绑定方式：销售生成 token / 手机号短信 / 都要？
15. 是否需要群聊场景？
