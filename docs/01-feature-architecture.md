# 01 - 三层特征架构

数据按 **Turn（对话轮次）→ Task（任务/会话）→ User（用户）** 三层组织，每层的提取方式不同：

- **Turn 层**：规则/Regex 提取，零 LLM 成本，毫秒级完成
- **Task 层**：LLM 标注，输入为完整对话上下文，每个 task 调用一次
- **User 层**：SQL/Pandas 统计聚合，零 LLM 成本

---

## 一、Turn 层特征（规则提取）

Turn 层特征从单条消息中提取，不需要理解语义，纯粹基于文本结构特征。

### 1.1 字段定义

| 字段 | 类型 | 提取方式 | 说明 |
|------|------|---------|------|
| `turn_id` | string | 原始数据 | 唯一标识 |
| `task_id` | string | 原始数据 | 所属任务/会话 |
| `user_id` | string | 原始数据 | 所属用户 |
| `timestamp` | datetime | 原始数据 | 消息时间戳 |
| `role` | enum | 原始数据 | `user` / `assistant` |
| `msg_length` | int | `len(text)` | 消息字符数 |
| `has_code_block` | bool | 正则匹配 ``` 包裹块 | 消息是否包含代码块 |
| `code_block_count` | int | 正则计数 | 代码块数量 |
| `code_language` | list[str] | 正则提取 ```lang | 代码块标注的语言 |
| `has_error_trace` | bool | 正则匹配 Traceback / Error / Exception 模式 | 是否包含报错信息 |
| `error_type` | string | 正则提取 | 报错类型（如 TypeError, SyntaxError） |
| `has_url` | bool | 正则匹配 URL 模式 | 是否包含链接 |
| `has_file_path` | bool | 正则匹配文件路径模式 | 是否引用了文件路径 |
| `code_to_text_ratio` | float | 代码块字符数 / 总字符数 | 代码占比 |
| `turn_position` | int | 在 task 内的顺序位 | 第几轮对话 |
| `turn_position_ratio` | float | 当前位置 / 总轮数 | 归一化位置（0=开头, 1=结尾） |
| `is_correction` | bool | 正则匹配否定/修改模式 | 用户是否在纠正 AI（"不是这样"、"我要的是"、"重新"） |
| `is_followup` | bool | 正则匹配追问模式 | 用户是否在追问（"继续"、"还有呢"、"然后呢"） |

### 1.2 提取示例

```python
import re

def extract_turn_features(text: str, role: str) -> dict:
    code_blocks = re.findall(r'```(\w*)\n(.*?)```', text, re.DOTALL)
    code_chars = sum(len(b[1]) for b in code_blocks)
    
    return {
        "msg_length": len(text),
        "has_code_block": len(code_blocks) > 0,
        "code_block_count": len(code_blocks),
        "code_language": [b[0] for b in code_blocks if b[0]],
        "has_error_trace": bool(re.search(
            r'(Traceback|Error|Exception|FAILED|panic|fatal)', text, re.I
        )),
        "has_url": bool(re.search(r'https?://\S+', text)),
        "has_file_path": bool(re.search(
            r'[/\\][\w\-\.]+[/\\][\w\-\.]+', text
        )),
        "code_to_text_ratio": code_chars / max(len(text), 1),
        "is_correction": bool(re.search(
            r'(不是|不对|错了|重新|不要|instead|wrong|no,|actually)', text, re.I
        )) if role == "user" else False,
        "is_followup": bool(re.search(
            r'(继续|还有|然后呢|接着|go on|continue|more|what else)', text, re.I
        )) if role == "user" else False,
    }
```

### 1.3 Turn → Task 聚合（预处理）

Turn 层特征在进入 Task 层分析前，需要按 `task_id` 聚合：

```python
task_turn_agg = {
    "total_turns": count,
    "user_turns": count(role=user),
    "assistant_turns": count(role=assistant),
    "avg_user_msg_length": mean(msg_length where role=user),
    "avg_assistant_msg_length": mean(msg_length where role=assistant),
    "total_code_blocks": sum(code_block_count),
    "has_error_trace": any(has_error_trace),
    "correction_count": sum(is_correction),
    "correction_rate": sum(is_correction) / user_turns,
    "languages_mentioned": union(code_language),
    "task_duration_seconds": max(timestamp) - min(timestamp),
}
```

---

## 二、Task 层特征（LLM 标注）

