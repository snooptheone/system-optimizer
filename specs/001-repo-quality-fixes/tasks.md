# Tasks: Repo Quality & Portability Fixes

**Input**: [spec.md](spec.md), [plan.md](plan.md), [research.md](research.md), [data-model.md](data-model.md), [quickstart.md](quickstart.md)
**Tests**: Test tasks are included where the constitution (Principle III) requires them: backup/restore (US1, US3), probe fallbacks (US6, US7) and JSON escaping (US7).

## Format: `- [ ] [ID] [P?] [Story] Description with file path`

## Phase 1: Setup

- [X] T001 Capture baseline in a directory made with `mktemp -d` (note its path in the commit/PR notes, do not use a fixed `/tmp` name): save `bash scripts/<probe>.sh --json` output of the 4 probes, run `wc -l SKILL.md` (392), save the list of target names found in `SKILL.md` with `grep -oE`, write a throwaway `compare.py` in that directory that compares two probe JSON files by (a) identical key sets and (b) identical values for every key except the volatile ones (`network.*`, `package_security.pending_security_updates`, `systemd_security_audit.*`, `ram.swap_gb`) — this is the comparison rule every later "matches the T001 baseline" step uses; and record the total line count of `scripts/*.sh` (`cat scripts/*.sh | wc -l`) for later comparison
- [X] T002 Create `tests/backup-smoke.sh` skeleton (shebang, `set -euo pipefail`, `mktemp -d` for `XDG_CONFIG_HOME`, trap cleanup)

**Checkpoint**: baseline recorded, test file exists.

## Phase 2: Foundational

The highest-severity audit finding is a documentation gap, so it goes first. It is labelled US4 because it satisfies that story's scenario 3.

- [X] T003 [US4] Add a "Safety contract" section near the top of `SKILL.md` (before Invocation): audit read-only first; present numbered proposals; apply only user-approved items; back up with `scripts/backup-config.sh` first; never run sudo/privileged commands, hand them to the user; remove or reword any line that says to apply changes without approval, including Full Optimization

**Checkpoint**: `SKILL.md` requires user approval before any change.

## Phase 3: User Story 2 - Scripts are lint-clean (P1)

**Goal**: `shellcheck -S warning scripts/*.sh` reports 0 findings, behavior unchanged.
**Independent Test**: shellcheck is silent; probe `--json` output matches the T001 baseline under the `compare.py` rule (same keys; same values except the volatile keys).

- [X] T004 [US2] In `scripts/detect-targets.sh:112` replace `ls ... | grep -qi "WindowsTerminal"` (SC2010) with a glob test (`compgen -G "$local_app_data/Packages/*WindowsTerminal*"` under `shopt -s nocaseglob` to keep the old case-insensitive match; restore the option afterwards) and rewrite the `A || B && C && return 0` chain as an explicit `if` so precedence is unambiguous (intentional behavior fix, see FR-004)
- [X] T005 [US2] In `scripts/detect-targets.sh` lines 148, 159, 170, 204 replace `[[ ! " ${arr[*]:-} " =~ " ${x} " ]]` (SC2076) with `[[ " ${arr[*]:-} " != *" ${x} "* ]]`
- [X] T006 [P] [US2] In `scripts/system-probe.sh:119` delete the unused `boot_cmdline=` line (SC2034) after confirming with `grep -n boot_cmdline scripts/system-probe.sh` that it is unused
- [X] T007 [P] [US2] In `scripts/target-inspect.sh:299` reword the quoted label so it no longer starts with `~` (SC2088 is about that label, not the `stat` argument; tested: changing only the `stat` argument to `"$HOME/.ssh"` does NOT clear it). New line: `echo "Permissions of ~/.ssh directory: $(stat -c '%a' "$HOME/.ssh" 2>/dev/null || echo absent)"`; no `shellcheck disable` comment. Nothing parses this label (checked with grep); record it as a text exception in FR-004
- [X] T008 [US2] Run `shellcheck -S warning scripts/*.sh` (expect no output) and compare the 4 probes' `--json` output with the T001 baseline using its `compare.py` rule

