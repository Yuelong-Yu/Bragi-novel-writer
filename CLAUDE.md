# Bragi - AI Novel Generation Harness

## What is Bragi?

Bragi is a multi-agent harness for generating high-quality web novels. It uses a GAN-inspired architecture: Generator agents (Architect, Writer) produce content, Critic agents (Structure Critic, Prose Critic) evaluate and provide feedback, and the cycle iterates until quality thresholds are met.

Named after the Norse god of poetry — son of Odin.

## Project Structure

```
bragi/
├── agents/              # Agent prompt definitions
│   ├── architect.md     # World-building + plotting (L0-L3)
│   ├── structure-critic.md  # Evaluates structure quality
│   ├── writer.md        # Generates chapter prose (L4)
│   └── prose-critic.md  # Evaluates prose quality
├── rubrics/             # Scoring criteria
│   ├── structure-rubric.md  # 6 dimensions for L0-L3
│   └── prose-rubric.md      # 7 dimensions for L4
├── templates/           # Input/state templates
│   ├── brief.yaml       # Novel brief input format
│   └── state.yaml       # Orchestration state machine
├── skills/              # Orchestrator workflows
│   └── write-novel.md   # Main generation pipeline
└── examples/            # Sample inputs
    └── demo-brief.yaml  # Example novel brief
```

## How to Run

1. Copy `templates/brief.yaml` to your working directory
2. Fill in the brief (or leave fields empty for Architect to decide)
3. Run: `/write-novel` or invoke the skill manually
4. The orchestrator will:
   - Create an output directory for your novel
   - Run Architect → Structure Critic loop (L0-L3)
   - Pause for your confirmation at key checkpoints
   - Run Writer → Prose Critic loop (L4) for each chapter
   - Output the final manuscript + all intermediate artifacts

## Key Concepts

### Generation Layers
- **L0**: Story core (logline, theme, genre, character cards)
- **L1**: Master outline (major conflicts, turning points, ending)
- **L2**: Volume outlines (core event chains per volume)
- **L3**: Chapter outlines (beat sheet per chapter)
- **L4**: Prose (full chapter text)

### Agent Communication
All agents communicate via files. No direct message passing.
- Generators write to `world/`, `outline/`, `manuscript/`
- Critics write to `feedback/`
- The orchestrator reads/writes `state.yaml`

### Quality Thresholds
- Pass: weighted score >= 7.0/10, no single dimension below 5.0
- Structure layer: max 5 iteration rounds
- Prose layer: max 3 iteration rounds per chapter
- If threshold not met at max rounds: flag for human intervention

### Human Checkpoints
The pipeline pauses for confirmation at:
- After L0 passes (confirm story direction)
- After L1 passes (confirm overall structure)
- After each volume's L2 passes (confirm volume direction)
- After Style Matcher recommends reference works (confirm style)

### Creation Modes
- **Original** (`mode: original`): From brief only — Architect generates everything from scratch
- **Adaptation** (`mode: adaptation`): Rewrite/remix — user provides `adaptation` block with source work, rewrite vision, and optional mapping hints. Architect uses these as structural constraints to build a new, original story
- **Expansion** (`mode: expansion`): Expand a short draft — user provides `existing_draft`. Architect extracts the skeleton first, then extends it into a full-length novel. Writer must not pad word count with filler

## Model Configuration

Models are user-configurable per agent in `brief.yaml`. Presets available:
- `quality`: All Opus
- `balanced`: Opus for generators, Sonnet for critics
- `budget`: All Sonnet/Haiku
- `cn-local`: DeepSeek/Qwen

## File-Driven State

The orchestrator maintains `state.yaml` in the output directory to track:
- Current phase and progress
- Scores and iteration counts per layer/chapter
- Status (running / awaiting_human / completed / failed)

This enables interrupt-and-resume: if the process stops, restart from where it left off.
