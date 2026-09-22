# WSL Panel - One-Click Toggle and RDP Launcher for a WSL2 Desktop

A WinForms tray-style utility for a desktop WSL distro: it boots the distro, brings up `xrdp`,
publishes it on `127.0.0.1` through a `netsh` portproxy, and opens a fullscreen RDP session into
the Linux desktop - then tears all of it down again. Exists because WSL2's NAT address plus VPN
split routes make a direct Windows-to-WSL connection unreliable, and `wsl --shutdown` idling kills
the desktop at the worst time.

**Suggested repo name:** `wsl-rdp-panel`
**Stack:** PowerShell 5.1, WinForms + System.Drawing, `wsl.exe`, `netsh interface portproxy`, `cmdkey`, `mstsc`
**Status:** finished
**Last modified:** 2026-09-11

## What it does

`WSL-Panel.bat` starts `WSL-Panel.ps1` with `-WindowStyle Hidden` so no console window appears. The
window has a status label and three buttons:

- **TURN ON** - `Start-WslStack`: runs the distro as root, `systemctl start xrdp xrdp-sesman`, and
  leaves a `sleep 4294967296` keeper process behind so the VM stays up. Then it polls for up to 30
  seconds: `Set-PortProxy` reads the VM's IP with `hostname -I` and rewrites the v4-to-v4 proxy
  `127.0.0.1:3390 -> <vm ip>:3390` (a no-op if it already points at the current IP), and
  `Test-NetConnection` confirms the port answers.
- **CONNECT DESKTOP** - starts the stack if needed, seeds `cmdkey /generic:TERMSRV/127.0.0.1:3390`
  with saved credentials, and launches `mstsc` on a generated `.rdp` (1920x1080, 32 bpp,
  screen-mode 2 = fullscreen, no credential prompt).
- **TURN OFF** - `wsl --shutdown`, `netsh interface portproxy reset`, and force-stops
  `vmmem*` / `wslhost`.

On every launch `Save-RdpCred` rewrites `%APPDATA%\wslpanel.rdp` and
`%APPDATA%\wslpanel.rdp.cred`.

## Layout

```
WSL-Panel.bat   launcher, no console window
WSL-Panel.ps1   everything else (functions, WinForms UI, RDP file generation)
```

## Running it

Double-click `WSL-Panel.bat`, or run it as administrator (required for `netsh interface portproxy`):

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File .\WSL-Panel.ps1
```

## Notes

- The RDP username and password are hardcoded near the top of `WSL-Panel.ps1` and are written to
  disk in plaintext-ish form on every start: the credential file is protected with
  `ConvertFrom-SecureString -Key (1..16)`, a fixed public key, not DPAPI. Remove both the literals
  and the fixed-key scheme before publishing, or the repo ships you a login.
- Assumes distro `Ubuntu-24.04` with XFCE and `xrdp` on 3390, and `root` as the login user. Change
  `$desktopDistro`, `$rdpHost`, `$rdpPort`, `$rdpUser` at the top for anything else.
- `netsh interface portproxy reset` clears every v4-to-v4 proxy on the machine, not just its own.