**Checkpoint**: SC-001 met.

## Phase 4: User Story 1 - CI catches broken scripts and bad output (P1)

**Goal**: CI fails on lint warnings, invalid probe JSON, or a broken backup cycle.
**Independent Test**: temporarily add `x=1` unused var to a script on a scratch branch: CI fails; revert: passes.
**Depends on**: US2 (otherwise the lint gate is red from day one).

- [X] T009 [US1] Implement `tests/backup-smoke.sh`: with temp `XDG_CONFIG_HOME`, create a file, run `scripts/backup-config.sh <file> smoke "test"`, run `--list smoke`, modify the file, run `--restore <bak> <file>`, assert contents equal the original; exit 1 on any mismatch
- [X] T010 [US1] In `.github/workflows/ci.yml` add a Linux-only step (`if: runner.os == 'Linux'`) running `shellcheck -S warning scripts/*.sh tests/*.sh`
- [X] T011 [US1] In `.github/workflows/ci.yml` replace the 4 plain probe invocations with a loop that pipes each `--json` output to `"$PY" -m json.tool > /dev/null` where `PY=$(command -v python3 || command -v python)` is defined once at the top of the job; the step fails loudly if neither exists and also fails when any probe's JSON is invalid
- [X] T012 [US1] In `.github/workflows/ci.yml` add a step running `bash tests/backup-smoke.sh` on both OSes; reuse the `PY` variable defined in the JSON-validation step (same job)
- [X] T013 [US1] Validate the workflow locally: run the three new commands from the YAML by hand and confirm exit 0; negative checks: add `x=1` as an unused variable to a copy of one script and confirm `shellcheck -S warning` exits non-zero, and pipe `echo '{bad'` through `"$PY" -m json.tool` and confirm it exits non-zero; discard both

**Checkpoint**: SC-002, SC-003 met once pushed and CI is green.

## Phase 5: User Story 3 - Restore never escalates privileges silently (P2)

**Goal**: sudo only after explicit confirmation; bad metadata paths refused.
**Independent Test**: `.meta` with `Original-Path: ../../etc/x` is rejected; restore to a root-owned dir without TTY exits non-zero and writes nothing.

- [X] T014 [US3] In `scripts/backup-config.sh` add helper `confirm_sudo <action> <path>`: returns 1 with an error message if stdin is not a TTY; otherwise prints the exact action and path and reads `y/N`
- [X] T015 [US3] In `scripts/backup-config.sh` restore mode route all three sudo paths (`sudo mkdir -p`, pre-restore `sudo cp`, final `sudo cp`) through `confirm_sudo`; on refusal exit 1 without writing
- [X] T016 [US3] In `scripts/backup-config.sh` restore mode make the pre-restore safety copy mandatory: today (lines ~129-134) `cp ... || (sudo cp ...) || true` swallows a failure and `echo "Saved safety copy ..."` prints unconditionally, so restoring into a read-only directory overwrites the file and claims a copy that does not exist (reproduced: `app.conf` content `changed` was lost, no `.pre-restore-*.bak` created). Print the "Saved" line only after the copy succeeded; if it fails (and the user declines or cannot escalate through `confirm_sudo`), exit 1 before touching the destination
- [X] T017 [US3] In `scripts/backup-config.sh` add `validate_dest_path` and call it on the `Original-Path` read from `.meta` (restore and diff modes): reject empty, non-absolute (POSIX `/…` or Windows `X:/…` after `normalize_path`), or any `..` segment; message and exit 1
- [X] T018 [US3] In `scripts/backup-config.sh` restore mode, before any copy compute `sha256sum` of the backup and compare with the `SHA256:` line in the `.meta`; on mismatch, or when the `.meta` or the `SHA256:` line is missing, empty or `unknown` (backup creation records `unknown` when `sha256sum` was unavailable), print an error and exit 1 without writing. Document in the script header that this detects corruption/edits to the backup but not an attacker who also edits the `.meta`
- [X] T019 [US3] In `scripts/backup-config.sh` replace `| xargs` path trimming (restore and diff modes) with a two-sided trim that keeps quotes and backslashes (`raw_orig=$(printf '%s' "$raw_orig" | sed -E 's/^[[:space:]]+//; s/[[:space:]]+$//')`), and print `Error: no Original-Path in <meta>` instead of exiting silently when the field is missing
- [X] T020 [US3] In `scripts/backup-config.sh` make the backup name unique within one second: use `date +%Y%m%d-%H%M%S-%N` (fallback loop adding `-1`, `-2` while the file exists, for platforms without `%N`); apply the same unique-name routine to the pre-restore safety copy (`${dest_file}.pre-restore-<ts>.bak`), which today uses a seconds-only timestamp and is overwritten by a second restore in the same second
- [X] T021 [US3] Extend `tests/backup-smoke.sh`: assert restore with a `.meta` containing `Original-Path: ../x` exits non-zero, and that restore to an unwritable dir under non-TTY exits non-zero without creating the file; the unwritable-dir case needs a user that cannot write there, so it prints an explicit `SKIP: <reason>` (and counts it) on Windows/Git Bash (NTFS ignores `chmod`) and when `id -u` is 0, instead of failing or silently passing
- [X] T022 [US3] Extend `tests/backup-smoke.sh`: (a) modified backup content, or a deleted `.meta`, makes restore exit non-zero and leaves the file unchanged; (b) a file named `it's a file.conf` backs up and restores through `.meta`; (c) three backups in a row give three distinct `.bak` files; (d) with the destination directory made read-only and `sudo` absent from `PATH` (use a restricted `PATH` of symlinks to the needed tools), restore exits non-zero, the destination keeps its content and no "Saved safety copy" line is printed (SKIP on Windows/root as above)
- [X] T023 [US3] Re-run `shellcheck -S warning scripts/backup-config.sh` and `bash tests/backup-smoke.sh`

