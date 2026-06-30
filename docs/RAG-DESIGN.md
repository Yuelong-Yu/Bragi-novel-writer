# RAG-DESIGN.md — 前后文一致性的 RAG 架构

本文档是 Bragi 自洽性子系统的最终设计冻结版，由 11 轮 grill-me 评审产出。所有架构决策已锁定；本文档之后的工作是落地实现。

## 目标

- 长篇小说（100+ 章）前后事实自洽：角色状态、物品位置、事件因果、知识传播、伏笔兑现都不矛盾
- 不引入任何新外部 API key（embedding / rerank 服务）
- 保持 Bragi 现有"agent 是 Markdown prompt，orchestrator 是 Claude Code 会话，文件即真相"的哲学

## 非目标

- 语义级别召回（"找思想/氛围相近的片段"）——暂砍，未来可加 embedding 列扩展
- 自动级联回滚已封板章节——人为危险，拒绝
- 组织/派系（organization）追踪——schema 预留，MVP 不实现
- 招式 / 概念 / 法术等 narrative concept 追踪（E4）——过早工程

## 架构总览

```
                          ┌─────────────────┐
                          │   brief.yaml    │
                          └────────┬────────┘
                                   ▼
                       ┌───── L0 → L1 → L2 ─────┐
                       │  (pin 进 prompt，不入索引)
                       └────────┬────────────────┘
                                ▼
                          L3 chapter outline ────────┐
                                │                    │
                                ▼                    │
   ┌────── for each chapter ────────────┐            │
   │                                    │            │
   │  ┌─[Python] build context pack ◀───┼────────────┘ S2 索引
   │  │  4 层: state + sidecar + prose + warnings    
   │  ▼                                                
   │  Writer (reads prose pack)                       
   │  outputs: ch{NNN}.md + ch{NNN}.events.yaml       
   │  │                                                
   │  ▼                                                
   │  [Python] V2.5 validator                         
   │  │  evidence 反查 prose                          
   │  │                                                
   │  ▼ FAIL → 回炉 Writer                            
   │  Writing Critic                                  
   │  │                                                
   │  ▼ FAIL → 回炉 Writer                            
   │  ▼ PASS                                          
   │  [Python] Ingest into SQLite ─────────► index.db │
   │                                                  │
   └──────────────────────────────────────────────────┘
                                │
                                ▼
                          (下一章重复)

显式命令（非自动）：
   bragi sync  →  扫描 SHA 差异 → 重 ingest + 软标记下游 stale
```

## 11 项核心决议

| # | 决议 | 选择 | 原因（一句话） |
|---|---|---|---|
| 1 | Fork 范围 | A1 — 只 fork 数据层（~8–10 文件） | 保留 Bragi 差异化（双 loop / 6 轴 / 分形张力 rubric），不被 webnovel-writer 淹没 |
| 2 | RAG 栈 | BM25(FTS5) + 实体图谱 + Orchestrator 文件式 rerank | 砍向量避新 API key；rerank 复用 orchestrator 当前 LLM，无新外部依赖；硬一致性 90%+ 已验证 |
| 3 | Python 调用模型 | α — Orchestrator-only | agent 保持纯 prompt；Python 输出物是文件，agent 读 |
| 4 | 索引内容 | S2 — L4 + L3 | L0/L1/L2 小，直接 pin；feedback 是过程产物，不入索引 |
| 5 | 实体数据源 | B1-β — Writer 自报 sidecar | 不引入新 agent；接受自报风险（V2.5 缓解） |
| 6 | sidecar 审计 | V2.5 — Python evidence 反查 | 强制 Writer 引用 prose 原文作 evidence；反查存在即过 |
| 7 | Context pack | P3+ — 4 层 | 当前状态 + sidecar 笔触摘录 + prose 召回 + 主动矛盾预警 |
| 8 | 章节切分 | K2 — 启发式 | 不污染 prose 文件；用空行/分隔符/转场词识别场景。**硬分隔符识别必须先 strip markdown emphasis**（`**`/`_`/`*`），否则会漏识别 `**---**` 类场景标记（demo 实测：漏识别 47 个场景，~13%） |
| 9 | 实体类型 | character + location + item + 独立 events 表 | 覆盖硬一致性 95%；事件因果通过 events 表追踪 |
| 10 | Ingestion 时机 | T2 — Critic PASS 后 | DB 永远只含封板事实，零 rollback |
| 11 | 文件修改同步 | R-final + C2 | 显式 `bragi sync` 触发；下游 stale 软标记不静默破坏 |

---

## SQLite Schema (`index.db`)

存储位置：`output/《项目》/index/index.db`

