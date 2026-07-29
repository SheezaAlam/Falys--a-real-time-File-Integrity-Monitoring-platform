# Agent Deployment: Compiled Binaries (Windows + Linux)

## What changed

Endpoints on **both** Windows and Linux no longer receive Python source
or need Python installed -- the same "just run the file" model as
compiled agents like Wazuh's. The dashboard ships a single compiled
binary per platform:

- **Windows** (`FalysAgent.exe`): applies the required Windows audit
  policy silently, registers and starts itself as a Windows service,
  starts monitoring immediately and on every subsequent boot. Entire
  install: **download, unzip, double-click, click "Yes" on one admin
  prompt.**
- **Linux** (`falys-agent`): applies the required auditd policy,
  installs itself to `/opt/falys-agent`, writes and enables a systemd
  service, starts monitoring immediately and on every subsequent boot.
  Entire install: **download, unzip, `sudo ./falys-agent`.**

No PowerShell, no batch files, no `pip install`, no manual
`python agent.py` on either platform.

The monitoring logic itself (`agent.py`, `attribution.py`,
`discovery.py`) is **unchanged** on both platforms -- it's compiled into
each binary as-is, not rewritten. Communication with the server and the
registration model are also unchanged.

## Architecture

```
agent/agent.py                <- unchanged monitoring logic (main()/stop())

agent/windows_service.py      <- pywin32 Windows Service wrapper +
                                  self-install logic. Compiled into
                                  FalysAgent.exe.
agent/FalysAgent.spec         <- PyInstaller build spec (Windows)
agent/build_exe.ps1           <- one-time build script (Windows, IT-run)

agent/linux_service.py        <- systemd wrapper + self-install logic.
                                  Compiled into falys-agent.
agent/FalysAgentLinux.spec    <- PyInstaller build spec (Linux)
agent/build_exe_linux.sh      <- one-time build script (Linux, IT-run)

server/deployment_builder.py  <- packages the prebuilt binary (either
                                  platform) + a fresh per-deployment
                                  config.json
server/build_output/          <- drop FalysAgent.exe and/or falys-agent
                                  here after building
```

## The one honest limitation (both platforms)

Producing a real compiled binary requires a matching build toolchain --
PyInstaller running ON Windows for `FalysAgent.exe`, and ON Linux for
`falys-agent`. Neither cross-compiles from the other, and the FALYS
server itself might run on either. So, per platform:

- **One-time, manual step**: an admin runs `agent/build_exe.ps1`
  (Windows) or `agent/build_exe_linux.sh` (Linux) once, on a matching
  machine with Python installed, and drops the resulting binary into
  `server/build_output/`.
- **After that**, every "Generate ... Deployment Package" click on the
  dashboard is instant for that platform -- it just zips that same
  compiled binary with a fresh config.json. No compiler runs on the
  server, ever.
- Until that one-time step happens for a given platform, the dashboard
  button instead hands back a "build kit" zip (source + spec + build
  script + instructions) for that platform, so the flow is never a dead
  end.
- The two platforms are independent: you might have a compiled
  `FalysAgent.exe` ready while Linux still serves the build kit, or vice
  versa, depending on which build step an admin has actually run.

