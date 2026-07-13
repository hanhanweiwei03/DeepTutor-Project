# HKDSE English Paper 4 Oral Practice — Module Handoff

## 1. 架构总览

```
┌──────────────────────────────────────────────────────────────────┐
│                     Frontend (Next.js 16 + React 19)             │
│                                                                  │
│  page.tsx                            market-api.ts              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐    │
│  │ Config   │→│ Reading │→│Discussion│→│  Feedback    │    │
│  │ Stage    │  │ Stage   │  │  Stage   │  │  Stage       │    │
│  └──────────┘  └──────────┘  └──────────┘  └──────────────┘    │
│       │              │            │               │              │
│       ▼              ▼            ▼               ▼              │
│  POST /oral-topics →(topic_id) → POST /oral-turn → POST         │
│                                   (NDJSON stream)  /oral-       │
│                                                    feedback      │
└──────────────────────────────────────┬───────────────────────────┘
                                       │ HTTP
┌──────────────────────────────────────▼───────────────────────────┐
│                   Backend (FastAPI, Python 3.11+)                │
│                                                                  │
│  hkdse_english.py (router)                                       │
│                                                                  │
│  OralTurnRequest(BaseModel):                                     │
│    topic_id: str                                                 │
│    history: list[dict[str, str]]  # [msg1, msg2, ...]            │
│    phase: str                     # "discussion" (unused now)    │
│                                                                  │
│  oral_turn() → _stream():                                        │
│    1. Backend determines:                                        │
│       - speaker (candidate_a/b/c / examiner)                     │
│       - continues (bool)                                         │
│       - next_phase (discussion / individual_response / feedback)  │
│       - next_speaker (who speaks next)                           │
│    2. Build prompt → call llm_stream()                           │
│    3. Stream chunks directly (NO header parsing needed)          │
│    4. Send turn_end with ALL metadata (backend-computed)         │
│                                                                  │
│  ┌──────────────────────────────────────────────────────┐        │
│  │ LLM: deepseek-v4-flash (via deeptutor.services.llm)  │        │
│  │   · complete() → str (non-streaming, for feedback)   │        │
│  │   · stream()  → AsyncGenerator[str] (for discussion) │        │
│  └──────────────────────────────────────────────────────┘        │
└──────────────────────────────────────────────────────────────────┘
```

## 2. API 端点

### `POST /api/v1/hkdse/english/oral-topics`
**功能：** 按分类随机选一个话题

请求：
```json
{ "category": "social_issues" }
```
返回：
```json
{
  "topic_id": "2019_set2_selfies",
  "topic": "Selfies",
  "article": "...",
  "discussion_task": "...",
  "guiding_questions": ["..."],
  "part_b_questions": ["..."],
  "category": "social_issues"
}
```

### `POST /api/v1/hkdse/english/oral-turn`
**功能：** 生成下一轮 AI 发言（流式）

请求：
```json
{
  "topic_id": "2019_set2_selfies",
  "history": [
    { "speaker": "candidate_a", "content": "..." },
    { "speaker": "candidate_d", "content": "..." }
  ],
  "phase": "discussion"
}
```

响应（NDJSON）：
```
{"type":"turn_start"}
{"type":"chunk","content":"I think this is..."}
{"type":"chunk","content":" an important issue..."}
...
{"type":"turn_end","speaker":"candidate_b","content":"...","next_speaker":"candidate_c","continues":true,"phase":"discussion"}
{"type":"error","message":"..."}
```

### `POST /api/v1/hkdse/english/oral-feedback`
**功能：** 生成整体评分

请求同上。返回 `OralFeedbackResult`（四维度评分）。

## 3. 后端关键设计：方案 B — 后端全控元数据