```sql
-- ============== 元数据 ==============

CREATE TABLE schema_meta (
    key TEXT PRIMARY KEY,
    value TEXT
);
-- key: schema_version (e.g. "1.0"), embedding_model (reserved), created_at

CREATE TABLE chapters (
    chapter INTEGER PRIMARY KEY,
    volume INTEGER NOT NULL,
    title TEXT,
    word_count INTEGER,
    ingested_at TIMESTAMP,
    ingested_hash_prose TEXT,        -- SHA256 of manuscript/vol{V}/ch{NNN}.md
    ingested_hash_sidecar TEXT,      -- SHA256 of manuscript/vol{V}/ch{NNN}.events.yaml
    ingested_hash_outline TEXT,      -- SHA256 of outline/L3-chapters/vol{V}-ch{NNN}.md
    stale_due_to TEXT,                -- JSON list, e.g. '["ch037"]'
    scene_marker_quality TEXT         -- 'clean' | 'degraded'
);

CREATE TABLE scenes (
    chapter INTEGER,
    scene_index INTEGER,
    start_line INTEGER,
    end_line INTEGER,
    char_count INTEGER,
    PRIMARY KEY (chapter, scene_index)
);

-- ============== 全文索引 (FTS5) ==============

CREATE VIRTUAL TABLE chunks USING fts5(
    chunk_id UNINDEXED,
    chapter UNINDEXED,
    scene_index UNINDEXED,
    source_type UNINDEXED,           -- 'prose' | 'outline_l3'
    content,
    tokenize = 'unicode61'           -- 中文用 jieba 预切词后存
);

-- ============== 实体图谱 ==============

CREATE TABLE entities (
    entity_id TEXT PRIMARY KEY,       -- '林月', 'L001-云栖宗主峰', 'I001-青锋剑'
    entity_type TEXT NOT NULL,        -- 'character' | 'location' | 'item' | 'organization'(reserved)
    canonical_name TEXT NOT NULL,
    introduced_chapter INTEGER,       -- 首次出现章节
    profile_json TEXT,                -- 6轴等结构化数据 (character only)
    metadata_json TEXT                -- 其他扩展字段
);

CREATE TABLE aliases (
    alias TEXT,
    entity_id TEXT,
    entity_type TEXT,
    confidence REAL DEFAULT 1.0,
    registered_chapter INTEGER,
    PRIMARY KEY (alias, entity_id)
);
CREATE INDEX idx_aliases_entity ON aliases(entity_id);

CREATE TABLE appearances (
    entity_id TEXT,
    chapter INTEGER,
    scene_index INTEGER,
    role TEXT,                        -- 'pov' | 'major' | 'mentioned' | 'reference'
    PRIMARY KEY (entity_id, chapter, scene_index)
);
CREATE INDEX idx_appearances_chapter ON appearances(chapter);

CREATE TABLE state_changes (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    entity_id TEXT NOT NULL,
    chapter INTEGER NOT NULL,
    scene_index INTEGER,
    field TEXT NOT NULL,              -- '伤势' | '位置' | '灵气' | '所有者' (item) | ...
    value TEXT,                       -- 'low' / structured string
    delta TEXT,                       -- optional, '+30%' etc.
    severity TEXT,                    -- 'low' | 'medium' | 'high' | 'fatal'
    evidence TEXT NOT NULL,           -- 必填，prose 原文短句
    triggered_by_event_id TEXT,       -- 可选，关联 events
    FOREIGN KEY (entity_id) REFERENCES entities(entity_id)
);
CREATE INDEX idx_state_entity ON state_changes(entity_id, chapter);

CREATE TABLE relationships (
    from_entity TEXT,
    to_entity TEXT,
    field TEXT,                       -- 'trust' | 'enmity' | 'romance' | 'owner_of' (item)
    chapter INTEGER,
    value TEXT,
    delta TEXT,
    evidence TEXT NOT NULL,
    PRIMARY KEY (from_entity, to_entity, field, chapter)
);

-- ============== 事件 ==============

CREATE TABLE events (
    event_id TEXT PRIMARY KEY,        -- 'E001', 'E002'
    name TEXT NOT NULL,
    aliases_json TEXT,                -- '["那一夜", "山门血夜"]'
    chapter INTEGER NOT NULL,
    scene_index INTEGER,
    event_type TEXT,                  -- 'death' | 'battle' | 'revelation' | 'promise_fulfilled' | ...
    significance TEXT NOT NULL,       -- 'low' | 'medium' | 'high' | 'pivotal'
    summary TEXT,
    evidence TEXT NOT NULL
);
CREATE INDEX idx_events_chapter ON events(chapter);

CREATE TABLE event_participants (
    event_id TEXT,
    entity_id TEXT,
    role TEXT,                        -- 'victim' | 'perpetrator' | 'witness' | 'beneficiary' | ...
    PRIMARY KEY (event_id, entity_id)
);

-- ============== 知识传播 ==============

CREATE TABLE knowledge_state (
    entity_id TEXT,                   -- 哪个角色
    event_id TEXT,                    -- 知道了哪个事件
    chapter_learned INTEGER,          -- 在哪一章习得
    learned_via TEXT,                 -- 'witnessed' | 'told_by:<entity>' | 'inferred' | 'document'
    evidence TEXT NOT NULL,
    PRIMARY KEY (entity_id, event_id)
);

-- ============== 伏笔 ==============

CREATE TABLE promises (
    promise_id TEXT PRIMARY KEY,      -- 'P001'
    chapter_set INTEGER NOT NULL,
    text TEXT NOT NULL,               -- "三日后山门相见"
    deadline_chapter INTEGER,
    status TEXT NOT NULL,             -- 'open' | 'fulfilled' | 'broken' | 'forgotten'
    fulfilled_chapter INTEGER,
    fulfilled_event_id TEXT,          -- 关联 events
    evidence_set TEXT NOT NULL,
    evidence_fulfilled TEXT
);
CREATE INDEX idx_promises_status ON promises(status);

-- ============== 修改同步标记（R-final + C2） ==============

CREATE TABLE stale_marks (
    chapter INTEGER PRIMARY KEY,
    stale_due_to TEXT NOT NULL,       -- JSON list of upstream chapter numbers
    marked_at TIMESTAMP,
    resolved_at TIMESTAMP
);
```

