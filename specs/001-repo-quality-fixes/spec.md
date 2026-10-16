# Feature Specification: Repo Quality & Portability Fixes

**Feature Branch**: `claude/repo-analysis-643868`

**Created**: 2026-10-15

**Status**: Draft

**Input**: User description: "Corrigir problemas da análise do repo system-optimizer: CI mais forte, avisos do shellcheck, restore seguro, neutralidade entre agentes, SKILL.md mais enxuto. Fora de escopo: dividir target-inspect.sh."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - CI catches broken scripts and bad output (Priority: P1)

A maintainer opens a pull request. The automated checks reject the change if a script has lint problems, if a probe emits invalid structured (JSON) output, or if the backup tool's basic create/list/restore cycle fails.

**Why this priority**: Without this, every other fix can silently regress.

**Independent Test**: Open a PR that introduces a lint warning, then one that breaks probe JSON; both must fail the checks. A clean PR must pass.

**Acceptance Scenarios**:

1. **Given** a script with a lint warning, **When** the checks run, **Then** the run fails and names the file and warning.
2. **Given** a probe that prints malformed JSON in `--json` mode, **When** the checks run, **Then** the run fails.
3. **Given** a working backup tool, **When** the checks run, **Then** a backup is created, listed and restored byte-identical in a temporary location.

---

### User Story 2 - Scripts are lint-clean (Priority: P1)

A contributor runs the linter over all scripts and gets zero warnings (currently 7: SC2076 ×4, SC2010 ×1, SC2034 ×1, SC2088 ×1), with behavior unchanged.

**Why this priority**: Prerequisite for the lint gate in Story 1.

**Independent Test**: Run the linter at warning level on all scripts: zero findings. Run all probes before/after: same keys and same values, except the volatile fields (ports in listening state, pending updates, systemd counts, swap size).

**Acceptance Scenarios**:

1. **Given** the current scripts, **When** the linter runs, **Then** no warnings are reported.
2. **Given** a fixed script, **When** its probe runs, **Then** output matches pre-fix output under the baseline comparison rule (same keys; same values except volatile fields).

---

### User Story 3 - Restore never escalates privileges silently (Priority: P2)

A user restores a backup. If a normal copy fails and elevation would be needed, the tool asks for explicit confirmation first. The destination read from the backup's metadata is validated before any write.

**Why this priority**: Closes a privileged-write risk from a tampered metadata file.

**Independent Test**: Restore with a metadata file pointing to a protected path: the tool prompts (or refuses in non-interactive mode) and never elevates unprompted.

**Acceptance Scenarios**:

1. **Given** a destination that needs elevation, **When** restoring interactively, **Then** the tool shows the exact destination and asks for confirmation.
2. **Given** the same case non-interactively, **When** restoring, **Then** it exits with an error and writes nothing.
3. **Given** metadata with an empty, relative, or traversal-containing original path, **When** restoring without an explicit destination, **Then** the tool refuses with a clear message.
4. **Given** a backup whose content no longer matches the checksum saved at backup time, or whose checksum record is missing, **When** restoring, **Then** the tool refuses and writes nothing.
5. **Given** an original path containing a quote character, or metadata with no original path, **When** restoring or diffing, **Then** the path is read correctly, or a clear error is shown.
6. **Given** three backups of the same file within one second, **When** listing backups, **Then** three distinct restorable backups exist.
7. **Given** an existing destination file whose safety copy cannot be written (for example a read-only directory and no permission to escalate), **When** restoring, **Then** the tool stops with an error, leaves the destination unchanged, and never prints that a safety copy was saved.

---

### User Story 4 - Skill works across agents (Priority: P2)

A user of Pi, Claude Code or Codex reads the docs and finds invocation instructions that apply to their agent, with no agent-specific path or command syntax assumed.

**Why this priority**: Enables reuse without hand-editing.

**Independent Test**: Search docs for agent-specific invocation syntax and install-dir placeholders: only a single, clearly labelled per-agent note remains.

**Acceptance Scenarios**:

1. **Given** the docs, **When** searching for the Pi-only command prefix, **Then** it appears only in the per-agent invocation note.
2. **Given** script references, **When** read, **Then** they use paths relative to the skill folder.
3. **Given** the main skill file, **When** read, **Then** a "Safety contract" near the top requires read-only audit first, numbered proposals, user approval before any change, and no agent-run privileged commands.

