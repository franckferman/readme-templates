<div align="center">

# Awesome Windows Internals for Offense

**Curated references on Windows internals, syscalls, EDR bypass, process injection, and kernel exploitation** — for Red Teamers and malware researchers.

<img alt="entries" src="https://img.shields.io/badge/entries-142-brightgreen?style=flat-square">
<img alt="last update" src="https://img.shields.io/badge/updated-2025--06-blue?style=flat-square">
<a href="https://creativecommons.org/publicdomain/zero/1.0/"><img alt="License" src="https://img.shields.io/badge/license-CC0--1.0-lightgrey?style=flat-square"></a>

</div>

> [!NOTE]
> All references are publicly available research. This list is purely educational — it documents existing knowledge, not novel techniques.

---

## Table of Contents

1. [Syscalls and NT API](#1-syscalls-and-nt-api)
2. [Process Injection](#2-process-injection)
3. [EDR Bypass and Hooking](#3-edr-bypass-and-hooking)
4. [Kernel Exploitation](#4-kernel-exploitation)
5. [Memory Forensics and Detection](#5-memory-forensics-and-detection)
6. [Books and Comprehensive References](#6-books-and-comprehensive-references)

---

## 1. Syscalls and NT API

| Resource | Author | Year | Notes |
|---|---|---|---|
| [Hell's Gate](https://github.com/am0nsec/HellsGate) | am0nsec, smelly__vx | 2020 | SSN extraction from live ntdll — original implementation |
| [SysWhispers3](https://github.com/klezVirus/SysWhispers3) | klezVirus | 2022 | Pre-baked syscall stubs generator — supports x86, x64, WoW64 |
| [RecycledGate](https://github.com/thefLink/RecycledGate) | thefLink | 2022 | Combines Hell's Gate + Halo's Gate + Tartarus' Gate |
| [Fresher than fresh: direct syscall resolution](https://0xdarkvortex.dev/hiding-in-plainsight/) | Brute Ratel | 2022 | Fresh-ntdll technique — map disk copy, avoid hooked stubs |
| [Windows x64 syscall table](https://j00ru.vexillium.org/syscalls/nt/64/) | j00ru | ongoing | Per-build SSN reference — essential for version-aware resolvers |

---

## 2. Process Injection

| Resource | Author | Year | Notes |
|---|---|---|---|
| [ThreadlessInject](https://github.com/CCob/ThreadlessInject) | CCob | 2023 | Vtable/function pointer overwrite — no thread creation |
| [Early Bird APC](https://www.cyberark.com/resources/threat-research-blog/masking-malicious-memory-artifacts-part-ii-insights-from-moneta) | CyberArk | 2018 | Queue APC before main thread runs — beats many hooks |
| [Process Injection Techniques](https://www.elastic.co/blog/ten-process-injection-techniques-technical-survey-of-common-and-trending-process) | Elastic | 2019 | Survey of 10 techniques with detection notes |
| [Shellcode injection without VirtualAlloc](https://bruteratel.com/research/feature-update/2021/06/01/PE-Reflection-Long-Dynamic-Chain/) | Brute Ratel | 2021 | PE reflection without explicit VirtualAlloc call |
| [Phantom DLL hollowing](https://www.forrest-orr.net/post/malicious-memory-artifacts-part-i-dll-hollowing) | Forrest Orr | 2019 | Map and hollow a DLL already loaded in the target |

---

## 3. EDR Bypass and Hooking

| Resource | Author | Year | Notes |
|---|---|---|---|
| [Bypassing EDR hooks](https://www.mdsec.co.uk/2019/03/silencing-cylance-a-case-study-in-modern-edrs/) | MDSec | 2019 | Userland hook removal — patch JMP stubs back to syscall |
| [Detecting Cobalt Strike](https://blog.xpnsec.com/how-to-argue-your-shellcode/) | Adam Chester | 2020 | CS watermarks — Beacon config, sleep mask, stack spoofing |
| [PPID Spoofing](https://www.ired.team/offensive-security/defense-evasion/parent-process-id-ppid-spoofing) | ired.team | 2019 | Spawn child with arbitrary parent via `PROC_THREAD_ATTRIBUTE_PARENT_PROCESS` |
| [ACG (Arbitrary Code Guard)](https://docs.microsoft.com/en-us/microsoft-365/security/defender-endpoint/exploit-protection-reference) | Microsoft | — | Process mitigation blocking rw→rx flips — breaks most injectors |
| [ETW-TI deep dive](https://posts.specterops.io/data-source-analysis-and-dynamic-threat-intelligence-with-attck-926a5a2b9cce) | SpecterOps | 2018 | EtwTiLogAllocExVm, EtwTiLogWriteVm — kernel telemetry for allocation events |

---

## 4. Kernel Exploitation

| Resource | Author | Year | Notes |
|---|---|---|---|
| [HackSys Extreme Vulnerable Driver](https://github.com/hacksysteam/HackSysExtremeVulnerableDriver) | HackSys | ongoing | Stack overflow, UAF, pool corruption — lab targets |
| [Windows Kernel Exploitation series](https://www.fuzzysecurity.com/tutorials/expDev/23.html) | FuzzySec | 2017 | SMEP bypass, pool spraying, token stealing — x64 |
| [CVE-2021-34527 (PrintNightmare)](https://github.com/cube0x0/CVE-2021-34527) | cube0x0 | 2021 | LPE via Windows Print Spooler — SYSTEM from low priv |
| [Kernel CFG (kCFG)](https://windows-internals.com/cet-on-windows/) | Alex Ionescu | 2021 | kCFG bypass techniques — ret2dir, indirect call gadgets |

---

## 5. Memory Forensics and Detection

| Resource | Author | Year | Notes |
|---|---|---|---|
| [Moneta](https://github.com/forrest-orr/moneta) | Forrest Orr | 2020 | Live memory scanner — detects shellcode, hollowing, reflective DLL |
| [pe-sieve](https://github.com/hasherezade/pe-sieve) | hasherezade | ongoing | Scan running process memory for injected/modified PE artifacts |
| [Volatility3](https://github.com/volatilityfoundation/volatility3) | Volatility Foundation | ongoing | Memory forensics framework — `malfind`, `dlllist`, `handles` |
| [Detecting process injection with ETW](https://blog.redbluepurple.io/windows-security-research/kernel-tracing-injection-detection) | RedBluePurple | 2020 | ETW + PsSetCreateThreadNotifyRoutine for injection detection |

---

## 6. Books and Comprehensive References

| Resource | Author | Notes |
|---|---|---|
| [Windows Internals 7th ed.](https://docs.microsoft.com/en-us/sysinternals/resources/windows-internals) | Russinovich, Ionescu, Solomon | The reference — processes, memory, I/O, security, virtualization |
| [The Art of Memory Forensics](https://www.memoryanalysis.net/) | Ligh, Case, Levy, Walters | Memory forensics — Windows, Linux, macOS |
| [Offensive Security Engineering](https://www.amazon.com/dp/B0BFSH4TKK) | Red Team Alliance | Tradecraft, C2, EDR bypass from first principles |
| [Windows APT Warfare](https://www.packtpub.com/product/windows-apt-warfare/9781804618110) | Sheng-Hao Ma | Malware internals for defenders — PE loading, shellcode, rootkits |

---

## Contributing

This list is maintained manually. To suggest an addition:
- Open an issue with the resource title, URL, author, year, and a one-line note on why it belongs here
- Or open a PR following the existing table format

Criteria: publicly available, technically accurate, not paywalled (books excepted).

---

## License

[CC0 1.0 Universal](LICENSE) — public domain. No attribution required.

---

## Contact

[![ProtonMail](https://img.shields.io/badge/ProtonMail-8B89CC?style=flat-square&logo=protonmail&logoColor=white)](mailto:contact@franckferman.fr)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-0077B5?style=flat-square&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/franckferman)
[![X](https://img.shields.io/badge/X-000000?style=flat-square&logo=x&logoColor=white)](https://www.twitter.com/franckferman)