---

## Sidecar Schema (`manuscript/vol{V}/ch{NNN}.events.yaml`)

```yaml
# 章节元数据
chapter: 42
volume: 1
word_count: 3120
scene_count: 4

# 本章新注册的实体（任何 sidecar 引用的实体必须先在此处或 world/* 中注册过）
register_entities:
  - id: I017-寒霜匕首
    entity_type: item
    aliases: [那柄短匕, 寒霜]
    profile:
      origin: 反派遗物
      material: 玄铁
      effects: 见血封喉

# 本章发生的关键事件
events:
  - id: E007
    name: 山门夜战
    aliases: [那一夜, 血色山门]
    event_type: battle
    significance: pivotal
    scene_index: 3
    summary: 反派在山门突袭，林月受重伤，陈峰目睹全程
    evidence: "剑锋斜斜划过左腿，血一下涌出来"
    participants:
      - { entity: 林月, role: victim }
      - { entity: 反派, role: perpetrator }
      - { entity: 陈峰, role: witness }

# 实体状态变更（每条都必须带 evidence）
state_changes:
  - entity: 林月
    field: 伤势
    to: 左腿剑伤
    severity: high
    scene_index: 3
    evidence: "剑锋斜斜划过左腿"
    triggered_by_event_id: E007
  - entity: 林月
    field: 灵气
    delta: "-30%"
    severity: medium
    scene_index: 3
    evidence: "她半跪在地，丹田中那一团火气几乎熄灭"
  - entity: I017-寒霜匕首
    field: 所有者
    to: 主角
    scene_index: 4
    evidence: "他俯身捡起那柄短匕"

# 关系变更
relationships:
  - from: 林月
    to: 陈峰
    field: trust
    delta: "+1"
    chapter: 42
    evidence: "她终于松开了剑，背对着他坐下"

# 知识传播（谁知道了哪个事件）
knowledge_state:
  - { entity: 陈峰, learns_event: E007, learned_via: witnessed, evidence: "他眼睁睁看着" }
  - { entity: 林月, learns_event: E007, learned_via: witnessed, evidence: "她记住了那道剑光" }
  # 主角不在场，不写入 → 后续章节如出现"主角得知 E007"必须显式 learned_via: told_by:陈峰 或类似

# 伏笔
promises:
  new:
    - id: P017
      text: 三日后山门相见
      deadline_chapter: 45
      evidence_set: "他低声说，我们三日后山门见"
  fulfilled:
    - id: P012
      fulfilled_event_id: E007
      evidence_fulfilled: "复仇之刃终于落下"

# 出场清单（不重要但便于快速过滤）
appearances:
  - { entity: 林月, scene_index: 1, role: pov }
  - { entity: 林月, scene_index: 3, role: pov }
  - { entity: 陈峰, scene_index: 3, role: major }
  - { entity: 反派, scene_index: 3, role: major }

# 对 P3+ 矛盾预警的响应（若 context-pack 有 warning，必须回应）
addressed_warnings:
  - warning_id: W001-ch041-林月伤势
    chosen: c
    rationale: prose 中描写林月运起聚灵诀强忍剧痛挥剑
    new_state_changes_added: [林月.灵气]
```

---

## Pipeline 集成点（`skills/write-novel.md` 修改）