| 数据项 | 控制方 | 说明 |
|--------|--------|------|
| `speaker` | 后端代码 | `_pick_next_ai_speaker()` 加权随机分配，排除连说 |
| `continues` | 后端代码 | 连续 AI 发言 ≤ 2 次，`_count_consecutive_ai()` + `_MAX_CONSECUTIVE_AI` |
| `next_phase` | 后端代码 | 用户发言 ≥ 5 次 → `individual_response` |
| `next_speaker` | 后端代码 | 旋转规则：A→B→C→A 或 handoff 给 `candidate_d` |
| 正文内容 | LLM 输出 | 模型只说纯文字，无任何头部/标签 |

**Part A → Part B 过渡：**
- `_count_user_turns(history)` 计数用户发言
- 用户发言 ≥ 5 次 + 考官尚未说话 → 生成考官提问
- 考官说话后 → 用户回答 → 收尾 → feedback

**Speaker 列表与渲染：**

| `speaker` | 前端标签 | 颜色 |
|-----------|---------|------|
| `candidate_a` | Candidate A | 蓝色 |
| `candidate_b` | Candidate B | 琥珀色 |
| `candidate_c` | Candidate C | 紫色 |
| `candidate_d` | You | 翠绿色 |
| `examiner` | Examiner | 玫红色 |

## 4. 前端状态机

```
    ┌──────────┐   select category   ┌──────────┐
    │  Config  │ ──────────────────→ │ Reading  │
    │  Stage   │                     │ 10min    │
    └──────────┘                     │ timer    │
         ↑                           └────┬─────┘
         │                                │ skip / expire
         │                          ┌─────▼──────┐
         │                          │ Discussion │ ← auto-sendTurn([])
         │                          │ Part A     │ → AI speaks
         │                          │   ─→ user  │ → AI rotates
         │                          │  (×5)      │ → Part B
         │                          │ Part B     │ → examiner
         │                          │   ─→ user  │ → feedback
         │                          └─────┬──────┘
         │                                │
    ┌────┴─────┐                    ┌─────▼──────┐
    │  Error   │                    │  Feedback  │
    │  Stage   │                    │  4-dims    │
    └──────────┘                    └────────────┘
```

**关键交互逻辑（`sendTurn + onTurnEnd`）：**
1. `next_speaker === "feedback"` → 调用 `getOralFeedback`
2. `event.phase === "individual_response"` → 同步更新 `phaseRef`
3. `next_speaker === "candidate_d"` → 启用用户输入
4. 其他 → 500ms 后自动触发下一轮 `sendTurn`

**Phase 同步问题解决方案：** 使用 `phaseRef`（useRef）存储当前 phase，避免 React 异步 state 更新带来的 stale closure 问题。`sendTurn` 始终从 ref 读取 phase。

## 5. 文件清单

| 文件 | 类型 | 行数 | 说明 |
|------|------|------|------|
| `deeptutor/api/routers/hkdse_english.py` | 后端 | ~735 | F4 节: 305-645 (topics/turn/feedback) |
| `web/app/(utility)/market/hkdse/english/oral-practice/page.tsx` | 前端 | ~646 | 完整页面，4 个子组件 |
| `web/lib/market-api.ts` | 前端 | ~80 | API: getRandomOralTopic/takeOralTurn/getOralFeedback |
| `web/types/market.ts` | 前端 | ~40 | OralMessage / OralTurnRequest / OralFeedbackResult |
| `web/app/(utility)/market/hkdse/english/page.tsx` | 前端 | — | Landing 页，F4 card |
| `oral_topics.json` | 数据 | ~590 | 24 个 2019 DSE Paper 4 话题 (OCR 生成) |

## 6. 当前问题

### 问题 1：模型输出 `<think>` 思考过程（核心阻塞）

**测试输出（`test_3.txt`）：**
```
Candidate A
<think>First, I need to open the discussion as Candidate A. The topic is about
whether selective killing should be used to control animal numbers in Hong
Kong. I should introduce the topic and share my first opinion...[300字思考]...
I'll use this.</think>Alright, let's get started. I think governments use...
```

**根因：**
- 模型（deepseek-v4-flash）自带 chain-of-thought 能力，用 `<think>...</think>` 标签包装
- 当前 system prompt（`_build_oral_system_prompt`）试图靠文字禁止思考，但模型忽略
- 后端 `_stream()` 直接将 LLM 输出逐 token 转发，包含 `<think>` 标签及其内容

