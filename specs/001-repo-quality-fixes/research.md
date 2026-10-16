# Research

## Lint findings (shellcheck -S warning, 7 total)
| File:line | Code | Decision |
|---|---|---|
| detect-targets.sh:112 | SC2010 | Replace `ls \| grep -qi` with a `compgen -G`/glob test. Also fixes `A \|\| B && C && return` precedence ambiguity: wrap in explicit `if`. |
| detect-targets.sh:148,159,170,204 | SC2076 | Quoted RHS of `=~` is intentional literal match. Use `== *" $x "*` glob match, same semantics, no warning. |
| system-probe.sh:119 | SC2034 | `boot_cmdline` is unused; delete the line (check `--json` output does not reference it first). |
| target-inspect.sh:299 | SC2088 | The quoted label `"~/.ssh Directory Permissions: …"` starts with `~`. Tested: switching only the `stat` argument to `$HOME/.ssh` does not clear it. Reword the label (`Permissions of ~/.ssh directory: …`) so it does not start with `~`; tested, no warning. Nothing parses the label. A `disable` comment was considered and rejected (permanent suppression for no gain). |

Alternatives: `# shellcheck disable` comments. Rejected (hides real intent, sets a precedent).

## Restore hardening
- Decision: one `confirm_sudo <action> <path>` helper; non-TTY → error exit 1.
- Path validation: reject empty, non-absolute (after `normalize_path`), or any `..` segment. Windows drive paths (`C:/...`) count as absolute.
- Alternative: drop sudo fallback entirely. Rejected: breaks legitimate `/etc/sysctl.d` restores.

## CI
- shellcheck is preinstalled on `ubuntu-latest`; on `windows-latest` run the lint step on Linux only (matrix `if`).
- JSON check: `python3 -m json.tool`, falling back to `python` (the Windows runner and the user's Windows VM may lack `python3`); no `jq` dependency. Previous claim "python3 preinstalled on both runners" was unverified.
- Smoke test uses `XDG_CONFIG_HOME=$(mktemp -d)`.

## Agent neutrality
Invocation table: Pi `/skill:system-optimizer`, Claude Code `/system-optimizer`, Codex mention the skill by name. Codex entry is untested.

## Open item
Spec said "7 warnings: SC2076 ×5, SC2034 ×2, SC2010 ×2, SC2088 ×1". Actual is SC2076 ×4, SC2010 ×1, SC2034 ×1, SC2088 ×1 (my earlier count was wrong). Spec corrected.

## Decisions for the audit follow-ups (US3, US6, US7)

| Topic | Decision | Alternatives rejected |
|---|---|---|
| Restore integrity | Compare backup `sha256sum` with `SHA256:` in `.meta`; refuse on mismatch or missing record. Detects corruption/edits to the backup alone, not an attacker who also edits `.meta`. | Signing backups (needs key management, overkill); cross-check with the `MEMORY.md` ledger (kept as a future option). |
| Unique backup names | `date +%Y%m%d-%H%M%S-%N`; where `%N` is unsupported, loop adding `-1`, `-2` until the name is free. | Lock files (more code). |
| Path from `.meta` | Read with `sed` trim (both ends), never `xargs`; print an error when `Original-Path` is missing. | `xargs -0` (does not fit a text line). |
| Probe failures | Fields become `unknown` (text) or `null` (JSON numbers); `unknown` never counts as passed; score denominator unchanged. | Dropping unknown fields from the score (changes the denominator and hides the gap). |
| Concurrency | `recommended_concurrency` = logical cores for every RAM tier; the docs already say `jobs = nproc`. | Keeping the x2/x4 multipliers (no consumer, contradicts docs). |
| GPU list | `paste -sd';' \| sed 's/;/; /g'`. | `tr '\n' '; '` (maps only one character). |
| Backup store mode | `( umask 077; mkdir -p ... )`; docs say "private", not "immutable". | `chattr +i` (needs root). |
| Shared helpers | `scripts/lib.sh` sourced via `$(dirname "${BASH_SOURCE[0]}")`; fails with a clear message if missing. It starts with `set -euo pipefail` because constitution Principle III covers every `scripts/*.sh`; callers already use the same options. | Copy-paste (current state: 5 copies). |
| JSON escaping | `json_str`: escape `\` first, then `"`. Values are single-line; newlines/control characters are out of scope. | Full JSON encoder in bash (large), `jq` (new dependency). |
| Disk parent | `parent_disk` uses `lsblk -no pkname`, with a regex fallback tested for `nvme0n1p2`, `mmcblk0p1`, `sda1` and `vda`. | Keeping `sed 's/[0-9]*$//'` (wrong for NVMe and MMC). |
| Power plan name | PowerShell returns the raw `powercfg` text; bash keeps the text inside the final parentheses. Testable with a stub. | Trimming in PowerShell (not testable without Windows). |
| PUA labels | 0 Disabled, 1 Enabled, 2 Audit mode, other unknown. | Treating 2 as Disabled (current bug). |

## Windows baseline (Windows 11 VM, Proxmox VMID 120, `main` at 14e81cc)

All four probes exit 0 with valid JSON. `powershell.exe` is present; `pwsh` and `python3` are not. UAC is genuinely off on that VM (`EnableLUA = 0`) and the disk is genuinely reported as SSD, so those two fields are correct, not bugs. Confirmed bugs: the power-plan field includes a GUID, PUA value 2 is printed as "Disabled", and an empty line appears in the hardware text output.
