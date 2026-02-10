# DTEF Configs Repository

## Quick Context

You are working in **collect-intel/dtef-configs** (locally at `~/Documents/GitHub/dtef/dtef-configs`). This is the **blueprints repository** for the Digital Twin Evaluation Framework.

**Related Repository:** The evaluation platform and web dashboard is at `~/Documents/GitHub/dtef/dtef-app` (`collect-intel/dtef-app`).

## What is DTEF?

DTEF measures how accurately AI models predict demographic survey response distributions. Given a poll question and a demographic segment (e.g., "women aged 26-35 in Germany"), models predict how that group would respond — and DTEF scores them against actual survey data.

**Live site:** https://digitaltwinseval.org (deployed on Railway at `dtef-app-production.up.railway.app`)

## Two-Repo System

```
collect-intel/dtef-app                    collect-intel/dtef-configs
├── Next.js web dashboard                 ├── blueprints/     (YAML evaluation configs)
├── CLI tools (generate, import, run)     │   ├── gd1/ - gd6/  (base blueprints)
├── Evaluation engine                     │   └── gd1-ctx/ - gd6-ctx/ (context-enriched)
├── API routes                            └── models/         (model collection JSON)
└── Demographics aggregation
```

**Workflow:**
1. Survey data (Global Dialogues CSV) is imported via `pnpm cli dtef import-gd` in dtef-app
2. Blueprints are generated via `pnpm cli dtef generate` and published here via `pnpm cli dtef publish`
3. Weekly GitHub Actions cron fetches blueprints from this repo and runs evaluations
4. Results stored in S3, displayed on digitaltwinseval.org

## Directory Structure

```
dtef-configs/
├── blueprints/
│   ├── gd1/          # Global Dialogues Round 1 (base)
│   ├── gd1-ctx/      # GD1 with context-enriched prompts
│   ├── gd2/ gd2-ctx/ # Rounds 2-6 follow same pattern
│   ├── gd3/ gd3-ctx/
│   ├── gd4/ gd4-ctx/
│   ├── gd5/ gd5-ctx/
│   └── gd6/ gd6-ctx/
├── models/
│   ├── CORE.json               # 22 models (default for all blueprints)
│   ├── FRONTIER.json           # 14 cutting-edge models
│   ├── QUICK.json              # 5 models for fast testing
│   ├── EXPERIMENTAL.json       # New/beta models
│   ├── CLAUDES.json            # All Claude variants
│   ├── LLAMAS.json             # All Llama variants
│   ├── OPEN_SOURCE_CORE.json   # Open-source subset
│   └── OPENAI_GPT4O_SNAPSHOTS.json  # GPT-4o dated snapshots
└── .claude/
    └── CLAUDE.md               # This file
```

~566 blueprint YAML files across 12 directories (6 base + 6 context-enriched).

## Blueprint Format

DTEF blueprints use the `distribution_metric` point function — NOT the Weval `should`/`should_not` rubric format. Blueprints are **machine-generated**, not hand-authored.

### Example Blueprint

```yaml
title: "DTEF GD4: Age Group 18-25"
description: "Predict survey response distributions for age group 18-25"
configId: "dtef-global-dialogues-gd4-ageGroup:18-25"
tags:
  - dtef
  - demographic
  - global-dialogues-gd4
  - _periodic
models:
  - CORE

---
- id: "gd4-q1-ageGroup:18-25"
  prompt: |
    For the following poll question, predict how people in the demographic
    group "Age Group: 18-25" would respond.

    Poll Question: "Should AI systems be required to explain their decisions?"

    Respond with a JSON object mapping each answer option to a percentage...
  should:
    - fn: distribution_metric
      fnArgs:
        expected:
          "Strongly agree": 0.42
          "Somewhat agree": 0.31
          "Neutral": 0.15
          "Somewhat disagree": 0.08
          "Strongly disagree": 0.04
        metric: js-divergence
        threshold: 0.85
```

### Key Fields