**Checkpoint**: SC-004 met.

## Phase 6: User Story 4 - Skill works across agents (P2)

**Goal**: one per-agent invocation table; relative script paths.
**Independent Test**: `grep -rn '/skill:\|<skill_dir>' SKILL.md README.md references` returns only the table lines.

- [X] T024 [US4] Add an "Invocation by agent" table to `SKILL.md` (Pi `/skill:system-optimizer`, Claude Code `/system-optimizer`, Codex by naming the skill; mark Codex as untested) and a one-line "examples use Pi syntax; adjust per table" note
- [X] T025 [P] [US4] In `SKILL.md` replace every `/skill:system-optimizer` in examples with `system-optimizer` and every `<skill_dir>/scripts/` with `scripts/`
- [X] T026 [P] [US4] Apply the same replacements in `references/optimization-guide.md` and `README.md`; in `README.md` change "for Pi" to list Pi, Claude Code, Codex
- [X] T027 [US4] Run the grep from Independent Test; expect only table lines

**Checkpoint**: SC-006 met.

## Phase 7: User Story 5 - Leaner skill instructions (P3)

**Goal**: `SKILL.md` ≤ 294 lines with every target still reachable.
**Independent Test**: line count ≤ 294; every target name from the T001 baseline still found in `SKILL.md` or `references/`.
**Depends on**: US4 (same file; do after to avoid conflicts).

- [X] T028 [US5] In `SKILL.md` collapse the long Invocation Modes example list into a compact target-by-domain table (one row per domain, targets comma-separated)
- [X] T029 [US5] In `SKILL.md` shorten the Tier 1–5 workflow to one-line summaries per tool and move the removed per-tool detail into matching sections of `references/optimization-guide.md` (append, do not duplicate existing text)
- [X] T030 [US5] In `SKILL.md` merge the "Cross-Tool Synergies" bullets into one table (tools, effect) and drop wording that repeats the tiers
- [X] T031 [US5] Verify: `wc -l SKILL.md` ≤ 294, and for each target name in the T001 list `grep -l` finds it in `SKILL.md` or `references/optimization-guide.md`

**Checkpoint**: SC-005 met.

## Phase 8: User Story 6 - Probes report only what they know (P3)

