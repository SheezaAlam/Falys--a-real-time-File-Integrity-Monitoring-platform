# File Event Attribution — Setup Guide

## What this is

By default, every FALYS agent reports the OS user its own process is
running as (`getpass.getuser()`). That is **not** the same thing as "the
user who touched the file" — a service, a scheduled task, or an agent
running under a shared/service account can all touch files on behalf of
someone else, and the process-level user won't reflect that.

To get a trustworthy answer to "who actually did this", FALYS queries the
operating system's own audit trail:

| Platform | Mechanism            | Requires                                   |
|----------|-----------------------|---------------------------------------------|
| Linux    | `auditd` / `ausearch` | An audit watch rule on the monitored folder |
| Windows  | Security event log (`wevtutil`, event ID 4663) | Object Access auditing + a SACL on the monitored folder |

Neither is enabled by default on a fresh OS install. Until you run the
setup script for your platform, attribution lookups will consistently
return **unverified** — this is expected, not a bug, and the dashboard
labels it as such rather than guessing.

## How to tell if it's working

Every event the agent sends includes:

```json
{
  "user": "maimoona",                  // process-level user (always present)
  "attributed_user": "jdoe",           // OS-audit-verified user, or null
  "attribution_verified": true,        // true only if a real audit record was found
  "attribution_source": "auditd"       // or "security_log", or a reason code if unverified
}
```

In the Event History page, the **User** column shows a green "Verified"
badge with the audited account when `attribution_verified` is true, and an
amber "Unverified" badge with the process-level fallback otherwise.

`attribution_source` when unverified tells you why, e.g.:
- `ausearch_not_installed` / `wevtutil_not_available` — the OS audit tool isn't present
- `no_audit_record` / `no_security_event` — the tool is present, but no matching record was found (most commonly: the setup script below hasn't been run yet for this path)
- `unresolvable_uid` / `unparseable_event` — a record was found but couldn't be fully parsed

## Linux setup

```bash
sudo ./agent/setup_audit_linux.sh /path/to/watched/dir
```

This installs `auditd` if needed, adds a persistent watch rule
(`-w <dir> -p wa -k falys_watch`), and loads it immediately.

Verify manually:
```bash
sudo auditctl -l | grep falys_watch
touch /path/to/watched/dir/test.txt
sudo ausearch -f /path/to/watched/dir/test.txt -ts recent
```

## Windows setup

Run from an **elevated** PowerShell prompt:
```powershell
.\agent\setup_audit_windows.ps1 -WatchDir "C:\watch_falys\monitor"
```

This enables Object Access → File System auditing (`auditpol`) and applies
a SACL to the watched folder so access to files inside it actually gets
logged (auditing policy alone does nothing without a SACL on the object).

Verify manually:
```powershell
auditpol /get /subcategory:"File System"
Get-Acl -Path "C:\watch_falys\monitor" -Audit | Format-List
New-Item -Path "C:\watch_falys\monitor\test.txt" -ItemType File
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4663} -MaxEvents 5
```

## Known limitations (please read before relying on this in production)

- **This was built and unit-tested against realistic *sample* `ausearch`
  and Windows Security-log output**, not a live auditd/Windows install —
  this sandbox can't run either. The parsing logic
  (`agent/attribution.py`) has full unit test coverage
  (`agent/tests/test_attribution.py`) against representative real-world
  formats, including the ambiguous cases (unset `auid`, already-interpreted
  `ausearch -i` output, missing fields, malformed XML). Before relying on
  this in a live deployment, run the setup script on a representative
  endpoint and confirm `attribution_verified: true` shows up for a real
  test event.
- Windows event ID 4663 is only generated when a process opens the file
  with a monitored access right (write, delete, permission change, etc.) —
  a plain directory listing won't trigger it.
- `ausearch` output formatting varies across distributions and auditd
  versions. This module deliberately resolves numeric UIDs itself via the
  local passwd database rather than depending on `ausearch -i`'s inline
  name substitution, specifically to avoid being brittle to that variance
  — but if you see unexpected `unresolvable_uid` results on a distro this
  wasn't tested against, please report the raw `ausearch` output.
- Attribution lookups run on a background worker thread (see
  `agent/agent.py`) so they never block the filesystem watcher, but they
  still add a few hundred ms to a few seconds of latency between a file
  event happening and it appearing in the dashboard.