```
Phase 0 (Initialize)
  ...原有步骤...
  [新增] 创建 output/《项目》/index/index.db 空库 + 应用 schema
  [新增] 应用 schema_meta: schema_version="1.0"

Phase 1 (L0)
  ...原有步骤...
  人类 checkpoint 通过后：
  [新增] python -m bragi.context bootstrap_entities
         读 world/characters.md, world/locations.md, world/items.md
         一次性 ingest 到 entities + aliases 表

Phase 2-3 (L1-L2)
  ...原有步骤...
  L1/L2 不入索引（S2 规定）

Phase 5 (L3)
  每个 L3 章纲 PASS 后：
  [新增] python -m bragi.context ingest_outline --chapter=N
         切 chunk 入 chunks FTS5 表 (source_type='outline_l3')

Phase 6 (L4 Prose, 每章)
  对每章:
    [新增] python -m bragi.context build_pack --chapter=N
           → 生成 context/ch{NNN}-pack.md (4 层)
    Writer reads context-pack 作为额外输入
    Writer outputs manuscript/vol{V}/ch{NNN}.md + ch{NNN}.events.yaml
    [新增] python -m bragi.context validate --chapter=N
           → V2.5 evidence 反查
           → FAIL → 回炉 Writer (与 Critic 失败合并到 max_prose_rounds)
    Writing Critic reads chapter
    Critic FAIL → 回炉 Writer
    Critic PASS:
    [新增] python -m bragi.context ingest_chapter --chapter=N
           → 切 prose chunk 入 FTS5
           → 解析 sidecar 写入所有实体/事件/伏笔/知识表
           → 更新 chapters 表的 ingested_hash_*

Phase 7 (Finalize) - 不变
```

---

## Context Pack 格式（P3+ 的 4 层）

由 `bragi.context build_pack --chapter=N` 生成，文件路径 `context/ch{NNN}-pack.md`。

```markdown
# Chapter {NNN} — Context Pack

## 第一层：当前世界状态（SQLite 聚合）

### 涉及实体（从 L3 大纲提取）
- 林月 [character | 伤势=左腿剑伤(high) | 灵气=-30% | 位置=云栖宗主峰]
- 陈峰 [character | 伤势=轻伤右臂 | 位置=云栖宗主峰]
- 反派 [character | 真容=未露 | 位置=未知]
- 云栖宗主峰 [location | 状态=完好 | 防御=削弱(ch041)]
- 青锋剑 [item | 所有者=林月 | 状态=已损 | 位置=主峰演武场]

### 未兑现伏笔（按 deadline 排序）
- P017 [deadline ch45] "三日后山门相见" (set in ch41)
- P012 [deadline ch60] "复仇之约" (set in ch08)

## 第二层：相关 Sidecar 原文摘录（保留 evidence 笔触）

### 林月相关历史 sidecar (top 5, ranked by severity×recency)

**ch041 §3 - state_change(伤势)** 
- evidence: "剑锋斜斜划过左腿，血一下涌出来"

**ch041 §3 - state_change(灵气)**
- evidence: "她半跪在地，丹田中那一团火气几乎熄灭"

**ch038 §2 - knowledge_gained(陈峰真实修为)**
- evidence: "他指尖一转，那道剑气竟在空中折出 360 度——林月瞳孔一缩"

[...]

### 陈峰相关历史 sidecar
[...]

## 第三层：相关 Prose 召回片段（BM25 → Orchestrator rerank）

> **职责边界（demo 实测确认）**：第三层只负责"内容场景"召回。
> - ✅ 适合：「林月最近的伤势状态如何描写」「主角和反派的对峙是怎么写的」
> - ❌ 不适合：「X 在哪一章发生」类时间锚点查询——这种应优先走第一层（events / state_changes SQL）。
>   原因：BM25 OR-召回受高频 token（主角名）淹没，章节标题与内容字面不一致时（如标题"婚后"但内容不包含"结婚"），rerank 也无法救回——候选集里根本没有正确章节。

### Beat 1: 林月与陈峰商议反击
[ch041 §4] "她抬起头，目光里第一次没有戒备：'明日……我们该如何？'..."
[ch038 §1] "陈峰将剑横在膝上，低声道：'有些路只能一个人走。'..."

### Beat 2: 主角追查反派踪迹
[ch033 §1] "黑袍人留下的字条只有四个字：'风雪夜归。'..."
[ch028 §3] "李铮回头深深看了主角一眼，那目光中..."

[...]

## 第四层：⚠️ 潜在矛盾预警

### [CRITICAL] W001-林月-伤势冲突
**冲突点**：Beat 3 要求"林月挥剑斩出"，但 ch041 state_change 记录左腿剑伤 severity=high。
**Writer 需在 sidecar `addressed_warnings` 中处理**，选项：
- (a) 修改大纲——跳过此 beat 或换执行者
- (b) 伤势已愈合——prose 中描写治疗过程，sidecar 新增 state_change(林月, 伤势, 已愈合)
- (c) Exception 触发——prose 中说明如何强撑（灵气运转/止血丹），sidecar 新增对应代价 state_change

### [HIGH] W002-陈峰-axis exception 检查
**冲突点**：Beat 4 要求"陈峰主动出手相助"，但其 conflict_response baseline=8（回避退让）。
**Writer 选项**：触发 exception ("面对林月时切换为对抗")，prose 必须显化心理转折，sidecar 标注 axis_exception_triggered。

### [INFO] W003-伏笔密度提醒
P017 deadline=ch45，本章 ch42，余 3 章。建议本章为兑现做铺垫。
```