---

### User Story 5 - Leaner skill instructions (Priority: P3)

The agent loads a shorter main instruction file, because repeated content (invocation list, tier workflow, synergies) is stated once and details live in the reference guide.

**Why this priority**: Lowers per-session context cost; no functional change.

**Independent Test**: Compare size before/after; verify every target/domain previously named is still reachable.

**Acceptance Scenarios**:

1. **Given** the trimmed file, **When** compared with the original, **Then** it is at least 25% smaller.
2. **Given** any previously documented target, **When** looked up, **Then** its guidance is still present in the main file or the reference guide.

### User Story 6 - Probes report only what they know (Priority: P3)

A user runs the probes on a machine where a query fails (for example PowerShell is blocked). The output says "unknown" instead of inventing values or claiming protection is on.

**Why this priority**: Wrong facts lead the agent to tune for the wrong machine or call a weak one hardened.

**Independent Test**: Force the Windows query to fail; fields show "unknown" and are never counted as passed in the security score (the denominator stays the same). On Linux with two GPUs the GPU line is cleanly separated.

**Acceptance Scenarios**:

1. **Given** the hardware query fails, **When** the probe runs, **Then** CPU, RAM, disk and GPU fields are "unknown" (`null` for numeric JSON fields), never made-up numbers.
2. **Given** the Defender/UAC query fails, **When** the audit runs, **Then** those checks are labelled "unknown" and are not counted as passed (the score denominator stays the same).
3. **Given** a machine with two GPUs, **When** the probe runs, **Then** they are listed separated by `; ` with no trailing separator.
4. **Given** the probe output, **When** read, **Then** recommended concurrency equals the logical core count, matching the docs.
5. **Given** a first backup on a fresh machine, **When** the store is created, **Then** it is private to the user.

### User Story 7 - Shared helpers and robust output (Priority: P3)

A maintainer fixes Windows detection or JSON escaping once and every script benefits. Probe JSON stays valid even when a value contains quotes or backslashes.

**Why this priority**: Removes five copies of the same helper and a latent invalid-JSON case; the only user-visible changes are the label fixes listed in scenarios 5 and 6.

