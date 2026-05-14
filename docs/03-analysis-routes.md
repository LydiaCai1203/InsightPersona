# 03 - 分析路线

特征提取完成后，按以下 11 条路线进行分析。每条路线明确：输入特征、分析方法、预期产出、揭示的用户行为、对应的业务动作。

```
特征提取完成后
    │
    │   ── 基础分析 ──
    ├─→ 路线一：用户分群            → "我们有哪几种用户"
    ├─→ 路线二：激活留存            → "什么让用户留下来"
    ├─→ 路线三：商业化分层          → "谁会付费、付多少"
    ├─→ 路线四：产品能力 Gap        → "产品哪里好、哪里差"
    ├─→ 路线五：用户演化路径        → "用户怎么变化的"
    ├─→ 路线六：跨维度交叉          → 连接所有路线的深层洞察
    │
    │   ── 垂直深挖 ──
    ├─→ 路线七：垂直行业深挖        → "用户在做什么生意"
    ├─→ 路线八：交付物分析          → "用户拿 AI 输出去干嘛"
    ├─→ 路线九：探索行为分析        → "用户在拓展还是深耕"
    ├─→ 路线十：决策链与采购信号    → "谁能拍板买单"
    └─→ 路线十一：项目阶段覆盖度   → "产品覆盖了多少开发周期"
```

路线一是基础设施（先分群），路线二到十一在每个群落内部分别执行（不同群落的答案不同），路线六贯穿所有维度。

---

## 路线一：用户分群

> 核心问题：我们的用户到底是哪几种人？

### 输入特征

`primary_role`、`primary_industry`、`skill_level`、`task_type_dist`、`ai_dependency_mode`、`task_diversity`、`usage_frequency`、`task_success_rate`、`primary_intent`、`deliverable_dist`

### 方法

1. 对分布型特征做 UMAP 降维
2. 对类别型特征做 one-hot 或 target encoding
3. HDBSCAN 聚类（不需要预设 K 值，自动发现自然群落）
4. 对每个群落统计各维度的均值/众数，形成画像

### 预期产出

5-10 个自然用户群落，每个有清晰画像。示例（假设性，实际由数据决定）：

| 群落 | 画像 | 规模占比 |
|------|------|---------|
| A | 初级前端，学习驱动，高 AI 依赖，任务简单，成功率高 | ~35% |
| B | 中高级后端，生产力驱动，中等依赖，任务复杂，偏重 debug | ~15% |
| C | 非开发者（产品/运营），偶尔用，写脚本/数据处理，行业标签明确 | ~20% |
| D | 全栈高手，低依赖，只在复杂架构任务时来，任务多样性极高 | ~5% |
| E | 高频低质，correction_rate 高，成功率低，任务简单但反复修改 | ~10% |

### 揭示的用户行为

用户不是同质群体。"初学者用来学习"和"高手用来加速"是完全不同的使用模式，对产品的需求、对失败的容忍度、付费意愿都不同。

### 业务动作

- 针对每个群落设计差异化产品策略
- 资源分配依据：**群落规模 x 群落 LTV**，交叉后决定优先服务谁

---

## 路线二：激活与留存分析

> 核心问题：什么决定了用户留下来？

### 输入特征

`first_session_outcome`、`first_week_tasks`、`first_week_success_rate`、`activation_speed_days`、`time_to_first_success`、`lifecycle_stage`、`session_gap_pattern`、`retained_after_failure`

### 方法

1. **留存曲线**：按 cohort（注册周）画 D1/D7/D14/D30 留存
2. **逻辑回归 / 决策树**：以"D30 是否留存"为目标变量，所有首周特征为自变量，找最强预测因子
3. **分群对比**：对比留存用户 vs. 流失用户的首周行为分布差异

### 预期产出

**激活公式**（示例）：

```
如果用户在首周内满足以下条件，D30 留存率 > 60%：
  - first_session_outcome = success
  - first_week_tasks >= 3
  - first_week_success_rate >= 0.5
  - time_to_first_success < 20 min

否则 D30 留存率 < 15%
```

**关键时刻地图**（示例）：

```
Day 0-1:  首次任务成功是最强留存信号（OR=3.2）
Day 2-3:  第二次主动回来使用比首次体验更重要
Day 7:    如果还没形成 session_gap_pattern=regular，后续留存概率骤降
Day 14:   如果 task_type 仍然单一（diversity=1），用户大概率卡在 activated 无法进入 engaged
```

### 揭示的用户行为

- 首次成功体验是不是留存的决定性因素（还是频率或任务多样性更重要）
- 用户在哪个时间节点"做出留下的决定"
- 哪些首周行为是"假激活"（看起来活跃但注定流失）