**Goal**: no invented values; consistent concurrency; clean GPU list; private backup store.
**Independent Test**: simulate a failed PowerShell query (`run_ps() { return 1; }` via a test shim) and check output shows `unknown`; on Linux run with two fake GPU lines.
**Depends on**: US5 (docs wording edit touches `SKILL.md`).

- [X] T032 [US6] In `scripts/hardware-probe.sh` Windows branch: replace the whole-query fallback block (8.00 GB, SSD, 4 cores, "Generic Display Adapter") with `unknown` values, AND remove every per-field `|| echo "<default>"` invented value that fires when a single field is missing from the PowerShell output (for example `ram_total_gb ... || echo "16.0"`, `cpu_cores ... || echo "1"`, `rotational ... || echo "SSD"`, `mem_compression ... || echo "true"`), plus the PowerShell-side `else` defaults (`Cores ... else 1`, `RAMTotalGB`); make numeric JSON fields `null` and the tier/concurrency logic tolerate `null`
- [X] T033 [US6] In `scripts/security-audit.sh` Windows block, make all eight query-derived values `unknown` instead of invented defaults: `fw_summary`, `fw_all_active`, `realtime_prot`, `av_enabled`, `pua_prot`, `uac_enabled`, `exec_policy`, `sshd_status` (today lines 71-78 each end in `|| echo "<secure-looking default>"`), and change the PowerShell-side fallbacks (`else { $true }`, `else { 0 }`, `else { 5 }` when `Get-MpComputerStatus`/`Get-MpPreference` fail, which happens on machines with a third-party antivirus) to `$null` so the bash side reports `unknown`; `unknown` is never counted as passed but the denominator stays 6, and the text output labels it "unknown"
- [X] T034 [US6] In `scripts/hardware-probe.sh` set `safe_concurrency=$cpu_cores` for all RAM tiers (keep `memory_tier`); drop the ×2/×4 branches
- [X] T035 [US6] In `scripts/hardware-probe.sh` replace `tr '\n' '; ' | sed 's/; $//'` for GPUs with `paste -sd';' | sed 's/;/; /g'`
- [X] T036 [P] [US6] In `scripts/backup-config.sh` create the store with `( umask 077; mkdir -p "$target_backup_dir" )`; in `SKILL.md` and `references/optimization-guide.md` change "Central immutable store" to "Central backup store (private)"
- [X] T037 [US6] In `scripts/system-probe.sh` Windows branch: apply the same rule to the system probe (found while testing; FR-015 covers every probe): in JSON mode print `{"error": "powershell unavailable or returned no data"}` instead of an empty line when PowerShell returns nothing; replace the text-mode `|| echo "<guess>"` defaults (`Windows 11`, `Balanced`, `C: (NTFS)`, TRIM `Enabled`, 8.3 `Disabled`, compression/FastStartup `true`, TCP `normal`, counts `0`, lists `None`) with `unknown`; make the PowerShell side return `$null`/`unknown` when a query yields nothing (TRIM and 8.3 states, MemoryCompression, FastStartup, TCP); `tests/probe-fallback.sh` covers it
- [X] T038 [US6] Create `tests/probe-fallback.sh`: stub `powershell.exe`, `powershell` and `pwsh` as exported shell functions (`powershell.exe() { return 1; }; export -f powershell.exe`, not files, so it also works on Windows where a script named `*.exe` may not execute) and run the probes with `WINDIR=C:\\Windows`. Prototype on `main` shows the failing state clearly (hardware: 4 cores, 8.0 GB, "Generic Display Adapter", SSD; audit: score 6/6 with Defender, UAC and firewall all reported secure), so the test goes red before the fix. Assert (a) `hardware-probe.sh --json` is valid JSON and has none of `8.0`, `Generic`, `"SSD"`, `4` cores invented values (fields are `unknown`/`null`); (b) `security-audit.sh --json` and text output show `unknown` for all eight values of the audit task, the score is not 6/6, and nothing reads `Enabled`/`true`/`Active`; (c) a second case where the stub prints canned PowerShell JSON with the Defender fields `null` (simulating a failing `Get-MpComputerStatus`) still gives `unknown`, not `true`; wire the script into `.github/workflows/ci.yml` on both OSes
- [X] T039 [US6] Run `shellcheck -S warning scripts/*.sh`, the 4 probes' `--json` through `"$PY" -m json.tool` (`PY=$(command -v python3 || command -v python)`), `bash tests/backup-smoke.sh` and `bash tests/probe-fallback.sh`; compare probe output with the T001 baseline using its `compare.py` rule

