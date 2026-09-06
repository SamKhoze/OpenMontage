# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

**MANDATORY: Read [`AGENT_GUIDE.md`](AGENT_GUIDE.md) before responding to ANY user message.**

Do not act on the user's request until you have read AGENT_GUIDE.md.
It contains routing rules that determine your first action based on what the user asked.
Skipping it WILL cause you to take the wrong action.

All production and routing rules live in AGENT_GUIDE.md. Architecture, key files, and conventions live in [`PROJECT_CONTEXT.md`](PROJECT_CONTEXT.md), with a deeper walkthrough in `docs/ARCHITECTURE.md`. This file only adds developer commands and orientation that those documents do not cover.

## Development commands

The Makefile targets an active `$VIRTUAL_ENV` or `$CONDA_PREFIX` first and falls back to `.venv`. Activate `.venv` (`source .venv/bin/activate`) before running anything, or an already active conda env will receive the installs.

```bash
make setup                 # venv + requirements + remotion-composer npm install + piper-tts + HyperFrames cache + .env
make install-dev           # adds pytest, pytest-asyncio, httpx
make test                  # full suite (tests/), same as CI
make test-contracts        # schema/manifest/registry contract tests only
python -m pytest tests/contracts/test_pipeline_catalog.py -v                 # one file
python -m pytest tests/contracts/test_pipeline_catalog.py::test_name -v      # one test
make lint                  # py_compile smoke check on the core modules (what CI runs)
make preflight             # full provider menu from the tool registry
make hyperframes-doctor    # node/ffmpeg/npx probe plus `hyperframes doctor`
make demo                  # render the zero-key Remotion demos to projects/demos/renders/
python render_demo.py <name>   # render one demo; `--list` shows names
python -m backlot serve --port 4750     # Backlot board server in the foreground
python -m backlot open <project-id>     # start server if needed and open the browser
cd remotion-composer && npm start       # Remotion Studio for the React scene components
```

Registry preflight one-liner (the human-readable rollup AGENT_GUIDE.md asks for):

```bash
python -c "from tools.tool_registry import registry; import json; registry.discover(); print(json.dumps(registry.provider_menu_summary(), indent=2))"
```

### Tests

- `tests/conftest.py` blocks every non-loopback socket for the whole session. A test that needs a real endpoint must be marked `@pytest.mark.live_api` and is skipped unless `OPENMONTAGE_ALLOW_NETWORK=1` is set. Mock the transport instead (see the fake-`requests` pattern in `tests/tools/test_atlas_video.py`).
- The guard covers only the pytest process. Tests that shell out to node, ffmpeg, or npx are outside it, so never call a paid API from a subprocess in a test.
- `tests/contracts/` is the contract layer: it asserts registry fields, manifest shape, artifact schemas, and that instruction files (AGENT_GUIDE.md, director skills) still say what the code depends on. Editing a director skill or manifest can fail a contract test.
- `tests/qa/` holds tool-by-tool output inspection scripts driven by `tests/qa/QA_PLAN.md`; they are not part of `make test`.
- Piper TTS looks for its voice model (`<voice>.onnx`, gitignored) in the current working directory, so run from the repo root and fetch one with `python -m piper.download_voices en_US-lessac-medium`.

## Orientation

Things that take several files to work out:

- **The agent is the orchestrator.** There is no Python pipeline runner. The agent reads `pipeline_defs/<pipeline>.yaml`, then `skills/pipelines/<pipeline>/<stage>-director.md` per stage, calls tools, self-reviews via `skills/meta/reviewer.md`, and writes checkpoints with `lib/checkpoint.py`. Python holds tools and persistence only; do not add orchestration, review, or checkpoint policy to Python.
- **Tool contract.** Every tool subclasses `BaseTool` in `tools/base_tool.py`, is called with `.execute(dict)` (not `.run`), and returns a `ToolResult` (`success`, `data`, `error`, `artifacts`). Class names are PascalCase with no `Tool` suffix (`VideoCompose`, `PiperTTS`). `tools/tool_registry.py` discovers tools by import; nothing should import provider tools ad hoc.
- **Selector plus provider.** Capability families (tts, image_generation, video_generation) each have a selector tool that routes to concrete provider tools found by `registry.get_by_capability(...)`. Adding a provider tool with the right `capability` field makes it reachable through the selector with no selector changes.
- **Availability is declarative.** A tool's `get_status()` (by default `check_dependencies()`) decides whether the registry lists it as configured, and the tool's `install_instructions` and `dependencies` fields are what the agent shows users. Keep those fields accurate; skills and prompts must not hardcode provider names, env var names, or setup URLs.
- **Three composition runtimes.** `tools/video/video_compose.py` dispatches on `edit_decisions.render_runtime` to FFmpeg, Remotion (`remotion-composer/`, scene types in `remotion-composer/SCENE_TYPES.md`), or HyperFrames (`tools/video/hyperframes_compose.py`, consumed through `npx hyperframes`). The runtime is chosen at proposal time and must never be swapped silently downstream.
- **Canonical artifacts are the inter-stage contract.** Each stage writes one JSON artifact (`brief`, `script`, `scene_plan`, `asset_manifest`, `edit_decisions`, `render_report`) validated against `schemas/artifacts/`. Checkpoints live in `projects/<project-id>/checkpoint_<stage>.json`; superseded ones are archived to `projects/<project-id>/history/`. `lib/checkpoint.py` refuses to mark a gated stage `completed` without `human_approved=True`.
- **Backlot reads disk, never the agent.** The board (`backlot/`) derives everything from `projects/<project-id>/` (`project.json`, checkpoints, artifacts, assets). The agent's only board duty is `python -m backlot open <project-id>` at pipeline init. `projects/` is gitignored.
- **Style playbooks** in `styles/*.yaml` are validated by `schemas/styles/playbook.schema.json` and loaded by `styles/playbook_loader.py`, which also derives design tokens used by both Remotion themes and the HyperFrames CSS bridge (`lib/hyperframes_style_bridge.py`).
- **Three knowledge layers.** `tools/` (what exists), `skills/` (how OpenMontage uses it), `.agents/skills/` (vendor and technology knowledge, indexed in `skills/INDEX.md`). Each tool's `agent_skills` field links layer 1 to layer 3.

Project slash commands live in `.claude/commands/`: `/backlot`, `/ink-art`, `/animated-drawing`. Their Cursor equivalents in `.cursor/commands/` mirror them.

Pull requests to upstream are reviewed against `docs/PR_REVIEW_GUIDE.md`; provider changes are expected to update `docs/PROVIDERS.md` and the contract tests together.

If a `CLAUDE.local.md` exists next to this file, it holds owner- and machine-specific instructions for this clone and is loaded automatically; follow it as well.
