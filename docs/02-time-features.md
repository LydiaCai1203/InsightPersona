# 02 - 时间维度特征设计

时间维度特征全部基于已有的 Turn/Task/User 层标签和时间戳，通过统计聚合计算得出。**零 LLM 成本**。

核心价值：静态特征告诉你"用户是谁"，时间维度告诉你"用户正在变成谁"。

---

## 一、节奏特征（When & How Often）

### 1.1 对话层节奏

在单个 task 内部，用户与 AI 交互的时间模式。

| 字段 | 类型 | 计算方式 | 说明 |
|------|------|---------|------|
| `response_gap_seconds` | float | 用户收到 AI 回复后到发下一条的时间差 | 短（<30s）= 快速迭代；长（>5min）= 用户在消化/实际操作后回来 |
| `is_rapid_fire` | bool | 用户连续多条间隔 <10s | 用户在补充上下文，第一条没说清楚（交互设计优化信号） |

### 1.2 任务层节奏

| 字段 | 类型 | 计算方式 | 说明 |
|------|------|---------|------|
| `task_duration_minutes` | float | task 首条到末条的时间差 | 短（<2min）= 快问快答；长（>30min）= 深度协作 |
| `think_time_ratio` | float | 用户"思考时间"占总 task 时间的比例 | 高 = 用户在认真使用 AI 输出；低 = 连续追问没得到满意答案 |

### 1.3 用户层节奏

| 字段 | 类型 | 计算方式 | 说明 |
|------|------|---------|------|
| `active_hours` | list[int] | 用户所有 task 的小时分布 | 活跃时段 |
| `work_pattern` | enum | 基于 active_hours 聚类 | `nine_to_five` / `night_owl` / `fragmented` / `always_on` |
| `weekday_ratio` | float | 工作日 task 数 / 总 task 数 | 高 = 工作场景为主 |
| `peak_day` | string | 每周哪天 task 数最多 | 最活跃的星期几 |

**work_pattern 与业务策略映射：**

| 模式 | 推断 | 产品动作 |
|------|------|---------|
| `nine_to_five` + `weekday_ratio` 高 | 工作场景为主 | 商业化目标用户 |
| `night_owl` + 周末活跃 | 个人项目/学习 | 社区运营目标 |
| `fragmented`（全天零散分布） | 自由职业或随时被打断的角色 | 需要良好的会话恢复能力 |
| `always_on` | 重度依赖 | VIP 用户，流失预警优先级最高 |

---

## 二、演化特征（How They Change）

### 2.1 技能演化

| 字段 | 类型 | 计算方式 | 说明 |
|------|------|---------|------|
| `skill_trajectory` | enum | 按月统计 `user_skill_signal` 的中位数变化趋势 | `growing` / `stable` / `unclear` |
| `skill_velocity` | float | skill_signal 数值随时间的线性回归斜率 | 技能提升速度 |
| `stack_expansion_rate` | float | 每月新增的 tech_stack 数量 | 技术栈扩展速度 |

### 2.2 需求演化

| 字段 | 类型 | 计算方式 | 说明 |
|------|------|---------|------|
| `task_type_shift` | dict | 前半段 vs 后半段的 task_type 分布变化 | 用户任务类型迁移 |
| `complexity_trend` | enum | task_complexity 随时间的趋势 | `escalating` / `stable` / `simplifying` |
| `intent_shift` | dict | 前半段 vs 后半段的 intent 分布变化 | 意图迁移（如 learning → productivity） |

### 2.3 依赖演化

| 字段 | 类型 | 计算方式 | 说明 |
|------|------|---------|------|
| `dependency_trend` | enum | ai_dependency 随时间的趋势 | `deepening` / `stable` / `weaning` |
| `ai_autonomy_shift` | enum | 用户自主性变化方向 | `more_autonomous` / `same` / `more_dependent` |

**技能 × 依赖交叉解读：**

