<div align="center">

# PowerShell-Script-GUI-Template

**Ready-to-use scaffold for PowerShell GUI applications — WinForms, modular event handlers, profile-based logic, no C#/VB.NET required.**

[![PowerShell](https://img.shields.io/badge/PowerShell-5.1%2B-5391FE?style=flat-square&logo=powershell&logoColor=white)]()
[![Platform](https://img.shields.io/badge/platform-Windows-0078D4?style=flat-square&logo=windows&logoColor=white)]()
[![License](https://img.shields.io/badge/license-MIT-lightgrey?style=flat-square)](LICENSE)

</div>

> [!NOTE]
> This is a **template repository**. Use it as a starting point — fork it, rename it, and replace the illustrative content with your own logic. Do not use it as-is.

---

## Table of Contents

1. [What this template provides](#1-what-this-template-provides)
2. [How to use this template](#2-how-to-use-this-template)
3. [What to change](#3-what-to-change)
4. [File structure](#4-file-structure)
5. [Included example](#5-included-example)
6. [Requirements](#6-requirements)
7. [License](#7-license)
8. [Contact](#8-contact)

---

## 1. What this template provides

A complete, working PowerShell GUI application scaffold — not a blank file, not pseudocode. Every part shown is functional and tested:

- **WinForms GUI** — main window, panels, buttons, labels, input fields, dropdown menus wired to real handlers
- **Modular structure** — UI rendering, event logic, and business logic in separate functions; swap components without touching unrelated code
- **Profile system** — runtime selection between multiple operating modes (e.g. minimal / standard / full) via a dropdown or CLI flag
- **Logging** — timestamped output to both the GUI console panel and a `.log` file
- **Error handling** — `try/catch` around all system calls; user-facing messages without stack traces
- **No external dependencies** — pure PowerShell 5.1+, ships with Windows 7 and later; no modules to install

What this template does *not* include: specific business logic, domain-specific forms, or deployment tooling. Those are intentionally left out — this is the frame, not the picture.

---

## 2. How to use this template

**Option A — GitHub template button** *(recommended)*

Click **Use this template → Create a new repository** at the top of this page. GitHub creates a clean copy under your account with no commit history from this repo.

**Option B — Clone and reinitialize**

```bash
git clone https://github.com/franckferman/PowerShell-Script-GUI-Template.git MyNewTool
cd MyNewTool
rm -rf .git
git init
git add .
git commit -m "initial commit from PowerShell-Script-GUI-Template"
```

---

## 3. What to change

After forking, work through this checklist in order:

**Identity**

```
[ ] Rename the repo (GitHub Settings → Repository name)
[ ] Update README.md title, description, and badges
[ ] Update LICENSE if needed (swap author name and year)
[ ] Update .github/ISSUE_TEMPLATE/ with your project name
```

**Code — required changes**

```powershell
# In src/config.ps1 — rename the app and set your version
$Global:AppName    = "MyNewTool"          # was "PowerShell-Script-GUI-Template"
$Global:AppVersion = "1.0.0"
$Global:LogFile    = "$env:TEMP\MyNewTool.log"
```

```powershell
# In src/ui.ps1 — update window title and labels
$Form.Text          = "MyNewTool v$($Global:AppVersion)"
$LabelTitle.Text    = "MyNewTool"
$LabelSubtitle.Text = "Short description of what it does"
```

**Code — optional changes**

| File | What to adapt |
|---|---|
| `src/profiles.ps1` | Rename profiles, add/remove profile entries |
| `src/handlers.ps1` | Replace example button handlers with your logic |
| `src/logger.ps1` | Keep as-is, or add severity levels (INFO/WARN/ERROR) |
| `assets/icon.ico` | Replace with your own `.ico` (16×16 + 32×32 + 48×48 embedded) |

**What NOT to change** (unless you know what you're doing):

- `src/winforms_bootstrap.ps1` — handles DPI awareness, assembly loading, and STA thread setup; changing this breaks rendering on HiDPI screens
- The event loop in `main.ps1` — `[System.Windows.Forms.Application]::Run($Form)` is the correct pattern for blocking execution

---

## 4. File structure

```
PowerShell-Script-GUI-Template/
│
├── main.ps1                  Entry point — loads modules, builds UI, starts event loop
│
├── src/
│   ├── config.ps1            Global variables: AppName, AppVersion, LogFile, profile defaults
│   ├── winforms_bootstrap.ps1 DPI awareness, assembly loading (System.Windows.Forms, System.Drawing)
│   ├── ui.ps1                Form definition, layout, control instantiation
│   ├── profiles.ps1          Profile logic — what changes between minimal/standard/full
│   ├── handlers.ps1          Button/event handler functions — replace these with your logic
│   └── logger.ps1            Write-Log — timestamped output to GUI panel + file
│
├── assets/
│   └── icon.ico              Application icon (replace with your own)
│
├── tests/
│   └── Test-Handlers.ps1     Pester tests for handler functions (pure logic, no GUI)
│
└── docs/
    └── screenshot.png        Screenshot for README — update after customizing
```

**Execution flow:**

```
main.ps1
  → . src/config.ps1            (load globals)
  → . src/winforms_bootstrap.ps1 (load assemblies)
  → . src/logger.ps1            (load Write-Log)
  → . src/profiles.ps1          (load profile logic)
  → . src/handlers.ps1          (register handlers)
  → . src/ui.ps1                (build and show Form)
  → [Application]::Run($Form)   (event loop — blocks until Form closes)
```

---

## 5. Included example

The template ships with a working example: a simple **system information dashboard** that reads hostname, OS version, uptime, CPU, and RAM, and displays them in the GUI panel. It demonstrates:

- Reading system data from WMI (`Get-CimInstance`)
- Populating labels and a text box from script output
- A "Refresh" button wired to a handler
- Profile selection changing the displayed columns
- Logging each refresh to file

**To run the example as-is:**

```powershell
# From an elevated PowerShell prompt (right-click → Run as Administrator if needed)
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
.\main.ps1
```

Or directly:

```powershell
powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\main.ps1
```

---

## 6. Requirements

- Windows 7 or later (Windows 10/11 recommended)
- PowerShell 5.1+ — ships with Windows by default, no install needed
- .NET Framework 4.x — present on all supported Windows versions
- No external modules, no NuGet packages, no admin rights to run (only to install services or write to system paths, depending on your handlers)

**For tests:**

```powershell
Install-Module -Name Pester -Force -Scope CurrentUser
Invoke-Pester tests/
```

---

## License

GNU Affero General Public License v3.0. See [LICENSE](LICENSE) for full terms.

---

## Contact

[![ProtonMail](https://img.shields.io/badge/ProtonMail-8B89CC?style=flat-square&logo=protonmail&logoColor=white)](mailto:contact@franckferman.fr)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-0077B5?style=flat-square&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/franckferman)
[![X](https://img.shields.io/badge/X-000000?style=flat-square&logo=x&logoColor=white)](https://www.twitter.com/franckferman)