---

## 矛盾扫描规则（Python，规则法）

`bragi.context.warning_scanner` 模块。无 LLM 调用。每条规则独立可测。

```
Rule R1: state_field_conflict
  对每个 entity-field 组合，从 state_changes 取最新值
  如果 L3 大纲提到该 entity 执行了与最新状态冲突的动作 → CRITICAL
  冲突表（可扩展）：
    伤势=high + 动作=挥剑/突进/搏斗 → 冲突
    位置=A + 动作=在B出现 → 冲突
    state=死亡 + 任何动作 → 冲突

Rule R2: axis_baseline_violation
  对每个 character，比对 6 轴 baseline 与大纲要求的行为
  如 conflict_response=8 (回避) + 大纲要求"主动出手" → HIGH（除非显式声明 exception）

Rule R3: knowledge_gap
  大纲要求角色 A 引用事件 E007
  查 knowledge_state where entity=A and event=E007
  不存在 → CRITICAL (A 不可能知道 E007)

Rule R4: promise_deadline
  open promises with deadline_chapter == current_chapter ± 3
  → INFO（伏笔逼近，建议关注）

Rule R5: item_ownership_conflict
  大纲提到 item X 被 entity A 使用
  state_changes 中 X 的最新所有者 ≠ A → HIGH

Rule R6: organization_status（reserved）
  predefined but no data in MVP

Rule R7: stale_reference (来自 C2)
  本章查询命中 stale 标记的下游 chunks
  → HIGH 警告 prefix
```

---

## V2.5 Validator 逻辑

`bragi.context.validator` 模块。无 LLM 调用。

```
def validate(sidecar_yaml, prose_text, chapter):
    # 1. YAML schema 合规
    assert_valid_schema(sidecar_yaml)
    
    # 2. evidence 反查
    for sc in sidecar_yaml.state_changes:
        if not fuzzy_match(sc.evidence, prose_text, threshold=0.85):
            return FAIL(f"state_change evidence not found in prose: {sc.evidence}")
    for ev in sidecar_yaml.events:
        if not fuzzy_match(ev.evidence, prose_text, threshold=0.85):
            return FAIL(f"event evidence not found in prose")
    # (同样校验 relationships, knowledge_state, promises)
    
    # 3. 实体引用合法性
    known_entities = set(db.entities.ids) | set(sidecar.register_entities.ids)
    for sc in sidecar_yaml.state_changes:
        if sc.entity not in known_entities:
            return FAIL(f"unknown entity: {sc.entity}, must register via register_entities")
    
    # 4. promise 引用合法性
    open_promises = set(db.promises.where(status=open).ids)
    for pid in sidecar_yaml.promises.fulfilled:
        if pid.id not in open_promises:
            return FAIL(f"fulfilled promise not open: {pid.id}")
    
    # 5. addressed_warnings 完整性
    pack_warnings = parse_warnings_from_pack(f"context/ch{chapter:03d}-pack.md")
    critical_warnings = [w for w in pack_warnings if w.severity == "CRITICAL"]
    addressed_ids = set(w.warning_id for w in sidecar_yaml.addressed_warnings)
    missing = [w.id for w in critical_warnings if w.id not in addressed_ids]
    if missing:
        return FAIL(f"unaddressed CRITICAL warnings: {missing}")
    
    return PASS
```

---

## Orchestrator Rerank 协议（文件式）

经 demo 验证（90% recall@5），取代原计划的 Claude Haiku via SDK。**零新 API key**，复用 orchestrator 本身的 LLM 实例。

### 协议三步

```
┌─────────────────────────────────────────────────────────┐
│ Step 1: Python 写候选文件                                │
│   context/_rerank_pending/{chapter}-{beat_id}.md         │
│   ├── Query 文本                                          │
│   ├── Top-K requested                                     │
│   └── Candidates 列表 (BM25 top-50, 每条截 300 字)        │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│ Step 2: Orchestrator 读文件 + 输出 result              │
│   context/_rerank_result/{chapter}-{beat_id}.txt        │
│   └── 空格分隔的 top-K 候选编号，best → worst             │
│   （由 skills/write-novel.md 在 P3+ 构建阶段调度）      │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│ Step 3: Python 读 result，组装 context-pack 第三层      │
└─────────────────────────────────────────────────────────┘
```

### 候选文件格式（实测可用）

```markdown
# Rerank request — ch042-beat3

**Query:** 林月相关的最近伤势与战斗状态
**Top-K requested:** 5

## Candidates (BM25 top 50)

### [0] ch041-s03  (BM25 score: -7.10)
（300 字截断）...

### [1] ch038-s02  (BM25 score: -6.11)
...

...

### [49] ch023-s07  (BM25 score: -1.48)
...

---

## Instructions for orchestrator

Pick the top 5 candidates most relevant to the query above (semantic relevance, not literal match).
Write the result to `context/_rerank_result/ch042-beat3.txt`
Format: space-separated indices, ranked best to worst.
Example: `12 3 47 8 0`
```

