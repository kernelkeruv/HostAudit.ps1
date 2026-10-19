# HostAudit.ps1 | Windows endpoint integrity triage

PowerShell 7 script to inventory Windows host health, file-system minifilters, and kernel driver metadata. Source: [HostAudit.ps1](HostAudit.ps1). Created for defensive research and authorized assessment.

## What the script collects

| Artifact | Data |
| --- | --- |
| `Summary_*.txt` | DISM CheckHealth result, Defender and Sysmon status, artifact paths |
| `DISM_CheckHealth_*.txt` | DISM component-store health |
| `fltmc_*.txt`, `MiniFilters_*.csv` | Minifilter enumeration and parsed name, instances, altitude, frame |
| `RunningKernelDrivers_*.csv` | Driver state, file path, SHA-256 hash, Authenticode status and publisher |
| `NonMicrosoftDrivers_*.csv` | Driver review subset based on non-Microsoft publisher or invalid signature |
| `Transcript_*.log` | Best-effort PowerShell execution transcript |
| `SFC_*.txt` | Optional SFC output; produced when `-Quick` is absent |

## Important impact and evidence limits

**This utility is not strictly read-only in its default mode.** `sfc /scannow` can repair Windows protected files and change system state. For collection that avoids SFC repairs, **use `-Quick`**. DISM `/CheckHealth` is a health check, not `/RestoreHealth`.

Other caveats: PowerShell auto-elevates when needed; every run writes report files. Driver hashes are SHA-256 fingerprints, not proof that a driver is benign. Missing or invalid signatures alone are not sufficient evidence of compromise. CSV/text exports are derivatives and do not independently establish evidence integrity or chain of custody.

For forensic work, use a disposable or approved host, preserve originals separately, record local and UTC times, and hash the exported reports after a run. Output can contain sensitive computer names and software metadata; do not commit production results to GitHub.

## Requirements

- Windows 10/11 or Windows Server with PowerShell 7+ (`pwsh`)
- Administrative privileges for filter/driver inspection
- A writable report destination

## Safe demonstration (Windows / PowerShell)

Select a **writable working directory** and specify it explicitly; do not rely on the script's fallback location. The example avoids SFC repairs.

```powershell
$WorkDir = 'C:\Forensics\LabOutput'
$Script = (Resolve-Path '.\HostAudit.ps1').Path
# Review script before granting elevation.
pwsh -NoProfile -File $Script -Quick -NoExplorer -OutputRoot $WorkDir
# Inspect only the newest result produced by this invocation.
$Latest = Get-ChildItem -LiteralPath $WorkDir -Directory |
  Sort-Object LastWriteTimeUtc -Descending | Select-Object -First 1
Get-ChildItem -LiteralPath $Latest.FullName -File | Select-Object Name,Length
Get-ChildItem -LiteralPath $Latest.FullName -File |
  Get-FileHash -Algorithm SHA256 |
  Format-Table Path,Hash -AutoSize |
  Out-String -Width 240 | Set-Clipboard
```

**Expected artifacts (schema, not an actual machine result):** `Summary_*.txt`, `DISM_CheckHealth_*.txt`, `fltmc_*.txt`, `MiniFilters_*.csv`, `RunningKernelDrivers_*.csv`, `NonMicrosoftDrivers_*.csv`. The last command places file hashes on the Windows clipboard for verification.

## Review and verification

1. Read `Summary_*.txt` for DISM/Defender/Sysmon status and any collection errors.
2. Open `RunningKernelDrivers_*.csv`; compare `SignatureStatus`, `Publisher`, `SHA256` and `Path` to trusted software inventory.
3. Inspect `MiniFilters_*.csv` and validate unfamiliar filters against known hardware/security software.
4. Record the command and tool versions (`$PSVersionTable.PSVersion`, `Get-Command fltmc`).
5. Re-run only on an authorized lab system; compare hashes and differences rather than treating all changes as threats.

## Validation status

This repository documents a source script, not a certified forensic product. No CI test here demonstrates successful collection on a Windows host. A future Windows integration test should verify CSV headers, `-Quick` behavior, nonzero failure handling and traceability of the execution transcript. Do not advertise the script as forensically validated until such tests exist.

MIT licensed.
