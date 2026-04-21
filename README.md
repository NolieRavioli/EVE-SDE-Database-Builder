# EVE SDE Database Builder

A Windows Forms (.NET 10, VB.NET) tool that downloads CCP's EVE Online Static Data Export (SDE) and builds it into a database (SQLite, MS Access, MS SQL Server, MySQL, PostgreSQL, CSV or JSON). The SQLite output is the database format consumed by [EVE Isk Per Hour (EVE-IPH)](https://github.com/EVEIPH/EVE-IPH).

This repo contains two projects:

- [EVE SDE Database Builder/EVE SDE Database Builder.vbproj](EVE%20SDE%20Database%20Builder/EVE%20SDE%20Database%20Builder.vbproj) — the main GUI app
- [EVE SDE Database Builder Updater/EVE SDE Database Builder Updater.vbproj](EVE%20SDE%20Database%20Builder%20Updater/EVE%20SDE%20Database%20Builder%20Updater.vbproj) — the auto-updater stub

---

## 1. Prerequisites

- Windows 10/11
- [.NET 10 SDK](https://dotnet.microsoft.com/download/dotnet/10.0) (Desktop runtime included)
- Optional, only if you want non-SQLite outputs:
  - MS Access redistributable (Microsoft Access Database Engine 2016+)
  - A reachable MS SQL / MySQL / PostgreSQL server

Verify the SDK:

```powershell
dotnet --info
```

You should see `.NET SDK 10.0.x` and `Microsoft.WindowsDesktop.App 10.0.x`.

---

## 2. Building

The project file pins `OutputPath` to a path that only exists on the original author's machine, so override it on the command line:

```powershell
cd "EVE SDE Database Builder"
dotnet build -c Release -p:OutputPath="..\Root Directory\net10.0-windows\"
```

Output lands in [Root Directory/net10.0-windows/](Root%20Directory/net10.0-windows/) next to the existing prebuilt binaries.

To build both projects from the solution root:

```powershell
dotnet build "EVE SDE Database Builder.slnx" -c Release `
  -p:OutputPath="..\Root Directory\net10.0-windows\"
```

---

## 3. Running and producing a database

1. Launch [Root Directory/net10.0-windows/EVE SDE Database Builder.exe](Root%20Directory/net10.0-windows/EVE%20SDE%20Database%20Builder.exe).
2. **SDE Download Folder** — pick a working folder, then click **Download SDE**. The app fetches `https://developers.eveonline.com/static-data/tranquility/latest.jsonl`, reads the latest `sde` build number, downloads `eve-online-static-data-<build>-jsonl.zip`, and extracts it into a dated subfolder. The SDE Path is then set automatically.
3. **Database Type** — select **SQLite** (required for EVE-IPH).
4. **Database Name** — e.g. `SDE`. The tool appends `.sqlite`.
5. **Final DB Path** — pick an output folder.
6. **Check All** to import every file.
7. Click **Build Database**. Wait for all rows in the grid to reach 100%.

Result: `<Final DB Path>\<Database Name>.sqlite`.

---

## 4. Moving the database into EVE-IPH

EVE-IPH ships with a SQLite file named `SDE.sqlite` (sometimes `EVEIPH.sqlite` depending on version) sitting in the EVE-IPH program folder.

1. Close EVE-IPH if it is running.
2. Locate the EVE-IPH install directory (typically `C:\Program Files (x86)\EVE-IPH\` or wherever you installed it).
3. Back up the existing `SDE.sqlite` (rename it `SDE.sqlite.bak`).
4. Copy the freshly built file in and rename it to match exactly: `SDE.sqlite`.
5. Start EVE-IPH. On first launch with a new SDE it may prompt to run any required schema patches — accept them.

If EVE-IPH refuses to load the new file, confirm:

- The file name and folder match what EVE-IPH expects (check EVE-IPH's settings dialog → SDE path).
- You built with **all grid items checked** — partial imports will fail EVE-IPH's table validation.

---

## 5. Automation: pulling the latest SDE without manual steps

The app now uses the build number returned from `latest.jsonl` to construct the zip URL, so you no longer have to "point" it at a specific SDE archive — just click **Download SDE** and it always grabs the current build. The fix is in [EVE SDE Database Builder/frmMain.vb](EVE%20SDE%20Database%20Builder/frmMain.vb#L2225-L2233).

Relevant CCP endpoints (per the [EVE Developers SDE docs](https://developers.eveonline.com/docs/services/sde/)):

| Purpose | URL |
| --- | --- |
| Latest build manifest | `https://developers.eveonline.com/static-data/tranquility/latest.jsonl` |
| Versioned data zip | `https://developers.eveonline.com/static-data/tranquility/eve-online-static-data-<build>-<variant>.zip` |
| Changes for a build | `https://developers.eveonline.com/static-data/tranquility/changes/<build>.jsonl` |
| Always-latest JSONL zip | `https://developers.eveonline.com/static-data/eve-online-static-data-latest-jsonl.zip` |
| Always-latest YAML zip | `https://developers.eveonline.com/static-data/eve-online-static-data-latest-yaml.zip` |

Variants are `jsonl` (used by this builder) and `yaml`.

### Fully unattended rebuild (PowerShell)

Save the snippet below as `update-sde.ps1` next to the built exe. It:

1. Reads the current build number from `latest.jsonl`.
2. Compares it against the last build number stored in `LastBuild.txt`.
3. If different, launches the GUI so you can click **Build Database** — or, if you keep the program settings file populated, you can swap the `Start-Process` line for whatever scripted import path you prefer.

```powershell
param(
    [string]$WorkDir = "$PSScriptRoot\sde-cache",
    [string]$ExePath = "$PSScriptRoot\EVE SDE Database Builder.exe"
)

$ErrorActionPreference = 'Stop'
New-Item -ItemType Directory -Force -Path $WorkDir | Out-Null

$latestUrl = 'https://developers.eveonline.com/static-data/tranquility/latest.jsonl'
$latestFile = Join-Path $WorkDir 'latest.jsonl'
Invoke-WebRequest -Uri $latestUrl -OutFile $latestFile -UseBasicParsing

$sdeRecord = Get-Content $latestFile |
    ForEach-Object { $_ | ConvertFrom-Json } |
    Where-Object { $_.'_key' -eq 'sde' } |
    Select-Object -First 1

if (-not $sdeRecord) { throw "No 'sde' record in latest.jsonl" }

$current = $sdeRecord.buildNumber
$stampFile = Join-Path $WorkDir 'LastBuild.txt'
$previous = if (Test-Path $stampFile) { (Get-Content $stampFile).Trim() } else { '' }

if ("$current" -eq $previous) {
    Write-Host "SDE already at build $current. Nothing to do."
    return
}

Write-Host "New SDE build available: $previous -> $current"

# Hand off to the GUI; click 'Download SDE' then 'Build Database'.
Start-Process -FilePath $ExePath -Wait

Set-Content -Path $stampFile -Value $current
Write-Host "Recorded build $current as current."
```

Schedule it with Task Scheduler (daily, at logon, etc.) and you'll be notified — or fully driven through the UI — whenever CCP publishes a new SDE.

> Want a true headless build? The current codebase only exposes the import pipeline through the WinForms UI; a CLI mode would need a small refactor to call the existing `SDE Import Classses\*` types from a console entry point. Open an issue if you want help wiring that up.

---

## 6. Troubleshooting

- **`Failed to download Build Data`** — usually a transient network error or CCP outage. Retry. Confirm `https://developers.eveonline.com/static-data/tranquility/latest.jsonl` opens in a browser.
- **404 on the SDE zip** — out-of-date build cached. Use the **Reset Saved Build Number** menu item to clear `LatestBuild` and re-download.
- **Build fails with `OutputPath` errors** — pass `-p:OutputPath="..\Root Directory\net10.0-windows\"` as shown above.
- **EVE-IPH shows missing tables** — rebuild with **Check All** selected; some EVE-IPH features rely on every SDE table being present.

---

## 7. Project layout

```
EVE SDE Database Builder/         Main WinForms app
  Database Classes/               Per-backend writers (SQLite, MySQL, ...)
  SDE Import Classses/            One importer per SDE JSONL file
  frmMain.vb                      UI + orchestration (Download SDE button is here)
EVE SDE Database Builder Updater/ Auto-updater
Root Directory/net10.0-windows/   Default build output (prebuilt binaries shipped)
```