### 业务动作

- 根据激活公式重新设计 onboarding
- 设计 Day 2-3 的召回触达
- 如果 `task_diversity` 是关键因子，引导用户尝试第二种任务类型

---

## 路线三：商业化分层

> 核心问题：谁会付费、付多少？

### 输入特征

`ai_dependency_mode`、`complexity_dist`、`team_signal_mode`、`codebase_scale_mode`、`usage_frequency`、`scope_dist`、`primary_role`、`primary_industry`、`is_within_capability` 失败率

### 方法

1. **价值评分模型**：

```python
value_score = (
    w1 * usage_frequency_normalized +
    w2 * task_complexity_avg +
    w3 * ai_dependency_depth +        # cannot_do_without 占比
    w4 * team_signal_score +           # solo=0, small_team=1, org=2
    w5 * codebase_scale_score +        # personal=0, startup=1, enterprise=2
    w6 * scope_breadth                 # scope 多样性
)
```

2. **分层**：按 value_score 分 Free / Pro / Enterprise
3. **交叉验证**：看每层用户的 `is_within_capability` 失败率

### 预期产出

| 层级 | 特征 | 占比 | 策略 |
|------|------|------|------|
| Free | 低频、简单任务、solo、学习驱动 | ~60% | 保持免费，漏斗顶部 |
| Pro | 中高频、中等复杂度、生产力驱动、偶有 team 信号 | ~25% | $20-30/月 |
| Enterprise | 高频、高复杂度、明确 team/org 信号、codebase 大 | ~5% | 按 seat 定价 |
| 灰色地带 | 高频但简单 / 低频但极复杂 | ~10% | 定价实验组 |

### 揭示的用户行为

- **AI 依赖深度**和**任务复杂度**哪个更能预测付费？
  - 依赖深度 → 定价围绕"用量"
  - 任务复杂度 → 定价围绕"能力"
- `is_within_capability=false` 的高频场景 = 用户愿意付费解锁的功能清单

### 业务动作

- 设计免费/付费边界：把 Pro 层用户"刚好够不到"的能力放在付费墙后面
- `is_within_capability` 失败率最高的 top 5 场景 → 优先开发

---

## 路线四：产品能力 Gap 分析

> 核心问题：产品哪里做得好、哪里做得烂？

### 输入特征

`task_type`、`task_outcome`、`is_within_capability`、`tech_stack`、`task_complexity`、`correction_rate`

### 方法

构建二维矩阵：

- X 轴：任务类型 x 技术栈（如 "debug x Python"）
- Y 轴：需求量（任务数占比）和成功率（task_outcome=success 比例）

### 预期产出

```
                     需求高                      需求低
              +-----------------------+-----------------------+
  成功率高    |  核心优势区            |  隐藏实力区            |
              |  write_code x Py      |  explain x Rust       |
              |  debug x JS           |  refactor x Go        |
              +-----------------------+-----------------------+
  成功率低    |  关键改进区            |  战略放弃区            |
              |  debug x C++          |  write_code x COBOL   |
              |  architect x Java     |  devops x Niche       |
              +-----------------------+-----------------------+
```

叠加 `is_within_capability` 层：

- 关键改进区 + `is_within_capability=true` + `outcome=failure` → **模型能力问题**，需优化 prompt/模型
- 关键改进区 + `is_within_capability=false` → **产品能力缺失**，需新增功能

### 揭示的用户行为

- 用户**真正用得最多的场景**
- 用户在哪些场景下**反复失败但仍然在尝试**（需求强烈，忍受力高——最该优先修的）
- 用户在哪些场景下**失败一次就不再尝试**（品牌认知已固化）

### 业务动作

- 核心优势区 → 营销卖点
- 关键改进区 → 下季度工程重点
- "失败后仍在尝试" → 最高优先级修复
- "失败后放弃" → 需同时修能力 + 修用户认知

---

## 路线五：用户演化路径

> 核心问题：用户是怎么成长的？

### 输入特征

`skill_trajectory`、`intent_shift`、`complexity_trend`、`dependency_trend`、`task_type_shift`、`lifecycle_stage`

### 方法

1. **Sankey 图 / 状态转移矩阵**：统计用户在不同阶段间的迁移概率
2. **序列模式挖掘**：找最常见的用户成长路径

### 预期产出

3-5 条典型用户旅程：

