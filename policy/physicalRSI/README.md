# PhysicalRSI

**Contributor:** HKU MMLAB | **Project:** [PhysicalRSI](https://mmlab.hk/research/PhysicalRSI)

An evaluation adapter with an API-backed agent, task-aware memory, and VLA and code-policy skills.

Shared evaluation conventions are documented in the [XPolicyLab README](../../README.md).

## Installation

Set up XPolicyLab and the RoboDojo simulator following their installation guides. These commands require Linux, an NVIDIA GPU, and `uv`. Run them from `policy/physicalRSI` in the same shell:

```bash
python download_assets.py --output skill-assets
uv sync --project skill-assets/implementations/openpi --python 3.11 --frozen --no-dev
policy_python="$PWD/skill-assets/implementations/openpi/.venv/bin/python"
bash install.sh "$policy_python"

PYTHONPATH=runtime "$policy_python" -m PhysicalRSI_baselines.robodojo.prepare_code_programs \
  "$PWD/skill-assets/implementations/code-skills"
PYTHONPATH=runtime "$policy_python" -m PhysicalRSI_baselines.robodojo.configure_skills \
  --assets "$PWD/skill-assets" --framework "$(cd ../.. && pwd)" \
  --python "$policy_python" --output "$PWD/skills.json" --evidence "$PWD/results" \
  --endpoint https://YOUR_API_HOST/v1/chat/completions --model YOUR_VISION_MODEL
```

Use an image-capable chat-completions endpoint and a new configuration output file.

## Data Processing

Not applicable (evaluation only).

## Training

Not applicable (evaluation only).

## Evaluation

```bash
export ROBODOJO_CONDA_ENV=YOUR_SIMULATOR_ENV
export PHYSICALRSI_SKILL_CONFIG="$PWD/skills.json"
export PHYSICALRSI_AGENT_API_KEY=YOUR_API_KEY
export PHYSICALRSI_EVAL_OUTPUT="$PWD/results"
PYTHONPATH=runtime "$policy_python" -m PhysicalRSI_baselines.robodojo.skill_preflight "$PHYSICALRSI_SKILL_CONFIG"

EVAL_ENV_TYPE=sim bash eval.sh \
  RoboDojo general_pickup skill-library arx_x5 joint 0 0 0 \
  "$policy_python" "$ROBODOJO_CONDA_ENV"
```

Set `ROBODOJO_CONDA_ENV` to the existing simulator environment name. `OPENAI_API_KEY` and `ARK_API_KEY` are also supported.
