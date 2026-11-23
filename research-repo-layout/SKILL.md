---
name: research-repo-layout
description: Design or refactor a research repository by choosing clear training and inference entry points, placing reusable code, and consolidating experiment variants into configuration.
---

# Research repository layout

## When to use

Use this skill to design a research codebase, simplify its public commands, assign
ownership across projects, or consolidate experiment launchers. For a local bug,
a model implementation change, or a framework migration, follow the repository's
existing layout unless the user also requests structural work.

## Choose the layout

Start with `workspace → project directories → project entry points and library`.
Use `train.py` and `infer.py` at each project root as the default pair. Existing
public equivalents such as `test.py`, installed commands, or a documented mode
can serve the same roles. Preserve commands that users and automation rely on.

Keep pipeline order visible in the top-level README and in the orchestration
flow. Numbered directories are useful for stages such as data preparation,
matching, and geometry. Place orchestration that spans several stages beside
those stages so its ownership is clear.

| Project shape | Keep in the entry | Keep in the library or model directory |
|---|---|---|
| Single model | Read configuration and show data → model → training or inference in execution order. LocalTrans illustrates visible entry flow with `train.py`, `test.py`, `config/`, and `lib/`. | Dataset, network, losses, reusable geometry, and code with its own testable contract. |
| Multiple models or runtimes | Select model and stages, check compatibility, and construct calls to the native runners. Keep adapters thin. | Native trainers, model-specific inference, and runtime dependencies stay with each model. Share code with a stable input/output contract. |

Extract a module when it has a clear owner, independent tests, or multiple real
callers. Keep short, single-caller glue near the flow it explains.

## Put variants in configuration

Compare two candidate launchers before extracting their common loop. Record
which differences affect the dataset, model, objective, schedule, distributed
strategy, or output schema. Use validated configuration to choose those
behaviors. Keep distinct algorithms as named implementations behind the same
command. Preserve defaults and compatibility for configs used by prior runs.

SSHNet-Loc keeps one main train entry and one main test entry. Represent its
online versus presplit data handling explicitly. SSHNet-rebuild shows how
regular, supervised, and multigpu scripts can repeat optimizer, validation,
and checkpoint logic.

RVHomo is a concrete script-fork example. The surveyed checkout has `train.py`
and twelve `train_*.py` files, including a 26-line adapter. The tree has about
4.9k lines across those files. Related scripts such as `train_dis_stage2.py` and
`train_dis_stage2_plain.py` repeat training infrastructure. Keep one loop and
select the stage and objective explicitly, with tests for their differences.

## Anti-patterns to check

- `train_<dataset>_<loss>_<gpu>.py` forks for choices a shared loop can express.
- Splitting roughly 600 lines into 18 micro-files when each file simply forwards
  to another and the reader has to reconstruct the execution order.
- Parallel management layers for launching, resuming, or recording the same run.
- Root-level clutter from experiment scripts, generated results, and competing
  current handoffs.
- Current documentation that still presents a retired command as runnable.
- Deleting weights, input data, frozen configs, or provenance-bearing caches as
  part of code cleanup.

## Reference layouts

- MMDetection and Detectron2 expose `tools/train.py` and `tools/test.py` and keep
  reusable implementation in `mmdet/` and `detectron2/`, respectively.
- PyTorch Lightning separates model and step logic in `LightningModule` from
  orchestration in `Trainer`. Borrow that boundary when the project needs it.
- nnU-Net v2 exposes `nnUNetv2_train` and `nnUNetv2_predict`, with reusable
  implementations under `nnunetv2/`.
- DETR uses root `main.py` with `--eval`. A small project can share one
  orchestration entry across modes when the command and modes are clear.

## Keep paths migration-safe

Design file placement and path code so a later rename/move is cheap:

- Derive every path from `__file__`; never hardcode absolute machine paths.
- Centralize repository paths in one module (for example `paths.py`) and let code
  import constants instead of repeating string literals.
- Do not use `parents[N]`: locate the root by walking up to a marker such as
  `.git` or a known file, so adding a directory level cannot silently break it.
- Store only repository-relative paths in configs/protocols, and never store
  machine paths, so a move can be bridged with a symlink without changing
  file hashes.
- Name directories by responsibility; use numeric prefixes only for pipeline
  order, and do not mix both conventions in one tree.
- Every rename/move ships with a `MIGRATION.json` mapping and, when historical
  runs must keep resolving, compatibility symlinks at the old paths.
- Before adding a root-level file, ask which project entry owns it. Root is a
  workspace index, not a scratch directory.

## Refactor checklist

1. Inventory the public commands, callers, launchers, and model-specific
   workers. Choose canonical names, document any compatibility window, and
   classify historical commands as historical.
2. Define the executed schema, defaults, overrides, and validation. Keep
   experiment parameters, machine paths, and frozen dataset/evaluation
   protocols distinct. Verify the exact command passed to each runner.
3. Capture baseline CLI plans, exit behavior, output paths, and schemas.
   Test both entries and affected adapters. Compare the before/after plans
   and use an approved small run where runtime behavior needs verification.
4. Preserve weights, original and resolved configs, manifests, and caches
   needed for reproduction. Record the command, code revision, data identity,
   and checkpoint source and digest. Check cache identity and completion before
   reuse. Keep historical snapshots as recorded.
5. Before removing an entry or artifact, check imports, shell calls, docs,
   automation, active runs, and external consumers. Confirm a recovery source
   for its actual contents and obtain the required deletion approval. Record
   path moves separately from the original run evidence.
6. Give newcomers a path from README to the two commands, then to an executed
   config, the responsible implementation, and a run's provenance. Check the
   links and command examples after moving files.

## Two-minute review

- Can I find the train and inference commands and see their execution order?
- Can I change an experiment variant without copying the training loop?
- Does each adapter have a clear boundary with its native runner?
- Can I trace a result to its code, resolved config, dataset, and checkpoint?
- Which callers and saved artifacts would a rename or deletion affect?
