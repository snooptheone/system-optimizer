# Quickstart: validate the feature

```bash
# 1. Lint: expect no output
shellcheck -S warning scripts/*.sh tests/*.sh

PY=$(command -v python3 || command -v python)
# 2. Probe JSON: expect "ok" x4
for p in hardware-probe system-probe detect-targets security-audit; do
  bash scripts/$p.sh --json | "$PY" -m json.tool >/dev/null && echo "$p ok"
done

# 3. Backup smoke test: expect exit 0
bash tests/backup-smoke.sh

# 4. Restore refuses bad metadata, modified backups and non-TTY sudo (in the smoke test)
#    Probes say 'unknown' when PowerShell fails, governor/PUA labels are right, JSON escaping works
bash tests/probe-fallback.sh
bash tests/json-escape.sh

# 5. Docs neutrality: expect only the per-agent table lines
grep -rn '/skill:\|<skill_dir>' SKILL.md README.md references

# 6. Size: expect <= 294
wc -l SKILL.md

# 7. Helper dedup (SC-010): each helper is defined once, in scripts/lib.sh
grep -l '^is_windows()\|^run_ps()\|to_json_arr()\|^json_num()' scripts/*.sh

# 8. Negative checks (SC-003): both commands must exit non-zero
u=$(mktemp) && printf '#!/usr/bin/env bash\nx=1\n' > "$u" && shellcheck -S warning "$u"; echo '{bad' | "$PY" -m json.tool; rm -f "$u"
```

## Windows check (VM)

```bash
# from the repo root; needs SSH to the Proxmox host and the QEMU guest agent running in VM 120
ssh root@pve.desunet.dev 'qm guest exec 120 --pass-stdin 1 -- "C:\Program Files\Git\bin\bash.exe" -l -s' < tests/windows-baseline.sh
# optional: REPO=<url> REF=<branch> inside the script's environment to test a pushed branch
```

Baseline on `main` (14e81cc): all four probes exit 0, JSON valid, governor shows a GUID, concurrency 12 on 6 CPUs, PUA 2 printed as Disabled in text mode (code reading only).
