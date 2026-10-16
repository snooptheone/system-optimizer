# Data Model

**Backup file**: `<BACKUP_ROOT>/<target>/<name>.<YYYYMMDD-HHMMSS-NNNNNNNNN>.bak` (see Additions for uniqueness).
**Metadata** (`<backup>.meta`, text): `Original-Path`, `Target`, `Timestamp`, `SHA256`, `Size`, `Permissions`, `Description`.

Validation (new): `Original-Path` must be non-empty, absolute, free of `..` segments.
No state transitions.

## Additions (audit follow-ups)

**Backup name**: `<name>.<YYYYMMDD-HHMMSS-NNNNNNNNN>.bak` (nanoseconds; numeric suffix `-1`, `-2` if `%N` is unsupported). Two backups in the same second never share a name.

**Metadata integrity**: the `SHA256:` line is required for restore. Missing or mismatching means restore refuses and writes nothing.

**Store permissions**: the central backup directories are created with mode `700`.

**Probe field states**: a value is either measured, `unknown` (text, and JSON strings and booleans) or `null` (JSON numbers). `unknown` never counts as passed in any score.

**JSON strings**: any text value is passed through `json_str` (escapes `\` then `"`); values are single-line.

**Safety copy**: before overwriting an existing destination, restore writes `<dest>.pre-restore-<unique>.bak`; if that fails, restore aborts without touching the destination.