Task 层特征需要理解完整对话上下文才能判断，由 LLM 一次性读取整个 task 的对话后输出。

### 2.1 为什么在 Task 层而非 Turn 层标注

- **成本**：10 万个 task 调用 1 次 vs. 40 万个 turn 各调用 1 次，节省 3-5 倍
- **准确性**：很多标签（如 task_outcome、ai_dependency）需要看完整对话才能判断
- **一致性**：同一个 task 内的所有 turn 共享同一组标签，避免冲突

### 2.2 字段定义：Batch 1（客观事实字段）

这些字段对话中有明确证据，LLM 标注准确率预期 > 85%。

| 字段 | 类型 | 可选值 | 说明 |
|------|------|--------|------|
| `task_type` | enum | `write_code`, `debug`, `explain`, `refactor`, `review`, `architect`, `data_analysis`, `devops`, `test`, `learn`, `translate`, `other` | 任务类型 |
| `tech_stack` | list[str] | 开放标签 | 涉及的技术栈（语言、框架、工具） |
| `domain` | enum | `web_frontend`, `web_backend`, `mobile`, `data_science`, `devops`, `embedded`, `game`, `cli_tool`, `automation`, `database`, `ai_ml`, `other` | 技术领域 |
| `task_complexity` | enum | `trivial`, `simple`, `moderate`, `complex`, `expert` | 任务复杂度 |
| `role` | enum | `developer`, `data_analyst`, `student`, `pm`, `designer`, `ops`, `content_creator`, `researcher`, `other`, `unclear` | 推断的用户角色 |
| `industry` | enum | `finance`, `ecommerce`, `education`, `healthcare`, `gaming`, `media`, `enterprise_saas`, `iot_hardware`, `crypto`, `government`, `other`, `unclear` | 推断的行业 |
| `task_outcome` | enum | `success`, `partial`, `failure`, `abandoned`, `unclear` | 任务完成情况 |
| `is_within_capability` | bool | `true`, `false` | AI 是否具备完成该任务的能力（区分"AI 能力不足"和"用户没表达清楚"） |
| `deliverable` | enum | `code_snippet`, `complete_script`, `explanation`, `architecture_design`, `data_report`, `debug_solution`, `config_file`, `documentation`, `other` | 用户期望的交付物类型 |

### 2.3 字段定义：Batch 2（推断性字段）

这些字段需要 LLM 做推断，允许更多 `unclear`，准确率预期 60-80%。

| 字段 | 类型 | 可选值 | 说明 |
|------|------|--------|------|
| `user_skill_signal` | enum | `beginner`, `intermediate`, `advanced`, `expert`, `unclear` | 从对话推断的用户技能水平 |
| `intent` | enum | `learning`, `productivity`, `exploration`, `emergency_fix`, `routine_maintenance`, `unclear` | 用户使用 AI 的意图 |
| `scope` | enum | `snippet`, `function`, `module`, `project`, `system`, `unclear` | 任务涉及的代码规模 |
| `ai_dependency` | enum | `cannot_do_without`, `significant_help`, `minor_help`, `validation_only`, `learning_tool`, `unclear` | 用户对 AI 的依赖程度 |
| `project_phase` | enum | `prototyping`, `active_development`, `testing`, `maintenance`, `firefighting`, `unclear` | 项目所处阶段 |
| `comfort_zone` | enum | `familiar`, `exploring`, `unclear` | 用户是否在自己擅长的领域 |
| `time_pressure` | enum | `high`, `normal`, `low`, `unclear` | 时间压力信号 |
| `team_signal` | enum | `solo`, `small_team`, `org`, `unclear` | 团队规模信号 |
| `codebase_scale` | enum | `personal`, `startup`, `enterprise`, `unclear` | 代码库规模信号 |
| `decision_authority` | enum | `decision_maker`, `influencer`, `executor`, `unclear` | 用户在采购/决策中的角色 |

### 2.4 LLM Prompt 设计要点

```
你是一个对话分析专家。以下是用户与 AI 编程助手之间的完整对话。
请分析对话内容，按 JSON 格式输出以下标签。

规则：
1. 只基于对话中的证据做判断，不要过度推断
2. 如果信息不足以判断某个字段，填 "unclear"
3. tech_stack 使用标准名称（如 "Python" 而非 "py"）
4. task_outcome 判断标准：用户最后是否得到了可用的结果

对话内容：
{conversation}

请输出 JSON：
```

