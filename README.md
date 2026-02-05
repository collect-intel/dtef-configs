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

Files in `/models/` define reusable sets of model identifiers. The filename (e.g., `CORE.json`) becomes the placeholder used in blueprints.

Model IDs use the format `provider:model` (e.g., `openrouter:openai/gpt-4o`).

## License

All content in this repository is dedicated to the public domain under the [CC0 1.0 Universal Public Domain Dedication](LICENSE).
