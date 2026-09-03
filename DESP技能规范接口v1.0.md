# DESP 数字员工技能协议 v1.0

**Digital Employee Skill Protocol（数字员工技能协议）**

| 项目 | 内容 |
|---|---|
| 协议版本 | v1.0.0 |
| 作品方向 | 招聘 HR 数字员工（多智能体协同） |
| 状态 | 草案 · 供团队内部开发与比赛技术文档使用 |
| 修订日期 | 2026-08-28 |

---

## 目录

1. [协议概述](#一协议概述)
2. [核心概念](#二核心概念)
3. [技能契约字段规范](#三技能契约字段规范)
4. [组织架构：5 个数字员工](#四组织架构5-个数字员工)
5. [12 个技能定义](#五12-个技能定义)
6. [解耦规则](#六解耦规则)
7. [风格一致性白皮书](#七风格一致性白皮书)
8. [任务总线与调度机制](#八任务总线与调度机制)
9. [错误处理与降级规范](#九错误处理与降级规范)
10. [版本管理](#十版本管理)
11. [团队分工](#十一团队分工)
12. [附录：术语与枚举](#附录术语与枚举)

---

## 一、协议概述

### 1.1 DESP 是什么

DESP 是一套用于定义"数字员工技能（Skill）"的标准化契约。它把每个数字员工对外提供的能力，抽象成一份**可被机器读取、可被调度器自动发现和组合**的契约文件（YAML / JSON）。

一个 Skill 契约回答了三个问题：

1. **我是什么**：这个技能叫什么、属于谁、解决什么问题（`name` / `owner` / `description`）；
2. **怎么调我**：输入长什么样、输出长什么样、需要什么前置条件、会产生什么影响（`input_schema` / `output_schema` / `preconditions` / `effects`）；
3. **边界在哪**：我能碰什么工具、不能碰什么数据、失败怎么办、风格如何（`tools` / `permissions` / `error_contract` / `style`）。

### 1.2 DESP 解决什么问题

多智能体系统最容易烂在三件事上，DESP 逐一给出硬约束：

| 常见问题 | DESP 的对策 |
|---|---|
| 5 个人各写各的，接口不统一，接不起来（耦合） | 统一 `input_schema` / `output_schema`，契约优先 |
| A 直接调用 B 的代码，改一处全崩（强耦合） | 禁止直连，只通过任务总线 + 契约通信 |
| 5 个 agent 输出风格五花八门，不像一个团队 | `style` 字段 + 统一枚举 + 质检关卡 |
| 调度逻辑写死在代码里，加个 agent 要改一堆 | 发现式路由：调度器读契约动态决策 |

### 1.3 设计原则

1. **契约优先**：先定接口，再写实现；
2. **无状态技能**：技能只做"输入→输出"，状态全部进共享数据库；
3. **单一职责**：一个技能只做一件事；
4. **可验收**：每个输出都有明确的 `output_schema` 和判定枚举，能被客观校验；
5. **可解释**：凡有判断，必给 `reason`。

---

## 二、核心概念

| 概念 | 定义 |
|---|---|
| **数字员工（Employee）** | 一个有岗位职责的智能体，对外暴露若干技能。本作品共 5 名：招聘分析师、简历猎手、面试官、测评背调员、offer 与入职管家。 |
| **技能（Skill）** | 数字员工对外提供的最小可调用能力，一份契约文件 + 一个可执行实现。本作品共 12 个技能。 |
| **调度器（Dispatcher）** | 共享层，不占数字员工名额。读取技能契约，负责把任务拆解、路由、组合、验收。 |
| **任务总线（Task Bus）** | 技能之间传递消息的唯一通道，是一个共享的任务状态表（或消息队列）。任何技能不得绕过它。 |
| **共享数据库（Shared State）** | 全局状态（候选人档案、招聘进程、团队画像等）的唯一存放处。技能本身无状态。 |
| **技能注册表（Skill Registry）** | 全部技能契约的集合，调度器从这里发现技能。 |

```
用户指令
   │
   ▼
调度器（读技能注册表，规划，路由）
   │
   ▼
任务总线 ──► 技能A ──► 任务总线 ──► 技能B ──► ... ──► 交付物（可验收）
   │                                            ▲
   └──────────── 共享数据库（全局状态）──────────┘
```

---

## 三、技能契约字段规范

一个 Skill 契约由以下字段组成。**`*` 为必填字段**。

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `id`* | string | ✅ | 全局唯一，命名规则 `领域.动作`，如 `resume.score` |
| `name`* | string | ✅ | 中文名 |
| `owner`* | string | ✅ | 归属的数字员工 |
| `version`* | string | ✅ | 语义化版本，如 `1.0.0` |
| `description`* | string | ✅ | 给调度器看的"何时用我"的能力描述，写清楚适用场景 |
| `tags` | string[] | — | 便于检索的分类标签 |
| `input_schema`* | object(JSON Schema) | ✅ | 输入契约，定义参数、类型、必填项 |
| `output_schema`* | object(JSON Schema) | ✅ | 输出契约，定义返回结构、枚举、必填项 |
| `preconditions`* | string[] | ✅ | 前置条件，引用全局状态键，全部满足才可执行 |
| `effects`* | string[] | ✅ | 执行后产生/满足的状态键 |
| `side_effects` | string[] | — | 会改写的全局状态键；为 `[]` 表示纯只读技能 |
| `tools` | string[] | — | 本技能可调用的工具/插件 |
| `permissions` | string[] | — | 数据与操作边界，如 `read:resumes`、`no:send_email` |
| `style` | object | — | 风格覆盖项；默认继承全局风格白皮书（第七章） |
| `error_contract` | object | — | 失败、重试、降级约定 |
| `examples` | array | — | 输入输出示例，供调度器与大模型做 few-shot 理解 |

### 3.1 `output_schema` 的统一约定（强约束）

- 所有"判定类"输出必须用统一枚举 `verdict`：`shortlist`（进入下一轮）/ `hold`（待定，转人工）/ `reject`（淘汰）；
- 所有"评分类"字段统一刻度 `0–100`，整型；
- 凡有判断，必须输出 `reason`（可解释理由）；
- 输出骨架统一为四段：**结论 → 依据 → 风险 → 下一步**（见第七章）。

### 3.2 `preconditions` / `effects` 使用的状态键

全局状态键统一定义（附录 3），示例：

| 状态键 | 含义 |
|---|---|
| `demand_clarified` | 招聘需求已澄清 |
| `jd_ready` | 结构化 JD 已就绪 |
| `persona_ready` | 候选人能力画像已就绪 |
| `resume_parsed` | 简历已结构化解析 |
| `resume_scored` | 简历已完成打分 |
| `shortlist_ready` | 初筛通过名单已就绪 |
| `interview_conducted` | 面试已进行 |
| `assessed` | 测评与背调已完成 |
| `offer_generated` | offer 已生成 |
| `offer_accepted` | 候选人已接受 offer |
| `onboarded` | 入职流程已启动 |

---

## 四、组织架构：5 个数字员工

| # | 数字员工 | 岗位职责 | 暴露技能 |
|---|---|---|---|
| 1 | 招聘分析师 | 把模糊业务诉求转成结构化 JD 与能力画像 | `demand.clarify`、`jd.generate`、`persona.build` |
| 2 | 简历猎手 | 简历结构化解析 + 初筛打分 | `resume.parse`、`resume.score` |
| 3 | 面试官 | 出题 + 动态追问 + 面试纪要 | `interview.plan`、`interview.run` |
| 4 | 测评背调员 | 技能测评、真实性核查、文化匹配 | `assess.skill`、`assess.integrity`、`assess.culture` |
| 5 | offer 与入职管家 | offer 生成 + 30 天入职计划 | `offer.generate`、`onboard.plan` |

**共享层（不占数字员工名额）**：调度器、技能注册表、任务总线、共享数据库、质检关卡。

---

## 五、12 个技能定义

> 各技能 `style` 未列出的项默认继承全局风格白皮书（第七章）。

### 5.1 招聘分析师

#### 5.1.1 `demand.clarify` 需求澄清追问

```yaml
skill:
  id: "demand.clarify"
  name: "需求澄清追问"
  owner: "招聘分析师"
  version: "1.0.0"
  description: "当业务方只给了模糊招聘诉求（如'我要个会做数据的人'）时，主动追问岗位目标、硬性门槛、软性偏好，生成结构化需求澄清结果。"
  tags: ["hr", "demand", "clarify"]
  input_schema:
    type: object
    properties:
      raw_demand: { type: string, description: "业务方原始诉求" }
      context:    { type: string, description: "可选，团队/项目背景" }
    required: [raw_demand]
  output_schema:
    type: object
    properties:
      clarified: { type: string, description: "澄清后的完整需求陈述" }
      missing:   { type: array, items: { type: string }, description: "仍待确认的信息点" }
      questions: { type: array, items: { type: string }, description: "生成给业务方的追问清单" }
      reason:    { type: string }
    required: [clarified, missing, questions]
  preconditions: []
  effects: ["demand_clarified"]
  side_effects: []
  tools: ["llm"]
  permissions: ["read:team_context"]
  error_contract: { on_failure: "返回原诉求 + questions 追加'请人工确认'", retry: 0, fallback: "escalate_to_human" }
  examples:
    - input:  { raw_demand: "招一个会做数据的人" }
      output: { clarified: "招聘数据分析岗，负责业务报表与指标建设", missing: ["年限要求", "工具栈"], questions: ["数据量级多大？", "是否要求 Python？"] }
```

#### 5.1.2 `jd.generate` 结构化 JD 生成

```yaml
skill:
  id: "jd.generate"
  name: "结构化 JD 生成"
  owner: "招聘分析师"
  version: "1.0.0"
  description: "根据已澄清的需求生成结构化岗位描述：职责、硬性要求、加分项、协作关系。"
  tags: ["hr", "jd"]
  input_schema:
    type: object
    properties:
      clarified_demand: { type: string, description: "澄清后的需求" }
      company_context:  { type: string, description: "公司/团队背景" }
    required: [clarified_demand]
  output_schema:
    type: object
    properties:
      jd:
        type: object
        properties:
          title: { type: string }
          responsibilities: { type: array, items: { type: string } }
          hard_requirements: { type: array, items: { type: string } }
          nice_to_have: { type: array, items: { type: string } }
        required: [title, responsibilities, hard_requirements]
      reason: { type: string }
    required: [jd]
  preconditions: ["demand_clarified"]
  effects: ["jd_ready"]
  side_effects: []
  tools: ["llm", "knowledge_base"]
  permissions: ["read:team_context"]
  error_contract: { on_failure: "返回占位 JD 并标记待人工补全", retry: 1, fallback: "escalate_to_human" }
```

#### 5.1.3 `persona.build` 候选人能力画像构建

```yaml
skill:
  id: "persona.build"
  name: "候选人能力画像构建"
  owner: "招聘分析师"
  version: "1.0.0"
  description: "基于 JD 与团队画像，构建目标候选人的能力/性格画像，供筛选与面试使用。"
  tags: ["hr", "persona"]
  input_schema:
    type: object
    properties:
      jd:          { type: object, description: "结构化 JD" }
      team_profile: { type: object, description: "现有团队画像（团队拼图维度）" }
    required: [jd]
  output_schema:
    type: object
    properties:
      persona:
        type: object
        properties:
          core_competencies: { type: array, items: { type: string } }
          soft_traits:       { type: array, items: { type: string } }
          culture_fit_hint:  { type: string }
        required: [core_competencies]
      reason: { type: string }
    required: [persona]
  preconditions: ["jd_ready"]
  effects: ["persona_ready"]
  side_effects: []
  tools: ["llm", "vector_search"]
  permissions: ["read:team_profile"]
  error_contract: { on_failure: "返回最小画像 + reason 说明降级", retry: 1, fallback: "escalate_to_human" }
```

### 5.2 简历猎手

#### 5.2.1 `resume.parse` 简历结构化解析

```yaml
skill:
  id: "resume.parse"
  name: "简历结构化解析"
  owner: "简历猎手"
  version: "1.0.0"
  description: "把任意格式的简历（文本/图片/PDF 文本）解析为统一结构化字段，供后续打分与检索使用。"
  tags: ["hr", "resume", "parse"]
  input_schema:
    type: object
    properties:
      resume_raw: { type: string, description: "简历原文或可提取文本" }
    required: [resume_raw]
  output_schema:
    type: object
    properties:
      resume:
        type: object
        properties:
          name: { type: string }
          years: { type: number }
          skills: { type: array, items: { type: string } }
          experiences: { type: array, items: { type: string } }
          education: { type: string }
        required: [name, skills, experiences]
      reason: { type: string }
    required: [resume]
  preconditions: ["resume_collected"]
  effects: ["resume_parsed"]
  side_effects: []
  tools: ["ocr", "resume_parser"]
  permissions: ["read:resumes"]
  error_contract: { on_failure: "标记该简历为 hold 并转人工解析", retry: 1, fallback: "escalate_to_human" }
```

#### 5.2.2 `resume.score` 简历初筛打分

```yaml
skill:
  id: "resume.score"
  name: "简历初筛打分"
  owner: "简历猎手"
  version: "1.0.0"
  description: "根据 JD 对结构化简历做匹配打分，输出录用建议、命中项与缺口。"
  tags: ["hr", "resume", "screening"]
  input_schema:
    type: object
    properties:
      jd:     { type: object, description: "结构化 JD" }
      resume: { type: object, description: "结构化简历" }
    required: [jd, resume]
  output_schema:
    type: object
    properties:
      score:   { type: number, minimum: 0, maximum: 100 }
      verdict: { enum: [shortlist, hold, reject] }
      matched: { type: array, items: { skill: string, evidence: string } }
      gaps:    { type: array, items: { type: string } }
      reason:  { type: string }
    required: [score, verdict, reason]
  preconditions: ["jd_ready", "resume_parsed"]
  effects: ["resume_scored"]
  side_effects: []
  tools: ["llm", "vector_search"]
  permissions: ["read:resumes"]
  error_contract: { on_failure: "return {verdict: hold, reason: '解析失败，转人工'}", retry: 1, fallback: "escalate_to_human" }
```

### 5.3 面试官

#### 5.3.1 `interview.plan` 面试题纲生成

```yaml
skill:
  id: "interview.plan"
  name: "面试题纲生成"
  owner: "面试官"
  version: "1.0.0"
  description: "根据 JD、候选人画像与简历，生成结构化面试题纲与评分标准。"
  tags: ["hr", "interview", "plan"]
  input_schema:
    type: object
    properties:
      jd:      { type: object }
      resume:  { type: object }
      persona: { type: object }
    required: [jd, resume]
  output_schema:
    type: object
    properties:
      plan:
        type: object
        properties:
          questions: { type: array, items: { text: string, intent: string } }
          rubric:    { type: array, items: { dimension: string, weight: number } }
        required: [questions, rubric]
      reason: { type: string }
    required: [plan]
  preconditions: ["jd_ready", "resume_scored"]
  effects: ["interview_planned"]
  side_effects: []
  tools: ["llm"]
  permissions: ["read:resumes"]
  error_contract: { on_failure: "返回通用题纲模板", retry: 1, fallback: "escalate_to_human" }
```

#### 5.3.2 `interview.run` 面试执行与纪要

```yaml
skill:
  id: "interview.run"
  name: "面试执行与纪要"
  owner: "面试官"
  version: "1.0.0"
  description: "按题纲进行多轮动态追问，实时记录面试过程，生成结构化面试纪要、评分与录用建议。"
  tags: ["hr", "interview", "run"]
  input_schema:
    type: object
    properties:
      plan:       { type: object, description: "面试题纲" }
      transcript: { type: string, description: "候选人回答记录" }
    required: [plan, transcript]
  output_schema:
    type: object
    properties:
      minutes:
        type: object
        properties:
          summary: { type: string }
          qa:      { type: array, items: { question: string, answer: string, score: number } }
          verdict: { enum: [shortlist, hold, reject] }
        required: [summary, qa, verdict]
      reason: { type: string }
    required: [minutes]
  preconditions: ["interview_planned"]
  effects: ["interview_conducted"]
  side_effects: []
  tools: ["llm", "asr"]
  permissions: ["read:resumes"]
  error_contract: { on_failure: "保留原始记录转人工整理", retry: 1, fallback: "escalate_to_human" }
```

### 5.4 测评背调员

#### 5.4.1 `assess.skill` 技能测评

```yaml
skill:
  id: "assess.skill"
  name: "技能测评"
  owner: "测评背调员"
  version: "1.0.0"
  description: "根据岗位要求生成/执行技能测评题，并给出能力等级评定。"
  tags: ["hr", "assess", "skill"]
  input_schema:
    type: object
    properties:
      jd:     { type: object }
      resume: { type: object }
      answers: { type: array, items: { type: string }, description: "候选人答题记录（可空，则仅生成题）" }
    required: [jd, resume]
  output_schema:
    type: object
    properties:
      level:  { enum: [expert, proficient, basic, none] }
      score:  { type: number, minimum: 0, maximum: 100 }
      report: { type: string }
      reason: { type: string }
    required: [level, score, reason]
  preconditions: ["interview_conducted"]
  effects: ["assessed"]
  side_effects: []
  tools: ["llm", "code_runner"]
  permissions: ["read:resumes"]
  error_contract: { on_failure: "标记待人工测评", retry: 1, fallback: "escalate_to_human" }
```

#### 5.4.2 `assess.integrity` 真实性核查

```yaml
skill:
  id: "assess.integrity"
  name: "真实性核查"
  owner: "测评背调员"
  version: "1.0.0"
  description: "对简历与面试中的关键事实（经历、学历、时间线）做一致性与真实性核查，输出风险清单。"
  tags: ["hr", "assess", "integrity"]
  input_schema:
    type: object
    properties:
      resume:  { type: object }
      minutes: { type: object }
    required: [resume, minutes]
  output_schema:
    type: object
    properties:
      risk_level: { enum: [low, medium, high] }
      findings:   { type: array, items: { item: string, risk: string, evidence: string } }
      reason:     { type: string }
    required: [risk_level, findings]
  preconditions: ["interview_conducted"]
  effects: ["assessed"]
  side_effects: []
  tools: ["llm", "vector_search"]
  permissions: ["read:resumes"]
  error_contract: { on_failure: "标记为 medium 并转人工复核", retry: 1, fallback: "escalate_to_human" }
```

#### 5.4.3 `assess.culture` 文化/团队匹配

```yaml
skill:
  id: "assess.culture"
  name: "文化/团队匹配（团队拼图）"
  owner: "测评背调员"
  version: "1.0.0"
  description: "将候选人与现有团队画像做互补性匹配，输出'团队缺口补位'评分，而非单纯的岗位匹配。"
  tags: ["hr", "assess", "culture"]
  input_schema:
    type: object
    properties:
      candidate_profile: { type: object }
      team_profile:      { type: object }
    required: [candidate_profile, team_profile]
  output_schema:
    type: object
    properties:
      fit_score: { type: number, minimum: 0, maximum: 100 }
      fills_gap: { type: array, items: { type: string }, description: "补足团队的能力缺口" }
      overlaps:  { type: array, items: { type: string }, description: "与团队的冗余能力" }
      reason:    { type: string }
    required: [fit_score, fills_gap]
  preconditions: ["persona_ready"]
  effects: ["assessed"]
  side_effects: []
  tools: ["llm", "vector_search"]
  permissions: ["read:team_profile"]
  error_contract: { on_failure: "返回 fit_score 默认 50 并转人工", retry: 1, fallback: "escalate_to_human" }
```

### 5.5 offer 与入职管家

#### 5.5.1 `offer.generate` Offer 生成

```yaml
skill:
  id: "offer.generate"
  name: "Offer 生成与谈薪建议"
  owner: "offer 与入职管家"
  version: "1.0.0"
  description: "综合候选人的测评结论、市场薪资数据，生成 offer 草案与谈薪区间建议。"
  tags: ["hr", "offer"]
  input_schema:
    type: object
    properties:
      candidate_summary: { type: object, description: "候选人的测评与面试综合结论" }
      salary_data:       { type: object, description: "市场薪资基准（可选）" }
    required: [candidate_summary]
  output_schema:
    type: object
    properties:
      offer:
        type: object
        properties:
          position: { type: string }
          salary_range: { type: string }
          suggested:    { type: string }
          clauses:      { type: array, items: { type: string } }
        required: [position, salary_range]
      reason: { type: string }
    required: [offer]
  preconditions: ["assessed"]
  effects: ["offer_generated"]
  side_effects: []
  tools: ["llm", "doc_generator"]
  permissions: ["read:salary_benchmark"]
  error_contract: { on_failure: "返回 offer 模板占位", retry: 1, fallback: "escalate_to_human" }
```

#### 5.5.2 `onboard.plan` 入职计划生成

```yaml
skill:
  id: "onboard.plan"
  name: "入职与 30 天计划生成"
  owner: "offer 与入职管家"
  version: "1.0.0"
  description: "根据候选人的能力缺口与岗位，生成 30 天 onboarding 计划，并把缺口反哺回筛选标准（招聘飞轮）。"
  tags: ["hr", "onboarding"]
  input_schema:
    type: object
    properties:
      candidate_profile: { type: object }
      offer:            { type: object }
    required: [candidate_profile]
  output_schema:
    type: object
    properties:
      plan:
        type: object
        properties:
          week1_4: { type: array, items: { type: string } }
          goals:   { type: array, items: { type: string } }
        required: [week1_4]
      screening_feedback: { type: string, description: "反哺筛选标准的改进建议" }
      reason: { type: string }
    required: [plan, screening_feedback]
  preconditions: ["offer_accepted"]
  effects: ["onboarded"]
  side_effects: []
  tools: ["llm", "doc_generator"]
  permissions: ["read:team_profile"]
  error_contract: { on_failure: "返回通用 onboarding 模板", retry: 1, fallback: "escalate_to_human" }
```

---

## 六、解耦规则

### 规则 1：契约优先，禁止直连

- 技能之间**禁止直接调用对方实现代码**；
- 一切交互必须先过 `input_schema` / `output_schema`，经任务总线传递；
- 修改任何技能的内部实现，只要契约不变，**上下游零感知**。

### 规则 2：无状态技能，状态外置

- 技能自身不保存任何会话/业务状态；
- 所有状态（候选人档案、招聘进程）写入共享数据库，用全局状态键（`preconditions` / `effects`）描述；
- 好处：5 个人并行开发，互不踩状态；技能可随时替换、重跑、回放。

### 规则 3：发现式路由，不做硬编码调度

- 调度器**不写**"简历来了就调猎手"这类规则；
- 调度器读取技能注册表，依据 `description + preconditions + effects` 动态规划调用顺序；
- **验收标准**：新增或替换一个数字员工，调度器代码零改动——这是解耦的最有力证明。

### 规则 4：单一职责

- 一个技能只做一件事；一个数字员工聚合一组内聚技能；
- 拆分粒度标准：如果一个技能的输出可以再被两个以上技能复用，它就应该独立。

---

## 七、风格一致性白皮书

> 全团队共享，所有技能的 `style` 未覆盖项默认继承本白皮书。

### 7.1 统一术语表

| 统一用语 | 禁用表达 |
|---|---|
| 候选人 | 求职者、应聘者 |
| 岗位描述（JD） | 招聘广告 |
| offer / 录用要约 | 录取通知（口语） |
| 初筛通过 | 简历过了 |
| 能力画像 | 人才画像（口语） |
| 测评 | 测试 |

### 7.2 统一输出骨架

每个技能的 `reason` / 文字结论遵循四段式：

```
结论 → 依据 → 风险 → 下一步
```

### 7.3 统一枚举与刻度

| 项 | 取值 |
|---|---|
| 判定 `verdict` | `shortlist` / `hold` / `reject` |
| 评分刻度 | 整数 `0–100` |
| 风险等级 `risk_level` | `low` / `medium` / `high` |
| 能力等级 `level` | `expert` / `proficient` / `basic` / `none` |

### 7.4 统一语气

- 专业、克制、先事实后判断；
- 禁止主观臆测，凡判断必有依据；
- 不给出无法验证的结论。

### 7.5 质检关卡（风格一致性的执行机制）

调度器在**交付最终成果前**，自动跑一道共享的"风格校验"规则（非第 6 个数字员工，而是共享工具）：

1. 校验 `output_schema` 字段完整、枚举合法；
2. 校验术语表（出现"求职者"即退回重写）；
3. 校验四段式骨架是否存在；
4. 校验评分是否在 0–100。

不通过则退回对应技能重做，**风格一致性由规范强制，而非靠人自觉**。

---

## 八、任务总线与调度机制

### 8.1 任务消息格式

```yaml
task:
  task_id: "T-2026-001"
  goal: "为'数据分析岗'完成一轮招聘，直到生成 offer 草案"
  status: pending | claimed | running | delivered | accepted | failed | escalated
  current_state: [jd_ready, resume_parsed]     # 当前已满足的全局状态键
  payload: { ... }                              # 技能输入数据
  assignee: "简历猎手"                           # 当前承担者（可由调度器动态指派）
  created_by: "调度器"
```

### 8.2 调度流程

1. **意图理解**：调度器解析用户指令，拆出目标与约束；
2. **规划**：读取技能注册表，从当前状态出发，找到一条 `preconditions` 可满足的路径（如 `demand.clarify → jd.generate → resume.parse → …`）；
3. **路由**：把任务经任务总线派给对应数字员工，等待 `output_schema` 校验；
4. **验收**：校验输出，更新共享状态，判断是否达成交付目标；
5. **闭环**：交付最终成果（面试报告 / offer / 入职计划），若遇到 `fallback: escalate_to_human` 则转人工。

---

## 九、错误处理与降级规范

| 情形 | 处理 |
|---|---|
| 技能执行失败 | 按 `error_contract.retry` 重试；仍失败则 `fallback: escalate_to_human` |
| 输入不满足 `input_schema` | 拒绝执行，返回校验错误，不消耗重试 |
| 前置条件不满足 | 调度器不派发，返回"等待前置状态" |
| 输出不满足 `output_schema` / 质检不通过 | 退回该技能重做（最多 1 次） |
| 无法判定（`hold`） | 保留原始数据，明确标记，转人工复核 |

**兜底原则**：宁可"转人工"，不可"硬编造"。所有降级路径都要在交付物中留下可追溯的记录。

---

## 十、版本管理

- 协议用语义化版本：`主.次.修订`；
- 技能契约每次变更必须更新 `version` 并写 CHANGELOG；
- 兼容性约定：
  - `output_schema` 新增字段 = 次版本（向后兼容）；
  - 删除/修改必填字段、修改枚举 = 主版本（破坏性变更）；
  - `input_schema` 修改由该技能 owner 与调度器维护者共同确认。

---

## 十一、团队分工

| 成员 | 负责 | 交付物 |
|---|---|---|
| A | 招聘分析师 | `demand.clarify`、`jd.generate`、`persona.build` |
| B | 简历猎手 | `resume.parse`、`resume.score` |
| C | 面试官 | `interview.plan`、`interview.run` |
| D | 测评背调员 | `assess.skill`、`assess.integrity`、`assess.culture` |
| E | offer 与入职管家 | `offer.generate`、`onboard.plan` |
| 全员轮值/共同 | 调度器 + 技能注册表 + 任务总线 + 共享数据库 + 风格白皮书维护 | DESP 协议文档、演示脚本、设计文档 |

> 共享层是"地基"，建议先由全员一起敲定第 3、6、7 章，再各自去实现自己的技能。**接口先冻结，实现后并行**。

---

## 附录：术语与枚举

### 附录 1 术语

| 术语 | 含义 |
|---|---|
| DESP | Digital Employee Skill Protocol，数字员工技能协议 |
| 数字员工 | 暴露一组技能的智能体，有岗位职责 |
| 技能 Skill | 最小可调用能力，一份契约 + 一个实现 |
| 调度器 Dispatcher | 负责规划、路由、验收的共享编排层 |
| 任务总线 Task Bus | 技能间通信的唯一通道 |
| 质检关卡 | 交付前的共享风格/格式校验规则 |

### 附录 2 枚举汇总

| 枚举 | 取值 |
|---|---|
| `verdict` | `shortlist` / `hold` / `reject` |
| `risk_level` | `low` / `medium` / `high` |
| `level` | `expert` / `proficient` / `basic` / `none` |
| `task.status` | `pending` / `claimed` / `running` / `delivered` / `accepted` / `failed` / `escalated` |

### 附录 3 全局状态键

`demand_clarified`、`jd_ready`、`persona_ready`、`resume_collected`、`resume_parsed`、`resume_scored`、`shortlist_ready`、`interview_planned`、`interview_conducted`、`assessed`、`offer_generated`、`offer_accepted`、`onboarded`

---

*本文档既是比赛的技术方案说明，也是团队内部开发的唯一接口标准。所有实现以本协议为准，接口变更需先改文档、再改代码。*