**Checkpoint**: SC-009 met.

## Phase 9: User Story 7 - Shared helpers and robust output (P3)

**Goal**: one definition per helper; valid JSON for any text value; small probe inconsistencies fixed.
**Independent Test**: `grep -c 'is_windows()' scripts/*.sh` shows one definition in total; probes match the T001 baseline; JSON test passes; Windows CI job green.
**Depends on**: US6 (same scripts) and US1 (CI protects the refactor).

- [X] T040 [US7] Create `scripts/lib.sh` containing `is_windows`, `run_ps` (copied from `scripts/hardware-probe.sh`) and `json_str` (escapes `\` then `"` with `sed`); start the file with `# shellcheck shell=bash` (no shebang, it is sourced) followed by `set -euo pipefail` (the constitution requires it in every `scripts/*.sh`; every caller already sets the same options, so sourcing it changes nothing), and exit with a clear message if sourced standalone incorrectly
- [X] T041 [US7] In `scripts/hardware-probe.sh`, `scripts/system-probe.sh`, `scripts/detect-targets.sh`, `scripts/security-audit.sh` delete the local `is_windows`/`run_ps` and add `source "$(dirname "${BASH_SOURCE[0]}")/lib.sh"`
- [X] T042 [US7] Do the same in `scripts/target-inspect.sh` (separate task: large file, compare its output with the T001 baseline (`compare.py` rule) after)
- [X] T043 [US7] Wrap free-text values in `json_str` in the JSON output of `scripts/hardware-probe.sh` (CPU model, GPU, OS name, kernel, root device), `scripts/security-audit.sh` (firewall name, `to_json_arr` items, SSH key names), `scripts/system-probe.sh` and `scripts/detect-targets.sh` for any interpolated text
- [X] T044 [US7] Add `tests/json-escape.sh`: source `scripts/lib.sh`, pass `a"b\c` through `json_str`, wrap in quotes and assert `python3`/`python` (`command -v`) `-m json.tool` accepts it; wire it into `.github/workflows/ci.yml` on both OSes
- [X] T045 [US7] Add `parent_disk <device>` to `scripts/lib.sh` (handles `/dev/nvme0n1p2`, `/dev/mmcblk0p1`, `/dev/sda1`, `/dev/mapper/*` via `lsblk -no pkname` with this tested regex fallback after stripping `/dev/`: `sed -E 's#^(nvme[0-9]+n[0-9]+|mmcblk[0-9]+)p[0-9]+$#\1#; s#^([sv]d[a-z]+)[0-9]+$#\1#'`; do not use the sed `t` command, it skips the second rule after the `/dev/` strip) and use it in `scripts/hardware-probe.sh` instead of `sed 's/[0-9]*$//'`; add cases to `tests/probe-fallback.sh` asserting `parent_disk /dev/nvme0n1p2` -> `nvme0n1`, `/dev/mmcblk0p1` -> `mmcblk0`, `/dev/sda1` -> `sda` and `/dev/vda` -> `vda`
- [X] T046 [US7] In `scripts/security-audit.sh` make the Windows SSH key rule match Linux (`600`/`400` only); on Windows where `stat -c %a` is not meaningful, print `n/a (NTFS)` for directory and key modes and skip the key warning
- [X] T047 [US7] In `scripts/hardware-probe.sh` stop trimming the power plan in PowerShell: return the raw `powercfg` text and extract the name in bash (`sed -E 's/.*\(([^)]*)\)[[:space:]]*$/\1/'`), so `governor` is `Balanced` and not `<GUID>  (Balanced)`; also drop the empty line the text output prints when zram is inactive (the `$(if [[ "$zram_active" == true ]] ...)` line in the heredoc); add a case to `tests/probe-fallback.sh` where the stub prints canned PowerShell JSON with `"PowerScheme":"381b4222-f694-41f0-9685-ff5bb260df2e  (Balanced)"` and the JSON governor is `Balanced`
- [X] T048 [US7] In `scripts/security-audit.sh` text output map `pua_prot` 0 -> Disabled (recommend `Set-MpPreference -PUAProtection 1`), 1 -> Enabled (Secure), 2 -> Audit mode (not blocking; recommend 1), anything else -> unknown; add a case to `tests/probe-fallback.sh` where the stub prints canned JSON with `"PUAProtection":2` and the text output contains `Audit mode` and not `Disabled`
- [X] T049 [US7] Locally run `shellcheck -S warning scripts/*.sh`, the 4 probes through `"$PY" -m json.tool`, `bash tests/backup-smoke.sh`, `bash tests/json-escape.sh`, `bash tests/probe-fallback.sh`, compare probe output with the T001 baseline using its `compare.py` rule, and confirm each shared helper is defined once (`grep -c '^is_windows()' scripts/*.sh`) and that at least 60 duplicated helper lines were removed from the five scripts (measured from `git show HEAD:scripts/<f>.sh`: 74 lines of `is_windows`/`run_ps` plus the JSON helpers; total `scripts/*.sh` length is not a valid check because earlier stories added code)
- [ ] T050 [US7] Ask the user before pushing; then push the branch to the `fork` remote only. Then on the Windows 11 VM (Proxmox VMID 120) run `REPO=<fork url> REF=<branch> bash -s < tests/windows-baseline.sh` through `qm guest exec 120 --pass-stdin 1`, then run `tests/backup-smoke.sh`, `tests/probe-fallback.sh` and `tests/json-escape.sh` from the cloned branch on the VM (Git Bash) and confirm the SKIP lines are the only skips and confirm: 4 probes exit 0 with valid JSON, `governor` is `Balanced`, `recommended_concurrency` equals `cores`, and the Windows CI job is green. Also push a throwaway branch `scratch/ci-negative` that adds an unused variable to one script, confirm CI fails on lint (SC-003), then delete that branch from the fork (ask first)

**Checkpoint**: SC-010 met.

## Phase 10: Polish

- [X] T051 Run the full `quickstart.md` command list and confirm each expected outcome
- [X] T052 Remove the "Sync Impact Report" comment block at the top of `.specify/memory/constitution.md`
- [X] T053 Review `git status`; commit in logical groups (lint, CI, restore, docs, probes, shared helpers, tests, spec-kit scaffold) without attribution lines. Per constitution (Development Workflow) the structural refactor (US7 shared helpers: lib.sh extraction and sourcing) MUST be its own commit(s), separate from behavior fixes (US7 NVMe, SSH rule, governor, PUA; US3; US6); order them so the refactor commit is verified output-identical with the T001 `compare.py` rule before any behavior-fix commit

## Dependencies & Order

- The Safety contract task (Phase 2) runs before everything that edits `SKILL.md`.
- T001 → T002 → everything.
- US2 (Phase 3) → US1 (Phase 4): the lint gate needs a clean tree.
- US3 after T009 exists (it extends the smoke test), so after US1.
- US4 → US5 (same file).
- US3 and US4 are independent of each other.
- US6 after US5 (shares `SKILL.md`); its script edits are otherwise independent.
- US7 last: needs US6 (same probe scripts) and the CI from US1 as its safety net.

## Parallel Opportunities

- T006 and T007 (different files). T004 and T005 share `detect-targets.sh`, so they run sequentially.
- T025 and T026.

## Implementation Strategy

MVP = US2 + US1 (lint-clean tree plus CI gate): every later change is then protected. Then US3, US4, US5 in order. Commit and push after each phase so CI validates incrementally.

## Summary

53 tasks. Setup: 2, Foundational: 1, US2: 5, US1: 5, US3: 10, US4: 4, US5: 4, US6: 8, US7: 11, Polish: 3.
