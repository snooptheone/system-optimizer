# system-optimizer Constitution

## Core Principles

### I. Hardware-Agnostic, Zero Hardcoding
Scripts and guidance MUST derive CPU, RAM, disk, GPU and theme values from live probes.
No machine-specific value may be committed. Rationale: the skill must serve both old laptops and
workstations without edits.

### II. Read-Only by Default, Confirm Before Modifying
Probes and audits MUST NOT change the system. Any change to a file or setting MUST be a numbered
proposal that the user approves first, and the file MUST be backed up with `backup-config.sh`
before editing. Privilege escalation (sudo) MUST be explicitly confirmed and MUST NOT run
non-interactively.

### III. Lint-Clean, Tested Scripts
All `scripts/*.sh` MUST pass `shellcheck -S warning` with no findings, use `set -euo pipefail`,
and keep every probe's `--json` output valid JSON. New logic with a branch, parser or
privilege path MUST ship with a smoke test under `tests/`. CI enforces this.

### IV. Cross-Platform and Cross-Agent Portability
Behavior MUST work on Linux and Windows (Git Bash). Docs MUST NOT assume one agent's command
syntax or install path; per-agent invocation lives in a single table and scripts are referenced
by path relative to the skill folder.

### V. Simplicity and Single Source of Truth
Each target or workflow MUST be described once. `SKILL.md` stays concise and detail lives in
`references/`. No new dependency without justification; prefer shell built-ins and tools already
present on CI runners.

## Safety & Portability Constraints

- Metadata read from disk (e.g. backup `.meta`) is untrusted: validate paths (absolute, no `..`)
  before any write.
- CI and tests MUST NOT touch real user configuration; use temporary `XDG_CONFIG_HOME`.
- Claims about untested platforms or agents MUST be labelled as untested in the docs.

## Development Workflow

- Non-trivial changes (multi-file fixes, PRs) are planned with Spec Kit
  (`specify` -> `plan` -> `tasks`) before coding.
- Changes land via pull request; CI (lint, JSON validation, smoke tests) MUST pass on Linux
  and Windows.
- Refactors that alter structure (e.g. splitting `target-inspect.sh`) MUST be separate from
  behavior fixes.

## Governance

This constitution supersedes other practices in this repository. Amendments require a pull
request that updates this file, states the version bump, and notes any impact on `SKILL.md`,
the plan templates or CI. Versioning follows semantic versioning: MAJOR for removed or
redefined principles, MINOR for added principles or materially expanded guidance, PATCH for
wording fixes. Every PR review MUST check compliance with the principles above; deviations
MUST be justified in the plan's Constitution Check.

**Version**: 1.0.0 | **Ratified**: 2026-10-15 | **Last Amended**: 2026-10-15