| 技能趋势 | 依赖趋势 | 解读 |
|---------|---------|------|
| growing | weaning | 健康毕业：用户学会了，依赖自然下降 |
| stable | weaning | 危险信号：用户没变，但在远离工具 |
| growing | deepening | 最佳状态：用户越强，用得越深 |
| stable | deepening | 可能过度依赖：用户没成长但离不开 |

---

## 三、生命周期阶段特征（Where They Are）

| 字段 | 类型 | 计算方式 | 说明 |
|------|------|---------|------|
| `lifecycle_stage` | enum | 综合规则判断 | `new` / `activated` / `engaged` / `power_user` / `declining` / `dormant` / `churned` |
| `days_since_first_use` | int | 当前日期 - 首次使用日期 | 用户年龄 |
| `days_since_last_use` | int | 当前日期 - 最后使用日期 | 沉默天数 |
| `activation_speed_days` | int | 从首次使用到"激活"的天数 | 激活速度 |
| `time_to_first_success_minutes` | float | 首次 task_outcome=success 之前的总时间 | 首次成功时间 |

### lifecycle_stage 判定规则

```
new:        注册 < 7天，使用 < 3次
activated:  首次成功任务已完成，但尚未形成习惯
engaged:    周均使用 > 2次，持续 > 2周
power_user: 周均使用 > 5次，持续 > 4周，task_diversity > 3
declining:  使用频率较前 2 周下降 > 50%
dormant:    > 14天 未使用
churned:    > 30天 未使用
```

---

## 四、会话间隔模式（Usage Rhythm）

| 字段 | 类型 | 计算方式 | 说明 |
|------|------|---------|------|
| `session_gap_pattern` | enum | 基于 session 间隔的变异系数分类 | `regular` / `bursty` / `sporadic` |
| `avg_days_between_sessions` | float | session 间隔天数的均值 | 平均回访间隔 |
| `longest_gap_days` | int | 最长沉默天数 | 最长中断 |
| `comeback_after_gap` | bool | 是否在长间隔（>7天）后回来 | 长期留存信号 |

**session_gap_pattern 判定逻辑：**

| 模式 | 特征 | 运营策略 |
|------|------|---------|
| `regular` | 间隔稳定（每天或每隔一天） | 已融入工作流，留存稳定 |
| `bursty` | 集中在某几天高频使用，然后消失一段时间 | 项目驱动型：有任务时才来。不要误判为流失 |
| `sporadic` | 间隔不规律，偶尔出现 | 未形成习惯，随时可能流失 |

---

## 五、关键时刻特征（Decisive Moments）

| 字段 | 类型 | 计算方式 | 说明 |
|------|------|---------|------|
| `first_session_outcome` | enum | 首次 task 的 task_outcome | 首次体验结果 |
| `first_week_tasks` | int | 首 7 天的 task 数 | 首周活跃度 |
| `first_week_success_rate` | float | 首 7 天 task_outcome=success 的比例 | 首周成功率 |
| `first_negative_experience_day` | int | 首次 task_outcome=failure 出现在第几天 | 首次负面体验时间点 |
| `retained_after_failure` | bool | 首次 failure 后是否继续使用 | 失败容忍度 |

---

## 六、计算依赖

所有时间维度特征的计算依赖：

| 依赖 | 来源 |
|------|------|
| 每条消息的 `timestamp` | 原始数据 |
| `task_id` | 原始数据 |
| `user_id` | 原始数据 |
| `task_outcome` | Task 层 LLM 标注 |
| `user_skill_signal` | Task 层 LLM 标注 |
| `ai_dependency` | Task 层 LLM 标注 |
| `intent` | Task 层 LLM 标注 |
| `task_complexity` | Task 层 LLM 标注 |
| `task_type` | Task 层 LLM 标注 |

**执行顺序**：必须在 Task 层 LLM 标注完成后，才能计算演化类特征（skill_trajectory、dependency_trend 等）。节奏类和间隔类特征只依赖时间戳，可以与 LLM 标注并行进行。