### Result 文件格式

```
43 46 40 10 8
```

### 在 skills/write-novel.md 中的调度

```
Phase 6 (per chapter):
  ...
  [Python] python -m bragi.context build_pack_step1 --chapter=N
    → 生成 context/_rerank_pending/ch{N}-beat*.md
    → 输出 "rerank requests written, pending orchestrator"
  
  [Orchestrator] 读取每个 *.md，做语义 rerank
    → 写 context/_rerank_result/ch{N}-beat*.txt
  
  [Python] python -m bragi.context build_pack_step2 --chapter=N
    → 读 result 文件，组装最终 context/ch{N}-pack.md (P3+ 4 层)
  
  Writer 读 pack 进入写作
```

### 设计回顾

| 维度 | 选择 | 理由 |
|---|---|---|
| 触发机制 | Python 写候选文件 | 与 Bragi "文件即接口" 哲学一致 |
| 执行者 | Orchestrator（运行 skill 的 Claude 本体） | 零新 API key；rerank 与生成共用 LLM；可解释 |
| 通信媒介 | 文件 | 可审计、可重放、可单元测试 |
| 模型选择 | brief.yaml 里 orchestrator 用什么就用什么 | 用户已有的预算/模型组合直接复用 |
| Demo 实测 | 70% (BM25) → 90% (BM25+rerank) recall@5 | 见 `demo/findings.md` |

---

## `bragi sync` 命令（R-final + C2）

`bragi.cli.sync` 入口。用户显式触发。

```
$ bragi sync

[1/3] Scanning files for changes (SHA256-based)...
  ✓ world/characters.md         hash matches DB
  ! world/characters.md         hash CHANGED (was 4f3a... now 9b7c...)
  ! manuscript/vol1/ch037.md    hash CHANGED
  ! manuscript/vol1/ch037.events.yaml  hash CHANGED
  ✓ manuscript/vol1/ch038.md    hash matches DB
  ✓ ... (扫描 N 文件)

[2/3] Re-ingesting changed files...
  ✓ world/characters.md         → entities/aliases tables refreshed
  ✓ manuscript/vol1/ch037.md    → 删除旧 chunks，重新切分 + 入库
  ✓ manuscript/vol1/ch037.events.yaml → 删除旧 state_changes/events/...，重 ingest

[3/3] Marking downstream chapters as stale (C2)...
  Reference analysis shows ch037 is referenced by:
    ch038 (references state_change(林月.伤势), event E007)
    ch041 (references promise P017)
    ch043 (references event E007)
  ✓ ch038, ch041, ch043 → stale_marks 表添加 entry: stale_due_to=["ch037"]

Sync complete. 3 chapters re-ingested; 3 chapters marked stale.
Stale chapters will surface warnings in future context-packs.
Run `bragi unstale ch038 ch041 ch043` after manually verifying.
```

附属命令：
- `bragi unstale ch038 ...` — 用户手动确认后清除 stale 标记
- `bragi validate-consistency --range ch001-ch042` — 跑全部 R1-R7 规则做回顾性矛盾扫描

---

## Chapter Expansion 模式（针对已入库章节的扩写/改写）

**场景**：用户对已入库的全本小说中某一章不满意（太短、骨架感太强、缺细节），想"扩写"。
现有 `mode: expansion` 是面向**整本**短稿膨胀为长篇的，不适合单章局部加工。
新增 `mode: chapter_expansion` 解决此场景。

### E2 / E3 自动切换规则

| 增长倍数（target_chars / original_chars） | 模式 | 行为 |
|---|---|---|
| ≤ 2.0 | **E2 增量丰富** | sidecar 必须是原 sidecar 的"超集"；不准移除/重命名 events / promises；新增的 state_changes 不能与下游冲突 |
| > 2.0 | **E3 重构** | 允许 events 改写（end-state 不变即可）；允许 promise id 迁移（须提供 migration map）；下游章节自动 stale C2 软标记 |

阈值 2.0 来自经验：作者意图在 ≤2x 是"补血加肉"，>2x 必然要重构结构。

### Pipeline

```
/expand-chapter v1-ch042 --target-chars=5000
         │
         ▼
[Python] downstream_constraints.py --chapter=42 --target-chars=5000
  ├── 查 DB 编译"必须保留"清单
  └── 输出 constraints_ch042.md + 推荐模式 (E2 / E3)
         │
         ▼
[Python] expansion_brief_builder.py --chapter=42 --mode=E2
  ├── 加载原 ch042.md + ch042.events.yaml
  ├── 加载 P3+ 上游 pack（ch1..41 标准）
  ├── 加载 constraints_ch042.md
  └── 输出 expansion_brief_ch042.md
         │
         ▼
Writer 读 brief，输出新 ch042.md + 新 ch042.events.yaml
         │
         ▼
[Python] V2.5 evidence 反查（原有）
[Python] expansion_validator.py（新增）：
  ├── 模式 E2: sidecar 必须是超集（每个原 event_id / promise_id 都在；end-state 一致）
  └── 模式 E3: 允许结构变化但触发下游 stale 评估
         │
         ▼
Critic 评分（新增 1 维: 「扩写自洽性」）
         │
         ▼
[Python] 替换 DB 中本章 chunks + sidecar 衍生表
         │
         ▼ E3 路径
[Python] 给下游章节加 stale 标记（同 R-final + C2 机制）
```