**Independent Test**: Each shared helper is defined once; a probe fed a value with `"` and `\` still emits valid JSON; probe output matches the baseline.

**Acceptance Scenarios**:

1. **Given** the scripts, **When** searching for the Windows-detection and PowerShell-runner helpers, **Then** each is defined in exactly one file.
2. **Given** a text field containing a quote or backslash, **When** a probe runs with `--json`, **Then** the output parses as JSON.
3. **Given** an NVMe root device and a failed block-device query, **When** the fallback runs, **Then** it finds the correct disk.
4. **Given** SSH keys on Windows and Linux, **When** audited, **Then** the same permission rule applies, and Windows file modes that NTFS cannot express are shown as not applicable.
5. **Given** the Windows power plan is "Balanced", **When** the hardware probe runs, **Then** the governor field reads `Balanced`, without the plan's GUID.
6. **Given** Defender PUA protection is in audit mode (value 2), **When** the audit prints text output, **Then** it says audit mode, not "Disabled".

### Edge Cases

- Linter or Python unavailable on a CI runner → the step fails loudly with a clear message; it never passes silently.
- Probes run on a headless runner with no GPU → JSON must still be valid.
- Backup metadata (`.meta`) or its checksum line missing → restore refuses, because integrity cannot be verified (an explicit destination does not change this).
- Backup/restore smoke test must never touch the runner's real config files.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: CI MUST fail on any lint warning in `scripts/*.sh` and `tests/*.sh`.
- **FR-002**: CI MUST validate that each probe's `--json` output parses as JSON.
- **FR-003**: CI MUST run a backup → list → restore smoke test in a temporary directory and compare contents.
- **FR-004**: All 7 current lint warnings MUST be resolved without changing script behavior, with two intentional exceptions: the Windows Terminal detection condition in `detect-targets.sh` is rewritten so its operator precedence is explicit (today it requires the `Packages` folder even when `wt` is on `PATH`), and the label of the `~/.ssh` permissions line in `target-inspect.sh` is reworded (`Permissions of ~/.ssh directory:`) so it no longer starts with `~`.
- **FR-005**: Restore MUST ask for confirmation, showing the destination, before any elevated copy.
- **FR-006**: Restore MUST refuse elevation in non-interactive sessions.
- **FR-007**: Restore MUST reject a metadata-provided destination that is empty, relative, or contains `..`.
- **FR-008**: Docs MUST replace the Pi-only command prefix with a generic note plus one per-agent table (Pi, Claude Code, Codex).
- **FR-009**: Docs MUST reference scripts by path relative to the skill folder.
- **FR-010**: The main skill file MUST state each target/workflow once, moving detail to the reference guide, and MUST be ≥25% smaller by line count.
- **FR-011**: The main skill file MUST contain a Safety contract: audit read-only, numbered proposals, user approval before any change, agent never runs privileged commands itself.
- **FR-012**: Restore MUST verify the saved SHA256 and refuse on mismatch or when the checksum record is missing. Limit: this detects corruption and edits to the backup file alone; it does not protect against someone who can also edit the `.meta` file.
- **FR-013**: Restore and diff MUST read the original path without mangling quotes, and MUST print a clear error when it is missing.
- **FR-014**: Backups of the same file taken within one second MUST NOT overwrite each other.
- **FR-015**: When a hardware or security query fails, probes MUST output "unknown" for affected fields and never count them as passed (the score denominator stays the same); no invented defaults.
- **FR-016**: Recommended concurrency MUST equal the logical core count (single source of truth with the docs).
- **FR-017**: Multiple GPUs MUST be joined with `; ` and no trailing separator.
- **FR-018**: The central backup store MUST be created readable only by the user, and docs MUST NOT call it "immutable".
- **FR-019**: Each shared helper (Windows detection, PowerShell runner, JSON string escaping) MUST be defined once and reused by all scripts.
- **FR-020**: JSON output MUST remain valid for any text value, including quotes and backslashes.
- **FR-021**: The power-plan field MUST show only the plan name, and PUA protection values 0/1/2 MUST map to Disabled/Enabled/Audit mode in text output.
- **FR-022**: Restore MUST NOT overwrite an existing destination unless its safety copy was written successfully, and MUST NOT report a safety copy that was not written.

### Key Entities

- **Backup**: a copied config file plus its metadata (original path, timestamp, description).
- **Probe**: a script reporting system state in text or JSON.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Linter reports 0 warnings across all scripts (from 7).
- **SC-002**: 100% of probes emit parseable JSON in CI on both Linux and Windows runners.
- **SC-003**: A PR that adds a lint warning or breaks probe JSON is rejected 100% of the time.
- **SC-004**: Restore performs 0 privileged writes without an explicit user confirmation, and 0 overwrites of an existing file without a written safety copy.
- **SC-005**: Main skill file shrinks from 392 to ≤294 lines with no target lost.
- **SC-006**: Zero remaining uses of the Pi-only command prefix outside the per-agent note.
- **SC-007**: Three rapid backups of one file yield three distinct backups.
- **SC-008**: A restore from a backup that was corrupted or edited after creation (with its `.meta` unchanged) is refused 100% of the time.
- **SC-009**: A forced probe-query failure produces 0 invented values and 0 false "secure" results.
- **SC-010**: At least 60 lines of duplicated helper code are removed from the scripts that defined their own copies (measured against the original files), and the Windows CI job passes after the change.

## Assumptions

- Out of scope: splitting `target-inspect.sh` (1,652 lines); tracked as a follow-up.
- Windows runners keep running the bash scripts via Git Bash.
- Skill behavior and recommendations are unchanged; only wording, structure and safety checks change.
- Windows behavior is verified on the user's Windows 11 VM (Proxmox VMID 120, Git Bash 5.2, `powershell.exe`, no `pwsh`/`python3`) via `tests/windows-baseline.sh`, and by the Windows CI job. Baseline on `main` (14e81cc): all 4 probes exit 0 with valid JSON.
- CI finds Python as `python3` or `python`; the Windows runner may only provide `python`.
- Codex invocation is documented from the generic skill format and not tested live.
