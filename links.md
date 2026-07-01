<div align="center">

# Windows Offensive Research — Links

**Resources, papers, and tools collected during Red Team research on Windows internals, EDR bypass, and Active Directory attacks.**

<img alt="Last update" src="https://img.shields.io/badge/updated-2025--06-blue?style=flat-square">
<a href="https://creativecommons.org/publicdomain/zero/1.0/"><img alt="License" src="https://img.shields.io/badge/license-CC0--1.0-lightgrey?style=flat-square"></a>

</div>

---

## Process Injection

- [Hell's Gate](https://github.com/am0nsec/HellsGate) — direct syscall via SSN extraction from live ntdll
- [SysWhispers3](https://github.com/klezVirus/SysWhispers3) — pre-baked syscall stubs, x86/x64/WoW64
- [ThreadlessInject](https://github.com/CCob/ThreadlessInject) — vtable overwrite, no thread creation
- [Early Bird APC (CyberArk)](https://www.cyberark.com/resources/threat-research-blog/masking-malicious-memory-artifacts-part-ii-insights-from-moneta) — queue APC before main thread starts
- [Phantom DLL hollowing](https://www.forrest-orr.net/post/malicious-memory-artifacts-part-i-dll-hollowing) — hollow a DLL already loaded in target

## EDR Bypass

- [Silencing Cylance (MDSec)](https://www.mdsec.co.uk/2019/03/silencing-cylance-a-case-study-in-modern-edrs/) — userland hook removal
- [PPID Spoofing](https://www.ired.team/offensive-security/defense-evasion/parent-process-id-ppid-spoofing) — arbitrary parent process via `PROC_THREAD_ATTRIBUTE_PARENT_PROCESS`
- [ACG deep dive (Microsoft)](https://docs.microsoft.com/en-us/microsoft-365/security/defender-endpoint/exploit-protection-reference) — Arbitrary Code Guard — what breaks, what survives
- [ETW-TI explained (SpecterOps)](https://posts.specterops.io/data-source-analysis-and-dynamic-threat-intelligence-with-attck-926a5a2b9cce) — EtwTiLogAllocExVm, EtwTiLogWriteVm — what EDRs see at kernel level

## Active Directory

- [The Hacker Recipes](https://www.thehacker.recipes/) — ShutdownRepo — AD attack techniques, maintained and linked
- [Shadow Credentials](https://posts.specterops.io/shadow-credentials-abusing-key-trust-account-mapping-for-takeover-8ee1a53566ab) — msDS-KeyCredentialLink write → PKINIT → TGT
- [Targeted Kerberoasting](https://www.thehacker.recipes/ad/movement/access-controls/targeted-kerberoasting) — SPN write → TGS request → offline crack
- [RBCD abuse](https://www.thehacker.recipes/ad/movement/kerberos/resource-based-constrained-delegations) — msDS-AllowedToActOnBehalfOfOtherIdentity → S4U2Proxy
- [AdminSDHolder persistence](https://adsecurity.org/?p=1906) — Sean Metcalf — SDProp mechanism, how it works, how to abuse it

## Detection and Forensics

- [Moneta](https://github.com/forrest-orr/moneta) — live memory scanner, detects hollowing and shellcode
- [pe-sieve](https://github.com/hasherezade/pe-sieve) — scan process memory for injected/modified PE artifacts
- [Detecting Cobalt Strike (xpnsec)](https://blog.xpnsec.com/how-to-argue-your-shellcode/) — watermarks, sleep mask, stack spoofing
- [Sigma rules (SigmaHQ)](https://github.com/SigmaHQ/sigma) — SIEM-agnostic detection rules

## Tools (AD / Windows)

- [Impacket](https://github.com/fortra/impacket) — SMB/LDAP/Kerberos Python library, essential
- [BloodHound](https://github.com/BloodHoundAD/BloodHound) — AD attack path graph
- [CrackMapExec](https://github.com/byt3bl33d3r/CrackMapExec) — network protocol automation
- [Responder](https://github.com/lgandx/Responder) — LLMNR/NBT-NS/MDNS poisoner
- [Exegol](https://github.com/ThePorgs/Exegol) — ShutdownRepo — Docker pentest environment, pre-configured

## References

- [Windows Internals 7th ed.](https://docs.microsoft.com/en-us/sysinternals/resources/windows-internals) — Russinovich, Ionescu
- [Windows syscall table (j00ru)](https://j00ru.vexillium.org/syscalls/nt/64/) — per-build SSN reference
- [MITRE ATT&CK — Windows](https://attack.mitre.org/matrices/enterprise/windows/) — technique IDs, detections, mitigations

---

## Contact

[![ProtonMail](https://img.shields.io/badge/ProtonMail-8B89CC?style=flat-square&logo=protonmail&logoColor=white)](mailto:contact@franckferman.fr)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-0077B5?style=flat-square&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/franckferman)
[![X](https://img.shields.io/badge/X-000000?style=flat-square&logo=x&logoColor=white)](https://www.twitter.com/franckferman)
