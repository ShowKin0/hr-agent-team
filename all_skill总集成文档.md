# all_skill 总集成文档

> **本文档是 5 个独立项目合并成 1 个作品的"总契约"，优先级最高。**
> 文档层级：`all_skill总集成文档.md`（总目标/接口/合并） > `DESP技能规范接口v1.0.md`（单技能契约） > 各自实现。
> 任何接口冲突，以本文档为准；改接口必须先改文档、再改代码。

---

## 1. 共同目标（North Star）

### 1.1 一句话使命

> **给一句模糊的招聘诉求，5 个数字员工自主协作，端到端完成"需求澄清 → JD → 简历筛选 → 面试 → 测评 → offer → 入职计划"的完整闭环，交付 13 类可验收成果（报告/表格/网页），全程可见工具调用与人工确认节点。**

### 1.2 端到端验收标准（合并后必须同时满足）

| 指标 | 达标线 |
|---|---|
| 主链路跑通时间 | 单条招聘全流程 ≤ 5 分钟 |
| 可验收成果 | ≥ 3 类，且必须包含报告、表格、网页中的至少两类 |
| 真实工具调用 | 演示中出现 ≥ 3 次（如搜索、读网页、生成文档、写表格） |
| 人工确认节点 | ≥ 1 处 human-in-the-loop（如 offer 发送前确认） |
| 多智能体协同 | 演示中 ≥ 3 个数字员工"轮流上工"可见 |
| 可解释性 | 每个判断都有 `reason`，无"拍脑袋"结论 |

### 1.3 最终交付物清单（作品对外交付）

1. 需求澄清报告（A）
2. 结构化 JD（A）
3. 候选人能力画像（A）
4. 简历结构化表（B）
5. 初筛打分表（B）
6. 面试题纲 + 纪要 + 评分表（C）
7. 技能测评报告 + 风险清单 + 文化匹配报告（D）
8. offer 草案 + 谈薪建议表 + 30 天入职计划（E）

---

## 2. 共同接口（Common Interface）

> 接口分三层：**技能契约层 / 数据契约层 / 运行环境层**。合并能否成功，取决于这三层是否全部对齐。

### 2.1 技能契约层（引用 DESP v1.0）

- 每个技能的 `input_schema` / `output_schema` 必须符合《DESP技能规范接口v1.0.md》第三章字段规范；
- 每个技能的输入/输出字段，**必须使用本文档 2.2 定义的数据契约结构**，不得自造结构。

### 2.2 数据契约层（★ 合并的核心，5 人必须共享这 8 个结构）

> 规则：**每个数据结构的字段名、类型、必填项全团队冻结**。任何一个人改字段，必须全员同步，否则接不起来。

#### ① DemandClarified（需求澄清）— A 产出，B 消费
```json
{
  "clarified": "string",          // 澄清后需求陈述
  "missing":   ["string"],        // 待确认信息
  "questions": ["string"]         // 追问清单
}
```

#### ② JD（结构化岗位）— A 产出，B/C 消费
```json
{
  "jd_id": "string",
  "title": "string",
  "responsibilities": ["string"],
  "hard_requirements": ["string"],
  "nice_to_have": ["string"]
}
```

#### ③ Persona（能力画像）— A 产出，D 消费
```json
{
  "core_competencies": ["string"],
  "soft_traits": ["string"],
  "culture_fit_hint": "string"
}
```

#### ④ Resume（结构化简历）— B 产出，B/C/D 消费
```json
{
  "resume_id": "string",
  "name": "string",
  "years": 0,
  "skills": ["string"],
  "experiences": ["string"],
  "education": "string"
}
```

#### ⑤ ScoreResult（初筛打分）— B 产出，C 消费
```json
{
  "score": 0,
  "verdict": "shortlist|hold|reject",
  "matched": [{ "skill": "string", "evidence": "string" }],
  "gaps": ["string"],
  "reason": "string"
}
```

#### ⑥ InterviewPlan / InterviewMinutes（面试）— C 产出
```json
{ "questions": [{ "text": "string", "intent": "string" }],
  "rubric": [{ "dimension": "string", "weight": 0 }] }
```
```json
{ "summary": "string",
  "qa": [{ "question": "string", "answer": "string", "score": 0 }],
  "verdict": "shortlist|hold|reject" }
```

