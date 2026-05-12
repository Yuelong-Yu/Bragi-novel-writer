# Skill: write-novel

Generate a complete web novel using the Bragi multi-agent pipeline.

## Usage

```
/write-novel [path-to-brief]
```

If no brief path is provided, looks for `brief.yaml` in the current directory.

## Pipeline Overview

```
Brief → Architect → Structure Critic ─→ (iterate until pass) ─→ Human Checkpoint
                                                                       │
  ┌────────────────────────────────────────────────────────────────────┘
  │
  ▼
Style Matcher → Human Checkpoint → Writer → Prose Critic ─→ (iterate until pass)
                                      │                            │
                                      └────── per chapter ─────────┘
                                                    │
                                                    ▼
                                              Final Manuscript
```

## Orchestration Steps

### Phase 0: Initialize

1. Read `brief.yaml` and validate required fields
2. Read `mode` field and validate mode-specific requirements:
   - `original`: No additional requirements
   - `adaptation`: `adaptation` block must be present with at least `source_work` or `adaptation_idea` filled → **error if missing**: "改写模式需要填写 adaptation 区块"
   - `expansion`: `existing_draft` must be provided → **error if missing**: "扩写模式需要提供 existing_draft"
3. Create output directory: `output/《{project_name}》/`
4. Create subdirectories: `world/`, `outline/`, `outline/L3-chapters/`, `manuscript/`, `feedback/`, `final/`
5. Copy brief to output directory
6. Initialize `state.yaml` from template (set `mode` from brief)
7. If existing materials provided, copy them to appropriate directories

### Phase 1: L0 — Story Core

**Agent**: Architect (`agents/architect.md`)

1. Set `state.yaml`: `current_phase: L0`, `structure.L0.status: in_progress`
2. Invoke Architect with brief + mode context:
   - **original**: Architect generates from scratch
   - **adaptation**: Architect also receives `adaptation` block (source_work, adaptation_idea, mapping_hints) as structural constraints
   - **expansion**: Architect first reads `existing_draft`, performs skeleton extraction (extract characters, world rules, plot threads, hooks), then writes world files based on extracted skeleton
3. Architect writes: `world/core.md`, `world/setting.md`, `world/characters.md`, `world/timeline.md`
4. Invoke Structure Critic (`agents/structure-critic.md`) with `rubrics/structure-rubric.md`
5. Critic writes: `feedback/structure-review-r{N}.md`
6. Check verdict:
   - **PASS** → update state, proceed to Human Checkpoint
   - **FAIL** → feed feedback back to Architect, increment round counter
   - **Max rounds reached** → set `status: human_flag`, pause for intervention
7. **HUMAN CHECKPOINT**: Pause. Present L0 summary. Wait for user confirmation.
   - User confirms → set `checkpoints.L0_confirmed: true`, proceed
   - User requests changes → feed changes to Architect as additional directives, re-enter loop

### Phase 2: L1 — Master Outline

**Agent**: Architect

1. Set `current_phase: L1`
2. Architect reads `world/` + brief → writes `outline/L1-master.md`
3. Structure Critic evaluates → `feedback/structure-review-r{N}.md`
4. Iterate until PASS or max rounds
5. **HUMAN CHECKPOINT**: Present master outline. Wait for confirmation.

### Phase 3: L2 — Volume Outlines

**Agent**: Architect

For each volume (1 to `brief.volumes`):
1. Set `current_phase: L2`, `current_volume: {N}`
2. Architect reads L1 + world → writes `outline/L2-vol{N}.md`
3. Structure Critic evaluates
4. Iterate until PASS or max rounds
5. **HUMAN CHECKPOINT**: Present volume outline. Wait for confirmation.

### Phase 4: Style Matching (if `auto_style_match: true`)

1. Set `current_phase: style_match`
2. Read `world/core.md`, `world/characters.md`, `outline/L1-master.md`, brief
3. Recommend a reference work that best matches the novel's genre, tone, and worldview
4. Present recommendation with rationale
5. **HUMAN CHECKPOINT**: User confirms or overrides reference work selection
6. Analyze the confirmed reference work's style characteristics:
   - Sentence structure patterns
   - Paragraph rhythm
   - Narrative distance
   - Vocabulary register
   - POV handling
   - Dialogue style
7. Write `world/style-guide.md` with extracted style parameters
8. Set `style.status: confirmed`

### Phase 5: L3 — Chapter Outlines

**Agent**: Architect

1. Set `current_phase: L3`
2. For each volume, for each chapter:
   - Architect reads L2-vol + world + previous L3s → writes `outline/L3-chapters/vol{N}-ch{NNN}.md`
3. Structure Critic evaluates L3 as a batch per volume
4. Iterate until PASS or max rounds
5. No human checkpoint at L3 (already confirmed direction at L2)

### Phase 6: L4 — Prose Generation

**Agent**: Writer (`agents/writer.md`)

For each chapter in order:
1. Set `current_phase: L4_writing`, `current_chapter: {NNN}`
2. Writer reads:
   - Chapter outline (`outline/L3-chapters/vol{N}-ch{NNN}.md`)
   - World bible (`world/`)
   - Style guide (`world/style-guide.md`)
   - Previous 2 chapters (for continuity)
3. Writer writes: `manuscript/vol{N}/ch{NNN}.md`
4. Prose Critic (`agents/prose-critic.md`) evaluates with `rubrics/prose-rubric.md`
5. Critic writes: `feedback/prose-ch{NNN}-r{N}.md`
6. Check verdict:
   - **PASS** → update state, move to next chapter
   - **FAIL** → feed feedback to Writer, iterate
   - **Max rounds reached** → `human_flag`, pause
7. Update metrics in state after each chapter

### Phase 7: Finalize

1. Concatenate all chapters into `final/full-novel.md`
2. Generate `final/stats.md` with:
   - Total word count
   - Per-chapter word counts
   - Final scores per chapter
   - Total iterations used
   - Chapters that needed human intervention
3. Set `status: completed`

## State Machine Transitions

```
pending → running → completed
                  → failed
                  → awaiting_human → running (after confirmation)
```

Each layer within `structure` and each chapter within `chapters` follows:
```
pending → in_progress → passed
                      → failed → in_progress (retry)
                      → human_flag → in_progress (after intervention)
```

## Error Handling

- **Agent timeout**: Log error, set chapter/layer status to `failed`, continue with next
- **Persistent failure**: After max rounds, flag for human intervention (never silently skip)
- **State corruption**: Validate state.yaml on every read; if invalid, attempt recovery from last known good state
- **Resume**: On restart, read `state.yaml`, skip completed phases/chapters, resume from `current_phase` + `current_chapter`

## File Communication Protocol

All agent coordination happens through files:

| Agent | Reads | Writes |
|-------|-------|--------|
| Architect | `brief.yaml`, `feedback/structure-review-*.md` | `world/*`, `outline/*` |
| Structure Critic | `world/*`, `outline/*`, `brief.yaml`, `rubrics/structure-rubric.md` | `feedback/structure-review-*.md` |
| Writer | `outline/L3-chapters/*`, `world/*`, `manuscript/*` (prev chapters), `feedback/prose-*.md` | `manuscript/vol*/ch*.md` |
| Prose Critic | `manuscript/vol*/ch*.md`, `outline/L3-chapters/*`, `world/style-guide.md`, `rubrics/prose-rubric.md` | `feedback/prose-*.md` |
| Orchestrator | `state.yaml`, all of the above | `state.yaml`, `final/*` |

No direct agent-to-agent message passing. The orchestrator mediates all communication by reading outputs and passing them as inputs to the next agent.
