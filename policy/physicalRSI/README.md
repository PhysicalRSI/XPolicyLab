# PhysicalRSI execution skills for XPolicyLab

**Contributor:** HKU MMLAB | **Project:** [PhysicalRSI v2](https://yanming03.github.io/PhysicalRSI/v2/) | **arXiv:** Not specified | **Original code:** [PhysicalRSI](https://github.com/yanming03/PhysicalRSI)

This adapter exposes one XPolicyLab entrypoint backed by the PhysicalRSI skill
library. Internal capabilities live in `PhysicalRSI_baselines/robodojo/skills/`;
they are not installed as separate XPolicyLab entries.

The runtime exposes one skill library containing pi05, pi05-sparse-memory and
code-policy implementations, with separate weight identities recorded explicitly.
At episode start an API-backed agent reads instructions, observations and immutable
task-aware memory, then chooses a registered, configured skill composition. That
choice determines the actual execution skill. The composition remains stable until
reset; observations, policy instructions and actions are not rewritten. Missing
credentials or an invalid choice stop execution without a fallback.


Supported benchmark: RoboDojo; embodiment: `arx_x5`. Neural skills use joint actions; frozen code programs declare their original joint or EEF action contract. Evaluation must use the matching action type.

Shared conventions — argument meanings, checkpoint naming, split-machine deployment, `EVAL_ENV_TYPE` — are documented in the [XPolicyLab README](../../README.md). Official results: [RoboDojo LeaderBoard](https://robodojo-benchmark.com/LeaderBoard).


## Installation

From the installed `policy/physicalRSI` directory:

```bash
python download_assets.py --output skill-assets
uv sync --project skill-assets/implementations/openpi --frozen --no-dev
bash install.sh "$PWD/skill-assets/implementations/openpi/.venv/bin/python"
```

The downloader checks every archive part against `assets.json` before extraction.
OpenPI includes the sparse visual-memory architecture and both model configurations.
Runtime source is bundled under `runtime/`; no editable PhysicalRSI checkout is required.
For an existing compatible policy environment, run `install.sh /path/to/python`.

Copy `runtime/PhysicalRSI_baselines/robodojo/configs/skill_library.example.json` and set
checkpoint paths, model configuration, agent endpoint, and agent model. The example enables both neural skills; supply both checkpoint paths.
Enable code-policy only after supplying its validated assembly and runtime bindings.
Credentials belong in the environment, not in this configuration.

After installing the code-program archive, bind its verified templates to the
installation directory. This creates `programs.json` without changing the archive:

```bash
PYTHONPATH=runtime python -m PhysicalRSI_baselines.robodojo.prepare_code_programs \
  "$PWD/skill-assets/implementations/code-skills"
```

The code-program archive contains seven checksummed parts. Its perception and
motion-planning services require the dependencies included with each program,
in addition to the lightweight adapter environment.

Alternatively, generate a configuration from installed assets. This registers all
code programs present in an installed `implementations/code-skills/programs.json`
manifest and checks their declared source hashes:

```bash
PYTHONPATH=runtime python -m PhysicalRSI_baselines.robodojo.configure_skills \
  --assets "$PWD/skill-assets" \
  --python "$PWD/skill-assets/implementations/openpi/.venv/bin/python" \
  --output "$PWD/skills.json" --evidence "$PWD/results" \
  --endpoint https://YOUR_API_HOST/v1/chat/completions --model YOUR_VISION_MODEL
```

The generator refuses to overwrite an existing configuration. Without an installed
code-program manifest it configures the two neural skills only; inspect its printed
skill list before evaluation.

The frozen program services retain their original sidecar ports. Run one code
program service per host or isolated container to keep sidecar ports isolated.


```bash
export PHYSICALRSI_SKILL_CONFIG=/absolute/path/to/skills.json
export PHYSICALRSI_AGENT_API_KEY=...  # OPENAI_API_KEY or ARK_API_KEY also accepted
export PHYSICALRSI_EVAL_OUTPUT=/path/to/run-evidence
PYTHONPATH=runtime python -m PhysicalRSI_baselines.robodojo.skill_preflight "$PHYSICALRSI_SKILL_CONFIG"
```

The agent endpoint uses chat completions with image input. This protocol must
match the chosen provider; credentials alone do not establish API compatibility.

## Data Processing

Unsupported in this eval-only integration. No data conversion is needed for inference.

## Training

This integration performs inference and evaluation only. Data processing and
training are unsupported; `process_data.sh` and `train.sh` are intentionally absent.
Checkpoint architecture/configuration metadata is used solely for model loading.


## Evaluation

The scripts use XPolicyLab's server/client lifecycle and ten-argument interface:

```bash
EVAL_ENV_TYPE=debug bash /path/to/XPolicyLab/policy/physicalRSI/eval.sh \
  RoboDojo general_pickup skill-library arx_x5 joint 0 0 0 \
  /path/to/policy/venv base
```

Use `EVAL_ENV_TYPE=sim` and the simulator environment for real rollouts. The
policy environment accepts a uv project, a virtualenv directory, or a Python
executable. `uv` resolves the configured `policy_uv_env_path` (default
`runtime_env` under the installed adapter). The evaluation environment follows
XPolicyLab's shared client conventions. Single-environment evaluation is the
default, as in GPT Direct EEF. Batched skill state is isolated by environment.

## Unified skill and memory contracts

Executable skills expose observe, execute, reset, and close methods. Their
compositions use the repository's `Contract`, `Operation`, `Context`, and
`sequence`. Memory uses `MemoryStore.snapshot` and `Snapshot.reader`: revisions
are content-addressed and checked when read. Guidance and execution receipts
record skill, composition, and memory revisions.

The checked-in `configs/skill_compositions.json` defines memory access followed
by pi05 execution, sparse visual-history execution, or an isolated primitive
program. The agent chooses from the compositions enabled in the frozen configuration. The code-policy skill requires the existing validated primitive
assembly, harness, and binding; a metadata file alone is not executable.