#### ⑦ AssessmentReport（综合测评）— D 产出，E 消费
```json
{
  "skill_level": "expert|proficient|basic|none",
  "skill_score": 0,
  "integrity_risk": "low|medium|high",
  "findings": [{ "item": "string", "risk": "string", "evidence": "string" }],
  "culture_fit": 0,
  "fills_gap": ["string"],
  "verdict": "shortlist|hold|reject"
}
```

#### ⑧ Offer / OnboardingPlan（offer 与入职）— E 产出
```json
{ "position": "string", "salary_range": "string",
  "suggested": "string", "clauses": ["string"] }
```
```json
{ "week1_4": ["string"], "goals": ["string"],
  "screening_feedback": "string" }
```

### 2.3 共享数据库 schema（状态外置的唯一存放处）

```yaml
CandidateRecord:            # 候选人档案 = 贯穿 5 个 agent 的"脊椎"
  candidate_id: string
  resume: Resume            # B 写入
  score: ScoreResult        # B 写入
  interview_minutes: InterviewMinutes   # C 写入
  assessment: AssessmentReport          # D 写入
  offer: Offer              # E 写入
  status: string            # 当前阶段，由调度器维护
```

> **解耦关键**：5 个 agent 不互相传对象，而是各自往 `CandidateRecord` 写自己的字段。谁慢了、谁掉队了，都不阻塞别人读自己需要的前置字段。

### 2.4 任务消息格式（任务总线统一）

```yaml
task:
  task_id: "T-2026-001"
  goal: "为'数据分析岗'完成一轮招聘，直到生成 offer 草案"
  status: pending | claimed | running | delivered | accepted | failed | escalated
  current_state: ["jd_ready", "resume_parsed"]
  payload: { ... }
  assignee: "简历猎手"
  created_by: "调度器"
```

### 2.5 全局状态键流转图（调度器规划的依据）

```
demand_clarified ──► jd_ready ──► persona_ready
                                     │
resume_collected ─► resume_parsed ─► resume_scored ─► shortlist_ready
                                                          │
interview_planned ─► interview_conducted ─► assessed ─► offer_generated
                                                            │
                                              offer_accepted ─► onboarded
```

### 2.6 统一工具命名空间（避免重名冲突）

| 工具 | 用途 | 归属 |
|---|---|---|
| `llm` | 通用大模型调用 | 共享 |
| `knowledge_base` | 知识库检索 | A / E |
| `vector_search` | 向量相似检索 | B / D |
| `resume_parser` | 简历解析 | B |
| `ocr` | 图片/PDF 文字提取 | B |
| `asr` | 语音转写 | C |
| `code_runner` | 技能测评运行代码 | D |
| `doc_generator` | 生成文档/PDF | E |

规则：小写、下划线、动词+名词或领域名；**新增工具必须先登记到本文档**，防止 5 人各起名导致冲突。

### 2.7 统一 LLM 与提示词规范

| 项 | 约定 |
|---|---|
| 输出模式 | 统一 JSON 模式（符合数据契约） |
| 温度 | 0–0.3（保证确定性、可复现） |
| 必带字段 | 凡有判断必输出 `reason` |
| 术语 | 严格按风格白皮书（候选人/岗位/offer…） |
| 骨架 | 文字结论一律"结论→依据→风险→下一步" |

---

## 3. 项目合并计划（Merge Plan）

### 3.1 合并原则

1. **接口冻结优先**：先冻结本文档与《DESP》，再并行开发；冻结后改接口须全员开会；
2. **流水线顺序集成**：严格按 A→B→C→D→E 依赖顺序合并，不做跳跃；
3. **失败可 mock**：任何人没做完，用"占位数据 + 转人工"兜底，不阻塞整条链路；
4. **契约测试先行**：每对接口先写验收用例（见 3.4），再联调。

### 3.2 合并顺序（依赖链）

```
A(需求澄清/JD) → B(简历筛选) → C(面试) → D(测评) → E(offer/入职)
```

### 3.3 分阶段集成