### 2.5 标注成本估算

| 参数 | 估算值 |
|------|--------|
| Task 数量 | ~100,000 |
| 平均对话长度 | ~2,000 tokens |
| Prompt + output | ~3,000 tokens/task |
| 模型选择 | GPT-4o-mini / Claude Haiku |
| 单价 | ~$0.15-0.20 / 1M tokens |
| 总成本（Batch 1） | ~$8-12 |
| 总成本（Batch 2） | ~$7-10 |
| **总计** | **~$15-22** |

---

## 三、User 层特征（统计聚合）

User 层特征全部从 Task 层标签 + Turn 层统计聚合得出，零 LLM 成本。

### 3.1 分布型特征

对每个用户，统计其所有 task 的标签分布。

```python
user_features = {
    # 分布型（每个值的占比）
    "task_type_dist": {"write_code": 0.4, "debug": 0.3, ...},
    "domain_dist": {"web_frontend": 0.5, "data_science": 0.2, ...},
    "role_dist": {"developer": 0.8, "data_analyst": 0.2},
    "industry_dist": {"finance": 0.6, "ecommerce": 0.3, ...},
    "intent_dist": {"productivity": 0.5, "learning": 0.3, ...},
    "ai_dependency_dist": {"significant_help": 0.4, "cannot_do_without": 0.3, ...},
    "complexity_dist": {"simple": 0.3, "moderate": 0.5, ...},
    "deliverable_dist": {"code_snippet": 0.4, "complete_script": 0.3, ...},
    "scope_dist": {"function": 0.4, "module": 0.3, ...},
    "project_phase_dist": {"active_development": 0.5, "prototyping": 0.2, ...},
}
```

### 3.2 主标签特征

从分布中取众数或加权结果。

```python
user_primary = {
    "primary_role": "developer",           # role_dist 的众数
    "primary_industry": "finance",         # industry_dist 的众数（排除 unclear）
    "primary_domain": "web_frontend",      # domain_dist 的众数
    "primary_intent": "productivity",      # intent_dist 的众数
    "skill_level": "intermediate",         # user_skill_signal 的中位数或众数
    "ai_dependency_mode": "significant_help",  # ai_dependency_dist 的众数
}
```

### 3.3 行为统计特征

```python
user_behavior = {
    # 任务维度
    "total_tasks": 47,
    "task_success_rate": 0.72,             # task_outcome=success 的比例
    "task_failure_rate": 0.15,             # task_outcome=failure 的比例
    "capability_gap_rate": 0.08,           # is_within_capability=false 的比例
    "correction_rate": 0.23,              # 所有 task 的 correction_rate 的均值
    "task_diversity": 5,                   # 使用过的不同 task_type 数量
    "tech_stack_breadth": 8,              # 使用过的不同 tech_stack 数量
    "domain_breadth": 3,                  # 涉及的不同 domain 数量
    
    # 复杂度维度
    "avg_complexity": 2.3,                # task_complexity 的数值均值
    "max_complexity": "expert",           # 最高复杂度
    "complex_task_ratio": 0.15,           # complex+expert 占比
    
    # 交付物维度
    "primary_deliverable": "code_snippet",
    "deliverable_diversity": 4,           # 不同 deliverable 类型数量
    
    # 探索行为维度
    "comfort_zone_ratio": 0.7,            # comfort_zone=familiar 的比例
    "exploration_ratio": 0.3,             # comfort_zone=exploring 的比例
    
    # 团队与决策信号
    "team_signal_mode": "small_team",     # team_signal 的众数
    "codebase_scale_mode": "startup",     # codebase_scale 的众数
    "decision_authority_mode": "influencer",
}
```

### 3.4 生命周期特征

```python
user_lifecycle = {
    "first_task_date": "2024-01-15",
    "last_task_date": "2024-06-20",
    "tenure_days": 157,                   # 从首次到末次的天数
    "days_since_last_use": 3,
    "total_active_days": 45,              # 有任务的天数
    "usage_frequency": 0.29,              # active_days / tenure_days
    
    # 首次体验
    "first_task_type": "learn",
    "first_task_outcome": "success",
    "first_week_tasks": 5,
    "first_week_success_rate": 0.6,
    "time_to_first_success_minutes": 15,
    "activation_speed_days": 2,
    
    # 生命周期阶段
    "lifecycle_stage": "engaged",         # new/activated/engaged/power_user/declining/dormant/churned
}
```