```
路径 A（健康成长，~25%）：
  learning → productivity → deepening dependency → power_user
  特征：skill 持续上升，任务复杂度逐步提高
  AI 角色：从 learning_tool 迁移到 force_multiplier

路径 B（快速毕业，~15%）：
  learning → skill_growing → weaning → dormant
  特征：用户学会了就走了，AI 完成了教学使命

路径 C（卡住，~30%）：
  learning → stable_skill → stable_dependency → declining
  特征：用户技能没有提升，任务类型和复杂度不变，最终失去兴趣

路径 D（高价值锁定，~10%）：
  productivity → escalating_complexity → deepening → power_user
  特征：一来就是干活的，越用越深
```

### 揭示的用户行为

- **路径 C 是最大的产品问题**：30% 的用户"卡住了"。需要深挖卡在哪里
- **路径 B 不是流失，是毕业**。对这批用户做召回是浪费资源
- **路径 D 是商业化的锚点**：定义了 Pro/Enterprise 应该长什么样

### 业务动作

- 路径 C：检测到用户 2 周内 `complexity_trend=stable` 且 `task_diversity=1` 时，推荐新的使用场景
- 路径 B：在用户走之前引导从 learning 转向 productivity
- 路径 D：反向定义理想客户画像（ICP）

---

## 路线六：跨维度交叉分析

> 核心问题：各维度之间有哪些非显而易见的关联？

单维度分析给出 "what"，交叉分析给出 "why" 和 "so what"。

### 核心交叉矩阵

| 交叉组合 | 回答的问题 |
|---------|-----------|
| `role x ai_dependency` | 开发者和非开发者的依赖模式区别？非开发者 `cannot_do_without` 占比是否远高于开发者？ |
| `skill_level x task_outcome` | 新手失败率高还是高手失败率高？高手也高说明产品能力天花板太低 |
| `industry x task_type` | 金融行业是否集中在 data 类任务？教育行业是否集中在 explain？ |
| `ai_dependency x churn` | `cannot_do_without` 的用户是否留存最好？如果是，策略就是加深依赖 |
| `time_pressure x task_outcome` | 赶工状态下成功率是否显著下降？产品在用户最需要的时候是否掉链子 |
| `team_signal x codebase_scale` | 有 team 信号的用户 codebase 是否更大？直接验证 B2B 定价假设 |
| `project_phase x task_type` | 项目早期是否 architect/write_code 为主，后期是否 debug/optimize 为主？ |
| `lifecycle_stage x intent` | engaged 用户的 intent 分布和 new 用户有什么不同？ |
| `skill_level x ai_dependency x churn` | beginner + cannot_do_without 的用户留存率很高但成功率低 = 脆弱留存 |

### 预期产出举例

```
发现：
  skill_level=beginner 且 ai_dependency=cannot_do_without 的用户群体，
  D30 留存率 72%，远高于平均的 35%。
  但 task_outcome=success 率只有 45%。

解读：
  这批用户"离不开工具但经常失败"。
  留下来不是因为体验好，而是因为没有替代方案。
  这是脆弱留存——竞品体验稍好，他们会立刻迁移。

业务动作：
  1. 短期：针对 beginner + cannot_do_without 优化成功率（防御性投入）
  2. 长期：引导从 cannot_do_without 到 force_multiplier（提升技能的同时加深依赖）
```

---

## 路线七：垂直行业 x 领域深挖

> 核心问题：用户在做什么生意？能否为垂直场景提供专属供给？

这是你最关心的那条线——"发现量化用户 → 提供 K 线数据"的泛化版本。

### 输入特征

`industry`、`domain`、`tech_stack`、`deliverable`、`task_type`、`role`

### 方法

频次统计 + 关联规则挖掘。把 `industry x domain x tech_stack x deliverable` 交叉，找高频组合。

### 预期产出

**垂直场景发现表**（示例）：

| 行业 | 技术领域 | 技术栈 | 典型任务 | 典型交付物 | 用户量 | 产品机会 |
|------|---------|--------|---------|-----------|--------|---------|
| 金融 | 量化交易 | Python + pandas + backtrader | write_code, debug | 策略脚本、回测报告 | ~8% | 提供 K 线数据 API、内置回测模板 |
| 金融 | 风控 | Python + SQL + sklearn | data_analysis | 风控模型、SQL 查询 | ~3% | 提供风控特征工程模板 |
| 教育 | 课件制作 | HTML + CSS + JS | write_code, explain | 交互式网页、教学 demo | ~5% | 提供教学组件库 |
| 电商 | 数据分析 | Python + SQL + pandas | data_analysis | 报表、数据看板 | ~4% | 提供电商数据分析模板 |
| 游戏 | 游戏开发 | Unity + C# / Godot + GDScript | write_code, debug | 游戏脚本 | ~3% | 内置游戏开发 pattern 库 |
| 内容创作 | 自动化 | Python + requests + selenium | automation | 爬虫脚本、批处理工具 | ~6% | 提供常见平台 API 封装 |
| 硬件/IoT | 嵌入式 | C/C++ + Arduino/ESP32 | write_code, debug | 固件、驱动代码 | ~2% | 提供硬件 SDK 文档集成 |