### Downstream Constraints 输出格式（见 `runtime/downstream_constraints.py`）

给定章节 N，编译 5 类不变量：

```markdown
# Downstream Constraints — ch042

## 🔒 必须保留的事件
- E007（被 ks:陈峰@ch45 引用）

## 🔒 必须保留的实体
- 林月 (character) — 后续出场: ch43, ch45, ch48, ...

## 🔒 必须保留的伏笔
- P017 [open, deadline ch45]
- P012 [fulfilled@ch44 by E007]

## 🔒 必须保留的状态终值
- 林月.伤势 = 左腿剑伤 (下游在 ch44 修改)

## 🔒 必须保留的知识传播锚点
- 陈峰 在 ch45 学到 E007
```

每条都有 `entity_id / event_id / promise_id` 作为机器可校验的锚点。

### Expansion Validator 校验规则（新增模块）

E2 模式：
- 原 events.event_id 必须全部在新 sidecar 中存在
- 每个 event 的 participants 集合不变（可以增加 minor，但不能减少）
- 原 promises.promise_id 必须全部在；text / deadline_chapter / status 不变
- 新增 state_changes 不能让 downstream 的 reference 失效（通过查 chains_locked 校验）
- 新增 register_entities 不能与 entities 表已有 id 冲突

E3 模式：
- 上述约束放宽——允许 event_id 重命名（必须在 sidecar `event_migrations` 字段声明）
- promise id 改名同理 (`promise_migrations`)
- 下游有 reference 的 chunks 自动标记 stale，需用户后续显式 `bragi unstale` 清除

### Sidecar 新增字段（仅 E3 用）

```yaml
expansion_meta:
  mode: E3                          # E2 时不写或写 E2
  original_word_count: 2583
  new_word_count: 6200
  growth_ratio: 2.40

event_migrations:                   # 仅 E3 模式
  - from_id: E001-old
    to_id: E001
    rationale: "重构: 把'闯入'细分为'入楼+突破闸机'两个 sub-event"

promise_migrations:                 # 仅 E3 模式
  - from_id: P017-old
    to_id: P017
    rationale: "措辞调整，含义不变"

stale_downstream:                   # E3 触发的级联清单
  - chapter: 43
    reason: "ch43.events.yaml 引用了 ch42 旧 E007，新版本 E007 含义微调"
    severity: low
```

### 落地组件

| 组件 | 状态（demo） | 说明 |
|---|---|---|
| `runtime/downstream_constraints.py` | ✅ 已实现 | 跑 Octopus ch1 输出 14 个 invariants |
| `runtime/expansion_brief_builder.py` | ⏳ TODO | 拼接 brief markdown |
| `runtime/expansion_validator.py` | ⏳ TODO | E2/E3 校验 |
| `runtime/expansion_ingest.py` | ⏳ TODO | 替换 DB 中本章数据 + E3 stale 级联 |
| `skills/expand-chapter.md` | ⏳ TODO | 用户入口 skill |
| Writer prompt 模式分支 | ⏳ TODO | 区分 chapter_expansion vs 其他模式 |
| Critic 第 9 维「扩写自洽性」 | ⏳ TODO | 仅在 chapter_expansion 模式启用 |

### MVP 增量时间盒

| 任务 | 估计 |
|---|---|
| downstream_constraints.py | ✅ 0.5 天（已完成） |
| expansion_brief_builder.py | 0.5 天 |
| expansion_validator.py (E2) | 0.5 天 |
| expansion_validator.py (E3) + stale 级联 | 1 天 |
| skills/expand-chapter.md + Writer prompt | 0.5 天 |
| Critic 第 9 维 | 0.5 天 |
| Octopus 端到端 demo | 0.5 天 |
| **小计** | **3 天**（在原 MVP 10 天之外） |

---

## Fork 清单（从 webnovel-writer 取的 8–10 个文件）

放到 `bragi/runtime/bragi_context/`：

