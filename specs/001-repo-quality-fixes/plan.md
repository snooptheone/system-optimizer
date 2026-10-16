# Implementation Plan: Repo Quality & Portability Fixes

**Branch**: `claude/repo-analysis-643868` | **Date**: 2026-10-15 | **Spec**: [spec.md](spec.md)

## Summary

Seven ordered changes to a bash-only skill repo, each a user story in the spec: lint-clean scripts, a stronger CI gate (lint, JSON validity, smoke tests), safer `--restore` (confirmation, checksum, path validation, unique names), agent-neutral docs with a Safety contract, a shorter `SKILL.md`, honest probes (`unknown` instead of invented values), and shared helpers with robust JSON. No new dependencies beyond `shellcheck` and Python (any of `python3`/`python`) in CI.

## Technical Context

**Language/Version**: Bash 4+ (Git Bash on Windows runners)
**Primary Dependencies**: shellcheck (CI), `python3` or `python` for JSON validation (resolved in CI with `command -v`)
**Storage**: Files only (`~/.config/system-optimizer/backups`)
**Testing**: Shell smoke tests in `tests/` run by CI (`backup-smoke.sh`, `probe-fallback.sh`, `json-escape.sh`); `tests/windows-baseline.sh` is a manual helper; shellcheck lints `scripts/*.sh` and `tests/*.sh`
**Target Platform**: Linux, Windows (Git Bash)
**Project Type**: Agent skill + CLI scripts
**Performance Goals**: N/A
**Constraints**: Probe output is unchanged except for the deliberate fixes (US6, US7: `unknown` instead of invented values, governor name, concurrency, GPU list); smoke tests must not touch real user config (temp `XDG_CONFIG_HOME`)
**Scale/Scope**: 6 scripts plus `scripts/lib.sh`, 1 workflow, 4 test scripts, 3 docs

## Constitution Check

Constitution v1.0.0 (ratified 2026-10-15) applies.

| Principle | Result |
|---|---|
| I. Hardware-agnostic, zero hardcoding | Pass; US6 removes invented fallback values |
| II. Read-only by default, confirm before modifying | Pass; Safety contract (Phase 2), sudo confirmation (US3), pushes only after asking |
| III. Lint-clean, tested scripts | Pass; lint gate (US1), smoke tests for backup, fallback and JSON escaping |
| IV. Cross-platform, cross-agent | Pass; US4 plus Windows verification on the VM and CI |
| V. Simplicity, single source of truth | Pass; US5 trims docs, US7 removes duplicated helpers; no new dependency beyond Python in CI |
| Development Workflow: refactors separate from behavior fixes | Pass; T051 requires the lib.sh refactor in its own commit(s), verified output-identical before any behavior-fix commit |

Post-design re-check: pass.

## Project Structure

### Documentation (this feature)

```text
specs/001-repo-quality-fixes/
├── plan.md
├── research.md
├── data-model.md
├── quickstart.md
└── tasks.md        # /speckit-tasks
```

### Source Code (repository root)

```text
.github/workflows/ci.yml     # add shellcheck, JSON validation, smoke test
scripts/*.sh                 # lint fixes; backup-config.sh restore hardening; probe honesty
scripts/lib.sh               # NEW (US7): is_windows, run_ps, json_str, parent_disk
tests/windows-baseline.sh    # NEW: run probes on a Windows host (Git Bash) and validate JSON
tests/backup-smoke.sh        # NEW: backup -> list -> restore in temp dir
tests/probe-fallback.sh      # NEW: stubbed PowerShell failure -> probes say unknown
tests/json-escape.sh         # NEW (US7): json_str produces valid JSON
SKILL.md, README.md, references/optimization-guide.md
```

**Structure Decision**: one new `tests/` dir with plain bash scripts (no test framework) and one new sourced helper, `scripts/lib.sh`.

## Work Order

0. **Safety contract** in `SKILL.md` first (documentation only, highest severity).
1. **Lint fixes** (detect-targets.sh ×5, system-probe.sh ×1, target-inspect.sh ×1). Verify with shellcheck and by comparing probe output with the baseline under the T001 `compare.py` rule (same keys; same values except volatile fields).
2. **CI**: add shellcheck step (`-S warning`), loop over probes piping `--json` to `"$PY" -m json.tool` (`PY` = `python3` or `python`), run `tests/backup-smoke.sh`.
3. **Restore hardening** in `backup-config.sh`: all three sudo paths (mkdir, pre-restore copy, final copy) go through one `confirm_sudo` helper; refuse when stdin is not a TTY; validate `Original-Path` (absolute, no `..`, non-empty); make the pre-restore safety copy mandatory (abort if it cannot be written).
4. **Agent neutrality**: replace 53 occurrences of `/skill:` and `<skill_dir>`; add one per-agent invocation table.
5. **Trim SKILL.md** (392 → ≤294 lines): dedupe invocation list vs workflow vs synergies; move detail to the reference guide.
6. **Audit follow-ups** (added after ponytail-audit): Safety contract in `SKILL.md` (step 0); checksum check, quote-safe path parsing and unique backup names in `backup-config.sh` (step 3); probes report `unknown` instead of invented values, concurrency = logical cores, clean GPU list, private backup store (new last step). 
7. **Shared helpers & robust output** (last, behind CI): `scripts/lib.sh` with `is_windows`, `run_ps`, `json_str`; apply `json_str` to JSON text fields; fix NVMe fallback, SSH key rule, NTFS mode label. Windows verified on the Windows 11 VM (VMID 120) and by the Windows CI job after pushing to the fork.

## Risks

- Step 5 can drop a target silently → check with a target-name grep before/after.
- Windows runner may lack a TTY → restore smoke test uses an explicit destination and never triggers sudo.
- `lib.sh` is sourced by path; copying a single script elsewhere breaks it → `lib.sh` exits with a clear message if missing.
- Probe baselines contain volatile fields → comparison ignores a fixed list (see T001).
- Fallback tests stub PowerShell with exported shell functions, which also run on Windows; verified on the VM.