**垂直群体内部分层**（以量化为例）：

```
量化用户群体（假设 8000 人）：
├── 初学者（~40%）：用 AI 学写第一个策略，tech_stack 简单（pandas + yfinance）
│   需要：入门教程、策略模板、模拟交易环境
│
├── 进阶者（~35%）：已有策略框架，用 AI debug 和优化
│   需要：更快的回测、多因子分析模板、性能优化建议
│
└── 专业级（~25%）：自有完整系统，只在复杂算法问题上用 AI
    需要：高阶数学/统计支持、低延迟接口、私有部署
```

### 揭示的用户行为

每一行都是一个潜在的垂直产品方向。8% 的用户在做量化——如果有 10 万用户就是 8000 个量化从业者，提供 K 线数据对他们是刚需。

### 业务动作

- 每个垂直场景评估：规模 x 付费意愿 x 实现成本
- 优先做"规模大 + 付费意愿高 + 实现成本低"的垂直供给
- 垂直场景可以作为差异化竞争壁垒（通用 AI 编程工具都差不多，垂直供给能拉开差距）

---

## 路线八：交付物分析

> 核心问题：用户拿 AI 的输出去做什么？

### 输入特征

`deliverable`、`scope`、`task_type`、`domain`、`role`

### 方法

统计 `deliverable` 分布，并与 `role` 交叉。

### 预期产出

**交付物图谱**：

```
deliverable 分布：
├── 代码片段/函数（~35%）→ AI 是"代码供应商"，用户自己做集成
├── 完整脚本/工具（~20%）→ AI 是"工具制造机"，用户直接运行
├── 解释/文档（~15%）    → AI 是"知识翻译器"
├── 架构方案/设计（~10%）→ AI 是"技术顾问"
├── 数据分析结果（~8%）  → AI 是"分析师"
├── 调试方案（~7%）      → AI 是"排障专家"
└── 其他（~5%）
```

**deliverable x role 交叉**：

| | 代码片段 | 完整脚本 | 解释/文档 | 架构方案 | 数据报表 |
|---|---|---|---|---|---|
| 开发者 | 高 | 中 | 低 | 中 | 低 |
| 数据分析师 | 低 | 高 | 低 | 低 | 高 |
| 产品经理 | 低 | 中 | 高 | 中 | 中 |
| 学生 | 中 | 低 | 高 | 低 | 低 |

### 揭示的用户行为

AI 在用户工作流中扮演什么角色——"代码供应商"和"技术顾问"对产品形态的要求完全不同。

### 业务动作

- "完整脚本"占比大 → 强化"一键运行"能力（在线 sandbox、依赖自动安装）
- "架构方案"占比超预期 → 发展"技术方案评审"功能
- "数据报表" + industry 集中在金融/电商 → 垂直化的另一个切入点

---

## 路线九：探索行为分析

> 核心问题：用户在拓展能力边界还是守在舒适区？

### 输入特征

`comfort_zone`、tech_stack 随时间的变化、`skill_trajectory`、`task_diversity`

### 方法

按用户聚合 `comfort_zone` 的分布，结合时间维度看变化趋势。

### 预期产出

**探索者类型**：

```
├── 深耕型（~50%）：comfort_zone=familiar > 80%
│   一直在擅长领域用 AI，产品价值 = 效率提升工具
│
├── 扩张型（~20%）：comfort_zone=exploring > 40%，tech_stack 持续增加
│   不断尝试新语言/框架，产品价值 = 学习加速器
│
├── 转型型（~10%）：comfort_zone 从 familiar 迁移到 exploring，exploring 集中在同一方向
│   正在从一个技术栈转向另一个（如 Java → Go），产品价值 = 转型教练
│
└── 随机型（~20%）：没有明显模式
    可能多人共用账号或代写/外包
```

**comfort_zone x task_outcome 交叉**：

| | 成功率高 | 成功率低 |
|---|---|---|
| familiar | 正常：熟悉领域做得好 | 异常：产品能力有硬伤 |
| exploring | 优秀：产品学习辅助能力强 | 预期内，但 <30% 则用户会放弃探索 |

### 揭示的用户行为

`exploring + success` 率衡量"工具在多大程度上帮助用户突破能力边界"——这是一个核心产品指标。

### 业务动作

