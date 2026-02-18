# DTEF Evaluation Blueprints

This repository stores evaluation blueprints for the [Digital Twin Evaluation Framework (DTEF)](https://github.com/collect-intel/dtef-app). Blueprints are generated from demographic survey data and define how AI models should be tested for their ability to predict survey response distributions across demographic groups.

## How It Works

1. **Survey data** (e.g., from [Global Dialogues](https://github.com/collect-intel/global-dialogues)) is imported into DTEF format
2. **Blueprints are generated** by the dtef-app CLI, creating evaluation prompts for each demographic segment
3. **Blueprints are published** to this repository
4. **The DTEF platform** reads blueprints from here and runs evaluations against multiple AI models
5. **Results** show how accurately each model predicts response distributions across demographic groups

## Repository Structure

```
.
├── blueprints/          # Generated evaluation blueprints (YAML/JSON)
├── models/              # Reusable model collection definitions
│   ├── CORE.json        # Core set of models to evaluate
│   ├── FRONTIER.json    # Frontier/latest models
│   └── ...
├── CONTRIBUTING.md
├── LICENSE              # CC0 1.0 Universal
└── README.md
```

## Generating Blueprints

Blueprints are generated using the dtef-app CLI. Do not manually create or edit blueprint files.

```bash
# Import Global Dialogues survey data
pnpm cli dtef import-gd --round GD4 -o output/gd4.json

# Generate blueprints from imported data
pnpm cli dtef generate -i output/gd4.json -o output/blueprints/

# Publish to this repository
pnpm cli dtef publish -s output/blueprints/ -t ../dtef-configs/blueprints/
```

## Model Collections

Files in `/models/` define reusable sets of model identifiers. The filename (e.g., `CORE.json`) becomes the placeholder used in blueprints. Model IDs use the format `provider:model` (e.g., `openrouter:openai/gpt-4.1`).

### Tiered Evaluation (CORE_FAST / CORE_SLOW)

All blueprint configs reference `models: ["CORE"]`. The CORE collection is split into two speed tiers to maximize throughput:

| Collection | Models | Avg Response | Purpose |
|---|---|---|---|
| **CORE_FAST** | 13 models | 0.8–3.0s | Fast-responding models. Completes an eval in ~3–4 min. |
| **CORE_SLOW** | 6 models | 4.6–17s | Frontier & large open-source models. ~17 min per eval. |
| **CORE** | = active tier | — | **What blueprints actually run.** Set to CORE_FAST or CORE_SLOW. |

`CORE.json` is the **active run list** — its contents determine which models get evaluated. `CORE_FAST.json` and `CORE_SLOW.json` are the canonical tier definitions.

### Running a Two-Phase Evaluation

Since each eval is gated by its slowest model, running all models at once means every eval takes ~17 min (grok-4.1-fast). Splitting into phases gets results for 13 fast models ~5x sooner.

**Phase 1 — Fast pass:**
```bash
# CORE.json already contains CORE_FAST models (default state)
# Hourly cron runs all 1306 configs at ~3-4 min each
# Optionally set GEN_TIMEOUT_MS=15000 in Railway env
```

**Phase 2 — Slow pass (after all configs are fresh):**
```bash
# Copy CORE_SLOW contents into CORE.json
cp models/CORE_SLOW.json models/CORE.json
git add models/CORE.json && git commit -m "switch CORE to CORE_SLOW for Phase 2" && git push

# Update Railway env: GEN_TIMEOUT_MS=60000
# Cron detects all configs as stale (new content hash) and re-schedules them
```

**After both phases complete:**
```bash
# Restore CORE to the full combined list
# (merge CORE_FAST + CORE_SLOW, or just keep in phased mode for ongoing runs)
cat models/CORE_FAST.json models/CORE_SLOW.json | jq -s 'add' > models/CORE.json
git add models/CORE.json && git commit -m "restore CORE to full model list" && git push
```

Each phase creates a separate run per config (different content hash), so results from both phases are preserved and visible in the dashboard.

### Timeout Configuration

Set these environment variables in Railway to control per-API-call timeouts:

| Variable | Default | Description |
|---|---|---|
| `GEN_TIMEOUT_MS` | *(none — falls back to 30s HTTP client default)* | Timeout per model API call in milliseconds |
| `GEN_RETRIES` | *(none)* | Number of retries per failed API call |

Recommended values: `GEN_TIMEOUT_MS=15000` for CORE_FAST, `GEN_TIMEOUT_MS=60000` for CORE_SLOW.

### Available Collections

| File | Description |
|---|---|
| `CORE.json` | Active run list (set to CORE_FAST or CORE_SLOW) |
| `CORE_FAST.json` | 13 models with <3s avg response time |
| `CORE_SLOW.json` | 6 models with >3s avg (frontier, large open-source) |
| `FRONTIER.json` | Top-tier frontier models across providers |
| `QUICK.json` | 5 fast budget models for quick test runs |
| `CLAUDES.json` | All Anthropic Claude variants |
| `LLAMAS.json` | Meta Llama model family |
| `OPEN_SOURCE_CORE.json` | Open-weight models |

## License

All content in this repository is dedicated to the public domain under the [CC0 1.0 Universal Public Domain Dedication](LICENSE).
