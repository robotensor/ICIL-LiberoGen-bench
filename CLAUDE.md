# icilval — ICIL competition validator

Python 3.10, package `icilval` under `src/`. Scores BPP-architecture submissions one success
rate per skill (`spec.json` `skills`: pick_and_place and goal_chain on LIBERO, draw_anything on
DrawAnything-Sim; where each skill's tasks come from is `skills.<skill>.tasks`;
a unit is one task, one initial state, one prompt demonstration and a seed, spread evenly over a
skill's eligible tasks), runs
King-of-the-Hill duels on the mean over skills, publishes a signed append-only store, mirrors it
to a Hugging Face dataset, and posts live frames to the dashboard. Follows the conventions in
`../CLAUDE.md`.

## Commands

- Host env (pure, no simulator): `uv venv .venv && uv pip install -e ".[dev]"`; `ruff check . && ruff format --check .`; `pytest -m "not sim and not gpu and not container"`.
- Simulator/GPU env: the pinned BPP conda env `/root/miniconda3/envs/bp` (`BP=/root/miniconda3/envs/bp/bin/python`, with `PYTHONPATH=src` and `SDL_VIDEODRIVER=dummy`); `$BP -m pytest -m sim`, `ICILVAL_TEST_GPU=1 $BP -m pytest -m gpu`.
- End to end: `icilval smoke --store <dir> --model-dir <genesis dir with one subdir per skill>` then `icilval store verify <dir>`.
- Container: `docker/build.sh`; `pytest -m container`.

## Rules

- `spec.json` and `store-schema.json` are the contract. No number from them is duplicated as a literal; read through `icilval.spec`.
- Submissions are `<skill>/model.safetensors` + `<skill>/config.yaml` per skill, nothing else. The validator never unpickles entrant data. Model-side runs happen in a container with `--network none`.
- Skills are data: iterate `spec.skills`; never name a skill in code paths that should generalise (`side_runner` dispatches on `spec.simulator(skill)`).
- Stored scores are fractions `[0, 1]`; `duel.score_margin` is percentage points; convert only in `icilval.duel.score`.
- Simulator imports (`libero`, `robosuite`, `pygame`, `torch`, `behavior_prompting`) are function-local so the host CLI imports without them.
- Pools are built offline (`icilval pools build` / `pools upgrade`); pickled `.pruned_init`/hdf5/zarr are read only at build time; runtime reads npz.
- Everything published is deterministic from `spec.json` + `pool_id` + the two model refs: unit lists, seeds, ids.
- BPP is vendored as a pinned git submodule at `vendor/behavior_prompting` (with `deps/LIBERO`); do not import its runner/workspace/dataset code, only the policy/model classes, the LIBERO env utilities and `DrawEnv`. Its generator scripts run as subprocesses at build time.

## Conventions

- Small commits. One concern per commit (a rename, a schema change, a new stage, a doc update),
  never a whole issue in one commit. Each commit builds and passes the pure tests on its own, so
  the history bisects and reverts cleanly. Split mechanical moves from behaviour changes.
- Commit title: `(feat): …`, `(fix): …`, `(refactor): …`, `(docs): …`, `(test): …`, `(chore): …`;
  imperative, lower-case after the prefix, under 72 characters, no trailing period. Body: why the
  change, not what the diff shows; short bullets; `Refs #N` for the issue it advances,
  `Closes #N` only on the commit that finishes it.
- Issues stand on their own: someone who was not in the conversation that produced one must be
  able to act on it. The title is the outcome in plain words - what is true once it closes
  ("Rebuild the identical scene and verify it matches") - not a component name or a plan step; a
  bug's title is its symptom. The body, in this order:
  - `## Why`: the problem, and what goes wrong without the change. No "see the plan", no "as
    discussed".
  - `## Scope`: the deliverable as concrete bullets (behaviour, files, commands), then
    `Out of scope:` for what a reader might expect and will not get.
  - `## Acceptance criteria`: a `- [ ]` checklist of things that can be checked - a test, a
    command and its result, an observable behaviour. Never "works well".
  - `## Notes`, optional: constraints, pitfalls, upstream references with paths, `Depends on #N`.
  - A bug has `## What happened` (the command, the commit, the evidence), `## Expected` and, once
    known, `## Cause`, in place of Why and Scope.
  - On closing, add `## Outcome`: what shipped and in which PRs, the measured result, and anything
    that differs from the scope. A criterion that was dropped or changed is said, not silently
    ticked.
- One issue is one deliverable. Label it with its area, add `bug` for a defect, and put it in a
  milestone when the work belongs to one; split anything that will not land in one go and link the
  parts with `Depends on #N`.
- A branch carries a theme, not an issue number: related issues that touch the same code ship on
  one branch (`short-slug`, or `issue-N-short-slug` when it really is a single issue) and land in
  one PR, which says `Closes #N` for every issue it finishes and `Refs #N` for the ones it only
  advances. Tests and a CHANGELOG entry land with it. Rebase, do not merge `main` into the branch.
- Small changes go straight to `main`: a typo, a comment, a doc line, a version bump, a one-line
  fix that comes with its test. Anything that changes behaviour a reader would need explained,
  touches a published contract, or wants a second pair of eyes takes a branch and a PR.
- When a branch is merged, delete it locally and on the remote, so only `main`, long-lived
  `milestone-*` branches and deliberate `archive/*` refs remain.
- Reports and evaluation results are plain files in the repository or run directory, not hosted
  artifacts.