- 扩张型用户 = 最好的 beta 测试群体
- 转型型用户 = 检测到迁移方向后主动推荐目标技术栈的最佳实践
- `exploring + success` 率作为产品 KPI 跟踪

---

## 路线十：决策链与采购信号

> 核心问题：谁能拍板买单？

### 输入特征

`decision_authority`、`team_signal`、`codebase_scale`、`role`、`scope`

### 方法

交叉分析 `decision_authority x team_signal`，识别 B2B 采购信号。

### 预期产出

**B2B 采购信号矩阵**：

```
                    solo        small_team      org/enterprise
  decision_maker    个人付费     小团队决策者     企业采购入口
                    直接转化     可直接推销       需 sales 跟进

  influencer        --          技术 lead        架构师/技术总监
                                能推动采购        需提供 ROI 数据

  executor          学生/自由    团队成员         普通开发者
                    职业者       用个人账号试用    企业版种子用户
```

**关键发现模式**：

```
发现：某用户 team_signal=org, role=developer, decision_authority=executor
      该用户所在 IP/邮箱域名下还有 4 个用户
      5 个用户的 codebase_scale 一致（enterprise）

解读：一个企业团队在用个人账号试用
      5 人 x $20/月 = $100/月 已经在发生
      转为企业版 10 seats x $30/月 = $300/月

业务动作：
  1. 识别"同一组织多个个人用户"的模式
  2. 找到其中 decision_authority=decision_maker 或 influencer 的用户
  3. 定向推送企业版权益
```

### 揭示的用户行为

高价值用户可能没有采购权限。需要找到同一组织中的决策者。

### 业务动作

- 识别企业影子用户群
- 找到决策者定向推送
- 为 influencer 提供 ROI 数据（帮他们说服上级）

---

## 路线十一：项目生命周期覆盖度

> 核心问题：产品覆盖了多少开发周期？在哪些阶段缺席？

### 输入特征

`project_phase`、`task_type`、`scope`、`time_pressure`、`task_outcome`

### 方法

统计 `project_phase` 分布，交叉 `time_pressure` 和 `task_outcome`。

### 预期产出

**项目阶段分布**：

```
├── 启动/原型期（~25%）：architect, write_code, 小 scope
├── 开发期（~40%）：write_code, debug, 中等 scope
├── 测试/修复期（~20%）：debug, test, refactor
├── 上线/运维期（~10%）：devops, optimize, debug
└── 不明确（~5%）
```

**project_phase x time_pressure x task_outcome 三维交叉**：

| 阶段 | 低压 | 高压 |
|------|------|------|
| 启动期 | 探索式使用，多尝试 | 赶工出 MVP，需要快速脚手架 |
| 开发期 | 稳定产出 | deadline 前集中使用，correction_rate 可能上升 |
| 测试期 | 仔细调试 | 上线前最后一刻修 bug，最需要可靠性 |
| 运维期 | 优化性能 | 线上事故排障，对响应速度要求极高 |

### 揭示的用户行为

- 如果"运维期"只占 10% 且成功率低，可能是产品缺乏生产环境调试能力
- 如果"启动期"成功率最高，产品擅长"从零开始"但不擅长"融入已有项目"
- `time_pressure=high` 时段的用户体验是品牌口碑的放大器

### 业务动作

- 覆盖度缺口 = 产品机会
- "启动期"高成功率 → 营销重点（"用 XX 10 分钟启动一个项目"）
- `time_pressure=high` + `outcome=failure` → 最需要优先修复的场景（品牌风险）

---

## 汇总：11 条路线的最终产出物

| # | 路线 | 最终产出物 |
|---|------|-----------|
| 1 | 用户分群 | 5-10 个 Persona 画像档案 |
| 2 | 激活留存 | 激活公式 + 关键时刻地图 |
| 3 | 商业化分层 | Free/Pro/Enterprise 边界定义和定价依据 |
| 4 | 产品能力 Gap | 需求 x 成功率矩阵 + 改进优先级列表 |
| 5 | 用户演化路径 | 3-5 条典型旅程 + 每条路径的干预点 |
| 6 | 跨维度交叉 | 一组反直觉洞察 + 因果关系假设 |
| 7 | 垂直行业深挖 | 垂直场景发现表 + 专属供给策略 |
| 8 | 交付物分析 | AI 角色定位图 + 产品形态建议 |
| 9 | 探索行为 | 探索者类型分布 + beta 测试人群 + 产品 KPI |
| 10 | 决策链 | B2B 采购信号矩阵 + 企业影子用户识别 |
| 11 | 项目阶段覆盖 | 开发周期覆盖度 + 品牌风险场景 |