| 源文件（webnovel-writer） | Bragi 路径 | 改造点 |
|---|---|---|
| `scripts/data_modules/index_manager.py` | `runtime/bragi_context/index_manager.py` | 删除 vector 相关方法；schema 改成本文档版本 |
| `scripts/data_modules/index_chapter_mixin.py` | 同名 | 适配 Bragi 章节结构 |
| `scripts/data_modules/index_entity_mixin.py` | 同名 | 扩展 entity_type 4 类（含 reserved org） |
| `scripts/data_modules/entity_linker.py` | 同名 | 别名消歧，沿用 |
| `scripts/data_modules/query_router.py` | 同名 | 删除 vector 路由分支 |
| `scripts/data_modules/api_client.py` | 改名 `claude_haiku_client.py` | 大幅瘦身，只保留 Claude API rerank 调用 |
| `scripts/data_modules/config.py` | 同名 | 删除 EMBED_/RERANK_ env vars，只保留 Bragi 配置 |
| `scripts/data_modules/observability.py` | 同名 | 沿用 |
| `scripts/security_utils.py` | `runtime/bragi_context/utils.py` | 提取需要的 atomic_write_json 等 |
| `scripts/runtime_compat.py` | 同名 | Windows utf-8 支持，沿用 |

Bragi 新写的文件（不 fork）：

| 文件 | 职责 |
|---|---|
| `runtime/bragi_context/scene_chunker.py` | K2 启发式切场景（80 行） |
| `runtime/bragi_context/sidecar_parser.py` | YAML 解析 + schema 校验 |
| `runtime/bragi_context/validator.py` | V2.5 validator |
| `runtime/bragi_context/warning_scanner.py` | R1-R7 矛盾规则 |
| `runtime/bragi_context/pack_builder.py` | P3+ 4 层 context-pack 生成 |
| `runtime/bragi_context/rerank_protocol.py` | 文件式 orchestrator rerank（写 candidates 文件，读 result 文件） |
| `runtime/bragi_context/cli.py` | `bragi.context` / `bragi sync` 命令入口 |
| `runtime/bragi_context/__main__.py` | python -m 入口 |
| `runtime/bragi_context/schema.sql` | SQLite DDL（本文档版本） |
| `runtime/pyproject.toml` | 依赖声明 |

---

## 依赖

```toml
# runtime/pyproject.toml
[project]
name = "bragi-context"
version = "0.1.0"
requires-python = ">=3.10"
dependencies = [
    "pyyaml>=6.0",
    "jieba>=0.42",          # 中文分词用于 FTS5
    "rapidfuzz>=3.0",       # V2.5 evidence 反查的 fuzzy match
    "click>=8.1",           # CLI
]
```

无 sentence-transformers、无 modal、无 anthropic、无 sqlalchemy。SQLite 是 stdlib。Rerank 不走 SDK，由 orchestrator 通过文件接口完成。

---

## `brief.yaml` 新增字段

```yaml
# === RAG / Context system ===
context:
  enabled: true                       # false 则全部跳过，回到原始 Bragi 行为
  rerank_protocol: "orchestrator-file"  # 由当前 orchestrator LLM 文件式接管
  rerank_top_k: 5                     # BM25 召回 top-50 → Orchestrator rerank → top-K
  pack_size:
    sidecar_excerpts_max: 30
    prose_chunks_max: 10
  scene_chunker:
    enforce_min_chars: 200
    enforce_max_chars: 1500
  warnings:
    treat_high_as_critical: false     # 严格模式开关
```

---

## state.yaml 新增字段

```yaml
context:
  schema_version: "1.0"
  bootstrap_done: false               # L0_confirmed 后置 true
  last_sync_at: ""                    # bragi sync 时刻
  stale_chapters: []                  # 从 stale_marks 表镜像
```

---

## 推迟项 / 未来工作

| 议题 | 触发条件 | 升级路径 |
|---|---|---|
| Embedding 向量补回 | 软一致性需求显现（主题/情感呼应频繁丢失） | chunks 表加 embedding BLOB 列；接入本地 BGE-M3 |
| Critic 加事实自报维度（V3） | V2.5 后实际仍有自我美化 sidecar | writing-critic.md 加 9 维 rubric，writing-rubric.md 重平衡 |
| Organizations 启用 | 题材需要派系/宗门追踪 | 实施 already-reserved 的 entity_type=organization |
| Concepts/招式追踪（E4） | 实际题材重度依赖此 | 新表 `narrative_concepts`，独立于 entities |
| MCP server 化（γ） | webnovel-writer 升级为 MCP，或 Bragi 团队需要工具调用模式 | 把 cli.py 暴露为 MCP server |
| Auto-cascade rewrite suggestion | 用户希望 `bragi sync` 更智能 | 加 LLM 调用建议下游章节的重写要点 |

---

## 验收标准（MVP）

- [ ] 100 章长度的 demo 项目从头跑完，无矛盾扫描 CRITICAL 漏报
- [ ] `bragi sync` 能正确识别 SHA 差异并 stale-mark 下游
- [ ] V2.5 误报率（合理 sidecar 被 FAIL）< 5%
- [ ] context-pack 体积控制在 3000 字以内（fits in Writer prompt）
- [ ] 全套 RAG 增加单章生成耗时 < 30s（含 orchestrator 文件式 rerank）
- [ ] 不需要任何新的 API key
- [ ] `context.enabled: false` 时全套 RAG 静默旁路，Bragi 行为退化为原 prompt-only 形态
