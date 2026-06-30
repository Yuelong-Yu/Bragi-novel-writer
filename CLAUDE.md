# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

**Bragi** is a prompt-and-spec repo — not an application. It defines a multi-agent harness for generating web novels: agent prompts (`agents/`), scoring rubrics (`rubrics/`), I/O templates (`templates/`), and the orchestration skill (`skills/write-novel.md`). There is no build, no test runner, no package manifest. Edits are made to Markdown/YAML and consumed by Claude Code at run time.

Parent context: this repo lives inside `Odin/` (see `../CLAUDE.md`), an AI-native web novel platform. Bragi is the content-generation subsystem.

## Running the Pipeline

```
/write-novel [path-to-brief]    # defaults to ./brief.yaml
```

The orchestrator (`skills/write-novel.md`) reads `templates/brief.yaml`, creates `output/《{project}》/`, then drives the L0→L4 pipeline. There is no CLI binary — execution happens entirely inside a Claude Code session via the skill.

For a quick smoke run, copy `templates/demo-brief.yaml` to a working dir as `brief.yaml` and invoke `/write-novel`.

## Architecture — Big Picture

### Two-loop GAN-style harness

```
Architect ⇄ Architecture Critic   (structure loop, L0–L3)
Writer    ⇄ Writing Critic        (prose loop, L4, per chapter)
```

Generators write artifacts → critics score against the rubric → orchestrator either passes or feeds feedback back to the generator. Quality gate: weighted score ≥ 7.0 AND no single dimension < 5.0.

### Layer hierarchy (read top-down)

| Layer | Owner | Output |
|---|---|---|
| L0 — story core | Architect | `world/core.md`, `world/setting.md`, `world/characters.md`, `world/timeline.md` |
| L1 — master outline | Architect | `outline/L1-master.md` |
| L2 — volume outlines | Architect | `outline/L2-vol{N}.md` |
| Style match (optional) | Orchestrator | `world/style-guide.md` |
| L3 — chapter beats | Architect | `outline/L3-chapters/vol{N}-ch{NNN}.md` |
| L4 — prose | Writer | `manuscript/vol{N}/ch{NNN}.md` |

Critics write to `feedback/structure-review-r{N}.md` and `feedback/prose-ch{NNN}-r{N}.md`.

### File-driven state, no IPC

Agents never message each other. The orchestrator mediates everything by reading one agent's output files and passing them as input to the next. `state.yaml` (template at `templates/state.yaml`) tracks `current_phase`, `current_volume`, `current_chapter`, per-layer/chapter status, scores, rounds, and human-checkpoint flags. This enables interrupt-and-resume — restarting reads `state.yaml` and continues from `current_phase`.

### Human checkpoints

The pipeline pauses (`status: awaiting_human`) after L0 pass, L1 pass, each L2 volume pass, and the style recommendation. The orchestrator must not silently auto-continue past these.

### Three creation modes

Set `mode` in `brief.yaml`:

- `original` — generate everything from scratch
- `adaptation` — requires `adaptation` block (`source_work`, `adaptation_idea`, `mapping_hints`); Architect treats mappings as structural constraints and produces a new, independently coherent story. Writer must never reproduce source prose.
- `expansion` — requires `existing_draft`; Architect performs **skeleton extraction** at L0 (characters, world rules, plot threads, hooks) before writing world files. Writer must not pad word count with filler.

Mode validation happens in Phase 0 of `skills/write-novel.md` — missing required blocks should error out, not be silently defaulted.

### Iteration limits

- Structure loop: `max_structure_rounds` (default 5)
- Prose loop per chapter: `max_prose_rounds` (default 3)
- On exhaustion → status `human_flag`, never silently skip.

## Core Design Concepts (when editing agent prompts)

These principles are load-bearing across multiple files; changing them in one place without auditing the rest creates drift.

**Personality Axes (6 轴) + Axis Contrast Map** — Every major character has 6 dimensions (`decision_mode`, `conflict_response`, `trust_baseline`, `control_drive`, `emotional_volatility`, `moral_flexibility`), each with `baseline` (1–10) and `exception` (situational override). Defined in `agents/architect.md`, scored by `agents/architecture-critic.md` (under Character Dimensionality), and executed by `agents/writer.md` (under Personality Axis Execution). All three files must stay in sync. The rubric requires: every major pair has ≥1 axis with baseline diff ≥ 5; every exception is triggered ≥ once.

**Fractal Tension / 分形张力** — Tension at every scale (arc → chapter → scene → paragraph). Owned a dedicated 20% rubric dimension in `rubrics/architecture-rubric.md`. Architect must mark micro-conflict for every L3 scene.

**Cinematic Visualization / 画面感** — 15% prose rubric dimension. Defined in `agents/writer.md` and evaluated in `agents/writing-critic.md`.

## File Communication Protocol (authoritative)

| Agent | Reads | Writes |
|-------|-------|--------|
| Architect | `brief.yaml`, `feedback/structure-review-*.md` | `world/*`, `outline/*` |
| Architecture Critic | `world/*`, `outline/*`, `brief.yaml`, `rubrics/architecture-rubric.md` | `feedback/structure-review-*.md` |
| Writer | `outline/L3-chapters/*`, `world/*`, `manuscript/*` (prev chapters), `feedback/prose-*.md` | `manuscript/vol*/ch*.md` |
| Writing Critic | `manuscript/vol*/ch*.md`, `outline/L3-chapters/*`, `world/style-guide.md`, `rubrics/writing-rubric.md` | `feedback/prose-*.md` |
| Security Critic | `final/full-novel.md` (or per-chapter prose), `brief.yaml`, `world/*`, `rubrics/security-rubric.md` | `feedback/security-*.md` |
| Orchestrator | `state.yaml` + all above | `state.yaml`, `final/*` |

When adding a new artifact or rubric dimension, update both the agent prompts that produce/consume it and this table.

## Editing Conventions

- **Language follows the brief.** Chinese brief → Chinese outputs from agents. Don't translate fixed Chinese terms (分形张力, 画面感, 6 轴, baseline/exception YAML keys).
- **Critics never rewrite** — they emit directives only. Don't add rewrite responsibilities to critic prompts.
- **Architect never writes prose**; Writer never invents plot beats outside the L3 outline. Keep the separation strict when adding instructions.
- **Rubric weights must sum to 100%** in each rubric file. If you change weights, double-check.
- **Pass criteria are duplicated** in the skill, both agent prompts, and `README.md` (≥ 7.0 weighted, no dim < 5.0). Change all four together.