An additional Linux-specific note: unlike a statically-linked Go/Rust
binary, a PyInstaller binary is only portable across machines with a
compatible-or-newer glibc than the one it was built on. Build
`falys-agent` on the OLDEST distro version you need to support (see
`build_exe_linux.sh`'s header comment) -- otherwise endpoints running an
older distro can fail to start it with a `GLIBC_2.XX not found` error.
Windows doesn't have an equivalent concern.

## Why a self-installing EXE instead of an MSI

The request's example layout listed either a folder
(`FalysAgent.exe` + `config.json` + `README.txt`) or, preferably, an
`.msi`. We built the folder variant, for a concrete reason: a real
`.msi` requires the WiX Toolset (also Windows-only, also not available
in this build environment to author *and verify*), and would still only
get you to the same end-user experience an already-elevatable
self-installing exe provides on its own -- double-click, one admin
prompt, done.

If your organization specifically needs `.msi` for a software-push tool
(SCCM/Intune/GPO), the cleanest path is to wrap the already-working
`FalysAgent.exe` in a thin WiX or Inno Setup installer whose only job is
to lay the exe + config.json on disk and run `FalysAgent.exe` once --
that's a much smaller, lower-risk project than authoring OS-level
install logic in WiX/Inno directly, and it can be added later without
touching anything in this deployment.

## TLS (cert.pem)

Implemented. The server terminates HTTPS itself (via cheroot; see
DETECTION_ENGINE.md's neighbor doc or server/tls_utils.py for details)
using a self-signed certificate it generates on first run under
server/logs/certs/. There's no public CA on a LAN-only server, so
instead of a browser-style certificate chain, each agent pins the
server's exact certificate: it ships as cert.pem inside the deployment
package, next to config.json, and agent.py verifies every connection
against that specific file (see agent.py's CERT_PATH/_verify_param()).

Practical implications:
- `config.json`'s `"server"` field is `https://...` for every package
  generated since TLS shipped. Redeploy (not just re-copy config.json)
  if you're upgrading an older http:// agent.
- If the server's certificate is ever regenerated -- e.g. deleted, or
  its LAN IP changed enough that the old cert's SAN list no longer
  covers it -- existing agents' pinned cert.pem goes stale and their
  uploads start failing their TLS check (queued locally, not lost;
  see agent.py's existing retry/queue behavior). Redeploy affected
  agents to pick up the new cert.pem.
- Point `FALYS_TLS_CERT` / `FALYS_TLS_KEY` at a real internal-CA-issued
  certificate instead if your organization has one; the self-signed
  path is the zero-config default, not a requirement.
- `FALYS_TLS_ENABLED=0` reverts to the previous plaintext HTTP
  transport (shared API key only, no encryption) -- only for a
  throwaway local test, never a real deployment.

## Testing note

This was built and reviewed without access to a live Windows machine.
Before rolling out to real endpoints: run `build_exe.ps1` and the
resulting `FalysAgent.exe` (both the bare double-click self-install path
and the `install`/`start`/`stop`/`remove` command-line verbs) against a
disposable Windows VM first, and check the Windows Event Log /
`services.msc` to confirm the service registers, starts, survives a
reboot, and that Object Access auditing (`auditpol /get
/subcategory:"File System"`) is actually enabled on the watched folder
afterward.

## Status update — verified end-to-end

**Found and fixed a real bug**: `agent/requirements.txt` and
`agent/requirements-windows.txt` didn't exist, even though
`build_exe.ps1` explicitly required the latter (`pip install -r
requirements-windows.txt`) — running the build script as documented
would have failed immediately with a missing-file error. Both files now
exist with the correct dependency list.

**Tested over real HTTP** (this sandbox has no Windows or a live systemd
target, so this is as far as verification could go here):
- `/api/generate-agent/linux` (legacy Python-source path) — produces a
  complete, correct zip.
- `/api/generate-agent/windows-exe` and `/api/generate-agent/linux-bin` —
  both correctly detect no compiled binary exists yet for that platform
  and fall back to the matching build-kit zip (confirmed via the
  `X-Falys-Prebuilt: false` response header the dashboard reads).

**Not tested** (no Windows or Linux-with-systemd environment available
here to actually run a compiled binary on): the real `build_exe.ps1` /
`build_exe_linux.sh` compiles, and the resulting `windows_service.py` /
`linux_service.py` self-install flows on real machines. Both follow
standard patterns for their platform (pywin32 service framework;
systemd unit + `sudo` self-install) and read correctly, but please do
one real smoke test per platform before rolling out to many endpoints:
run the matching build script once, then run the resulting binary (bare
double-click on Windows, `sudo ./falys-agent` on Linux) and confirm the
full flow -- service/unit registered, running, survives a reboot, and
an event created in the watched folder actually reaches the dashboard.

## New — Linux now ships as a compiled binary too

Added the Linux counterpart to the Windows compiled-exe approach:
`agent/linux_service.py` (self-install + systemd wrapper, entry point
compiled into the binary), `agent/FalysAgentLinux.spec` (PyInstaller
spec), `agent/build_exe_linux.sh` (one-time build script), plus
`generate_linux_package()` in `server/deployment_builder.py` and the
`/api/generate-agent/linux-bin` route serving it. The dashboard's
"Generate & Download" button now uses this for Linux (previously it
used the older Python-source + systemd-install-script path in
`agent_generator.py`, which required `python3`/`pip3` on the endpoint --
that path still exists and still works, but is no longer linked from
the UI). Once `build_exe_linux.sh` has been run once and
`server/build_output/falys-agent` exists, both platforms behave
identically from the dashboard's point of view: pick a platform, click
Generate, get an instant ready-to-run zip, no Python on the endpoint
either way.

## Fix — Error 1053 on service start (Windows)

Real-world testing surfaced `Error 1053: The service did not respond to
the start or control request in a timely fashion` on every SCM-driven
service start (boot, `sc start`, `services.msc`). Root cause: the SCM
launches an installed service's exe with zero command-line arguments —
identical to a user double-clicking `FalysAgent.exe`. The old
`windows_service.py` treated "zero arguments" as always meaning
"double-click", so a real service start was routed into the interactive
installer flow instead of `StartServiceCtrlDispatcher()`; the process
never reported `SERVICE_RUNNING`, so the SCM timed out. A secondary
contributor: `FalysAgent.spec`'s manifest requested
`requireAdministrator`, which can itself interfere with a service's
non-interactive Session-0 SCM launch.

Fixed in both files:
- `windows_service.py` now always attempts
  `StartServiceCtrlDispatcher()` first on a zero-argument launch, and
  only falls back to the double-click installer flow if that call fails
  with `ERROR_FAILED_SERVICE_CONTROLLER_CONNECT` (1063) — the one
  reliable signal that this process was *not* started by the SCM.
- `SvcDoRun` now reports `SERVICE_RUNNING` as its very first action,
  before importing `agent.py` or touching config/network, and all of
  that work now happens on a background thread with retry-with-backoff
  and logging (`service.log` next to the exe, plus the Windows Event
  Log) so a bad `config.json` or an unreachable server leaves the
  service running and retrying instead of exiting.
- `FalysAgent.spec`'s manifest changed from `requireAdministrator` to
  `asInvoker`; elevation for the double-click install path is still
  requested explicitly via `ShellExecuteW(..., "runas", ...)`.

This still needs a real Windows smoke test (a stale/cached service
registration on a VM used for earlier testing may need `sc delete
FalysAgent` + a rebuild before re-testing) — but the specific
zero-argument dispatch bug above is a well-documented class of Error
1053 and this fix addresses it directly, not just its symptom.

Linux has no equivalent failure mode: systemd has no
StartServiceCtrlDispatcher-style handshake to get wrong, so
`linux_service.py`'s `--service-run` path is a straightforward
foreground run with its own retry-with-backoff loop (see
`linux_service.py`'s module docstring for the full comparison).