| 阶段 | 时间 | 内容 | 出口标准 |
|---|---|---|---|
| Phase 0 | D1–2 | 冻结接口：两份文档定稿，数据契约、枚举、术语全员签字 | 8 个数据结构 + 状态键 + 枚举无争议 |
| Phase 1 | D3–6 | 单技能自测：每人按契约跑通自己的技能 | 每技能输出通过 `output_schema` 校验 |
| Phase 2 | D7 | 两两对接：A↔B、B↔C、C↔D、D↔E | 每对接口的验收用例通过 |
| Phase 3 | D8–9 | 全链路联调：从一句话跑通到 offer/入职 | 端到端 ≤ 5 分钟，满足 1.2 全部指标 |
| Phase 4 | D10–14 | 交付打磨：演示、录屏、设计文档、商业分析 | 作品可对外答辩 |

### 3.4 集成测试用例（每对接口必过）

| 接口 | 用例 | 期望 |
|---|---|---|
| A→B | 输入"招个会做数据的人" | B 能拿到合法 JD，不缺字段 |
| B→C | 输入一份简历原文 | C 能拿到 `Resume` + `ScoreResult` |
| C→D | 输入面试问答 | D 能拿到 `InterviewMinutes` |
| D→E | 输入测评结论 | E 能拿到 `AssessmentReport` |
| 全程 | 一条模糊指令 | 13 类成果至少产出 3 类，含报告/表格/网页 |

### 3.5 风险与兜底

| 风险 | 兜底 |
|---|---|
| 某人掉队没做完 | 用 mock 数据填充该技能输出，标"转人工" |
| 数据契约字段对不上 | 以本文档为准，改实现不改接口 |
| 主链路某步不稳定 | 提前准备降级数据，保证演示主路径稳定 |
| 模型输出不符合 schema | 重试 1 次 + 质检关卡退回 + 最终转人工 |

### 3.6 合并完成的定义（Done）

合并完成 = **一句话模糊指令 → 5 个数字员工协同 → 产出 ≥3 类可验收成果 → 满足 1.2 全部指标**。缺一不算完成。

---

## 4. 协作规范

### 4.1 仓库结构（建议）

```
hr-agent-team/
├── docs/
│   ├── DESP技能规范接口v1.0.md
│   ├── all_skill总集成文档.md
│   └── 团队分工与交付说明.md
├── agents/
│   ├── analyst/        # A 招聘分析师
│   ├── scout/          # B 简历猎手
│   ├── interviewer/    # C 面试官
│   ├── assessor/       # D 测评背调员
│   └── concierge/      # E offer 管家
├── shared/             # 调度器、任务总线、数据库 schema、风格白皮书
└── demo/               # 演示视频、截图、设计文档
```

### 4.2 命名与提交

- 技能文件命名：`skill.<领域>.<动作>.yaml`（如 `skill.resume.score.yaml`）；
- 提交信息格式：`[agent] 动作 说明`（如 `[scout] add resume.score 初筛打分`）；
- 接口变更单独提交，并在 commit 里写明"BREAKING"或"兼容"。

### 4.3 每日同步（建议 15 分钟）

每人只说三句话：**今天完成了什么可验收成果 / 明天做什么 / 被什么接口卡住**。

---

## 附录 A：数据契约与产出/消费对照表

| 数据契约 | 产出者 | 消费方 |
|---|---|---|
| DemandClarified | A | （文档留存） |
| JD | A | B、C |
| Persona | A | D |
| Resume | B | B、C、D |
| ScoreResult | B | C |
| InterviewPlan/Minutes | C | D |
| AssessmentReport | D | E |
| Offer / OnboardingPlan | E | （交付） |

## 附录 B：枚举总表

| 枚举 | 取值 |
|---|---|
| `verdict` | `shortlist` / `hold` / `reject` |
| `risk_level` | `low` / `medium` / `high` |
| `skill_level` | `expert` / `proficient` / `basic` / `none` |
| `task.status` | `pending` / `claimed` / `running` / `delivered` / `accepted` / `failed` / `escalated` |

---

*本文档是合并的"宪法"。所有人开工前先读 2.2 数据契约和附录 A 对照表，确认自己的输入/输出字段与他人完全对齐后再动手。*