**影响：**
- 用户看到上百字思考过程，极其困惑
- 每次发言延迟增加（思考本身消耗 token）
- 体验完全不符合"自然对话"设计目标

**尝试过的方案均已失效：**
- ❌ "NEVER think out loud"、"NEVER explain" 等文字指令 → 模型无视
- ❌ 从 `SPEAKER:` 头部解析改为模型只输出纯文字 → 仍通过 `<think>` 泄漏思考
- ❌ 示例中不包含 `<think>` → 模型依然输出

### 问题 2：后端阶段逻辑未完整验证

- `_USER_TURN_THRESHOLD = 5` → Part B 过渡尚未在真实场景中跑通
- `_examiner_has_spoken()` → Part B 用户回答后的收尾到 feedback 也未测试
- `_build_examiner_feedback_prompt` 未被调用过

### 问题 3：前端部分路径未完整验证

- Feedback stage 渲染逻辑（`pronunciation_delivery` 锁定图标等）未经测试
- Error stage → "Back to settings" → 重新开始流程未充分验证

## 7. 建议解决方向

### 方向 A：后端剥离 `<think>` 标签（快速修复）
在 `_stream()` 函数中，转发前过滤掉 `<think>...</think>` 及之间的全部内容：

```python
# 在 yield chunk 之前
cleaned = re.sub(r'<think>.*?</think>', '', chunk, flags=re.DOTALL)
if cleaned:
    yield json.dumps({"type": "chunk", "content": cleaned}) + "\n"
```

但注意：
- `<think>` 和 `</think>` 可能分属于不同 chunk（streaming 分段问题）
- 需要缓冲区匹配 `<think>` 开启后累积到 `</think>` 出现才转发

### 方向 B：切换模型
当前模型 deepseek-v4-flash 有内置 chain-of-thought。换用不支持或不启用 CoT 的模型可从根本上解决问题。

### 方向 C：在 LLM service 层关闭 CoT
检查 `deeptutor.services.llm` 的 API 参数，查找是否有关闭思考的参数（如 `thinking: false` 或 `extended_thinking: false`）。

### 方向 D：后处理缓冲区剥离
在后端维持一个流式缓冲区，累积所有 chunk。流结束后做统一后处理（剥离 `<think>` 标签），再一次性推给前端。但这样会牺牲流式体验。

## 8. Prompts 设计

### System Prompt（当前）
```
You help simulate an HKDSE English Paper 4 group discussion (Speaking).

You will be told which role to play in each turn.
Your output must be ONLY that person's spoken words — no labels, no headers,
no explanations, no formatting.

RULES:
- Speak naturally, 30-70 words, like a real conversation.
- Do NOT output anything except the speech itself.
- Do NOT say 'Candidate A:', 'I think', or similar framing.
```

### Part A User Prompt（当前）
```
CONTEXT:
Discussion Topic: Selfies
Background Article: ...

You are Candidate A — a confident and direct student.
This is the very first turn. Open the discussion by introducing the topic
and sharing your first opinion. Speak naturally in 30-70 words.
```

### Examiner Prompt（当前）
```
You are an HKDSE English examiner. Based on the group discussion above,
ask the student ONE follow-up question about the topic.
The question should invite the student to explain their personal views
or experiences. Output ONLY the question text — no labels, no greetings,
no commentary.
```

## 9. 后端轮动参数

```python
_USER_TURN_THRESHOLD = 5    # 用户发言≥5次后进入Part B
_MAX_CONSECUTIVE_AI = 2     # AI连续发言最多2次

# 加权随机选发言人
# 权重逻辑：max_count - count + 1，发言越少权重越高
# 排除上次发言的AI（不连续同一个人）
```

`oral_topics.json` 中的24个话题来自2019 DSE Paper 4（OCR + LLM清洗），按分类：
- social_issues: 13
- technology: 8
- education: 2
- environment: 1