- **configId**: Contains colons (e.g., `ageGroup:18-25`, `country:australia`). These need URL encoding (`%3A`) in web contexts.
- **tags**: `dtef`, `demographic`, `global-dialogues-<round>`, `_periodic` (auto-run weekly)
- **fn: distribution_metric**: Compares predicted vs actual response distributions using JS-divergence, cosine similarity, or earth-mover distance.
- **fnArgs.expected**: Ground-truth distribution from survey data (values sum to 1.0)
- **fnArgs.metric**: `js-divergence` (default), `cosine`, or `earth-mover`
- **fnArgs.threshold**: Score threshold for passing (0.0–1.0)

### Context-Enriched Blueprints (`-ctx` directories)

The `-ctx` variants include additional context in prompts (e.g., prior questions/answers, survey metadata) to test whether more context improves prediction accuracy. Generated with `--context-questions` flag.

## How Blueprints Are Generated

Blueprints are generated programmatically from Global Dialogues survey data. **Do not hand-edit blueprint files** — regenerate them instead.

```bash
# In dtef-app repo:

# 1. Import Global Dialogues CSV data
pnpm cli dtef import-gd -r GD4 -o data/gd4.json

# 2. Generate blueprints from imported data
pnpm cli dtef generate -i data/gd4.json -o blueprints/gd4/

# 3. Generate context-enriched variants
pnpm cli dtef generate -i data/gd4.json -o blueprints/gd4-ctx/ --context-questions all

# 4. Publish to this configs repo
pnpm cli dtef publish -s blueprints/gd4/ -t ../dtef-configs/blueprints/gd4/

# 5. Validate data
pnpm cli dtef validate -i data/gd4.json

# 6. Preview before generating
pnpm cli dtef preview -i data/gd4.json --segment "ageGroup:18-25"
```

## Testing Locally

```bash
# In dtef-app repo:
pnpm cli run-config local \
  --config ../dtef-configs/blueprints/gd4/dtef-global-dialogues-gd4-ageGroup:18-25.yml \
  --eval-method llm-coverage

# Start dev server
pnpm dev  # http://localhost:3172
```

## Model Collections

Reference by UPPERCASE name in blueprint `models:` field. Files are JSON arrays of model IDs.

| Collection | Count | Purpose |
|-----------|-------|---------|
| CORE | 22 | Default for all blueprints |
| FRONTIER | 14 | Cutting-edge models |
| QUICK | 5 | Fast testing |
| EXPERIMENTAL | varies | New/beta models |
| CLAUDES | varies | All Claude variants |
| LLAMAS | varies | All Llama variants |
| OPEN_SOURCE_CORE | varies | Open-source subset |
| OPENAI_GPT4O_SNAPSHOTS | varies | GPT-4o dated snapshots |

Model ID format: `openrouter:<provider>/<model>` (e.g., `openrouter:openai/gpt-4o`)

## Tags

| Tag | Purpose |
|-----|---------|
| `dtef` | All DTEF blueprints |
| `demographic` | Demographic evaluation |
| `global-dialogues-<round>` | Source round (gd1–gd6) |
| `_periodic` | Auto-run weekly via cron |
| `_featured` | Show on homepage (none currently set) |
| `_test` | Exclude from production |

## Key Gotchas

- **ConfigIds contain colons**: `dtef-global-dialogues-gd4-ageGroup:18-25` — colons stay as `%3A` in Next.js URL params. All dtef-app pages use `decodeURIComponent()`.
- **Don't hand-edit blueprints**: Regenerate from source data using the CLI pipeline.
- **Global Dialogues data**: Git submodule at `data/global-dialogues` in dtef-app. Rounds GD1–GD7 + GD6UK.
- **Segment types**: O2 (age), O3 (gender), O4 (environment), O5 (AI concern), O6 (religion), O7 (country). O1 (language) is excluded.

## Deployment

- **Platform**: Railway (auto-deploys dtef-app from GitHub main branch)
- **Weekly cron**: GitHub Actions workflow (`.github/workflows/weekly-eval-check.yml`) triggers evaluations
- **S3 storage**: `collect-intel-dtef` bucket (us-east-1) for all results
