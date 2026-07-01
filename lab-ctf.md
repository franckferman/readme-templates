<div id="top" align="center">

[![License](https://img.shields.io/badge/license-AGPL--3.0-blue?style=flat-square)](LICENSE)
[![Platform](https://img.shields.io/badge/platform-Docker%20%7C%20AWS-lightgrey?style=flat-square)]()
[![Challenges](https://img.shields.io/badge/challenges-8-brightgreen?style=flat-square)]()

<h1 align="center">KerbLab</h1>
<p align="center">
  <em>Hands-on Active Directory Kerberos attack training platform with live proof-of-impact scoring.</em><br>
  8 scenarios. 14 containers. Flags are only issued when your attack produces a verifiable artefact.
</p>

</div>

---

## Table of Contents

<details open>
  <summary><strong>Click to collapse/expand</strong></summary>
  <ol>
    <li><a href="#what-is-kerblab">What is KerbLab</a></li>
    <li><a href="#attack-taxonomy">Attack Taxonomy</a></li>
    <li><a href="#judge-architecture">Judge Architecture</a></li>
    <li><a href="#network-topology">Network Topology</a></li>
    <li><a href="#project-structure">Project Structure</a></li>
    <li><a href="#quick-start--docker">Quick Start — Docker</a></li>
    <li><a href="#scenarios">Scenarios</a></li>
    <li><a href="#flag-system">Flag System</a></li>
    <li><a href="#judge-api">Judge API</a></li>
    <li><a href="#quick-start--terraform-aws">Quick Start — Terraform AWS</a></li>
    <li><a href="#monitoring">Monitoring</a></li>
    <li><a href="#troubleshooting">Troubleshooting</a></li>
    <li><a href="#bibliography">Bibliography</a></li>
    <li><a href="#license">License</a></li>
    <li><a href="#contact">Contact</a></li>
  </ol>
</details>

---

## What is KerbLab

KerbLab is a purpose-built Active Directory lab for learning Kerberos attack techniques through hands-on execution. It is the companion platform for [Phantasm](https://github.com/franckferman/phantasm), the AD ACL enumeration and abuse toolkit.

Classic CTF challenges ask you to find a flag in a file. KerbLab inverts this: you **prove your attack produced a cryptographic artefact**. The judge polls Kerberos infrastructure every 10 seconds. When it confirms your attack worked — by observing a forged TGT, an S4U2Proxy service ticket for a privileged account, a DCSync replication event, or a cracked hash in the dictionary — it generates a time-windowed cryptographic flag. Your artefact must still be fresh when you submit.

Two deployment modes:

- **Docker** — full local lab on a single machine with Samba 4 DC + member servers + workstations
- **Terraform** — cloud deployment on AWS with a domain-joined Windows AD environment

The platform documentation is at [franckferman.github.io/KerbLab](https://franckferman.github.io/KerbLab/).

> [!IMPORTANT]
> All containers are isolated on a private Docker network with IP masquerade disabled. Traffic cannot reach the internet from inside the lab. Attacks stay contained.

---

## Attack Taxonomy

KerbLab covers eight Kerberos and AD abuse techniques, grouped by authentication protocol step.

### Pre-Authentication Weaknesses

**AS-REP Roasting (scenario 01)**

Accounts with `DONT_REQUIRE_PREAUTH` set skip the AS-REQ encrypted timestamp step. The KDC responds to any AS-REQ for these accounts — no credentials required — with an AS-REP whose `enc-part` is encrypted with the account's password-derived key. That `enc-part` is crackable offline. In most production domains, this flag is set unintentionally on service accounts where an administrator configured a legacy application that predates pre-authentication.

**Kerberoasting (scenario 02)**

Any authenticated domain user can request a TGS for any Service Principal Name. The TGS `enc-part` is encrypted with the service account's key. Accounts with SPNs and weak passwords are crackable with `hashcat -m 13100`. The attack requires only a valid low-privilege ticket (LDAP enumeration → SPN query → TGS request) — no elevated rights.

### Delegation Abuse

**Unconstrained Delegation (scenario 03)**

Servers configured with unconstrained delegation (`TrustedForDelegation`) receive a copy of the authenticating user's full TGT in memory. An operator with admin rights on an unconstrained delegation host can extract these TGTs using `mimikatz::sekurlsa::tickets` or `Rubeus dump`. If a domain admin authenticates to that host (e.g., via printer coercion — scenario 03b), their TGT is captured and replayable.

**Constrained Delegation (scenario 04)**

S4U2Proxy allows a service to request a TGS on behalf of a user to a specific set of target SPNs (`msDS-AllowedToDelegateTo`). If the service account's TGT is compromised, an attacker can impersonate any user to those target services — including `administrator` — without requiring the user to authenticate first. The attack uses S4U2Self to obtain a forwardable ticket for the impersonated account, then S4U2Proxy to delegate it.

**Resource-Based Constrained Delegation / RBCD (scenario 05)**

An account with `GenericWrite` or `WriteProperty` over a computer object can set `msDS-AllowedToActOnBehalfOfOtherIdentity` to point to a controlled computer account. This allows the controlled account to impersonate any user (including domain admins) to the target computer's services via S4U2Proxy. No domain admin rights required to configure — only LDAP write access to the target computer object.

### Forged Tickets

**Silver Ticket (scenario 06)**

A service ticket (TGS) is encrypted with the service account's NT hash, not the KDC's `krbtgt` key. With the service account hash, an attacker can forge a TGS entirely offline — no KDC interaction, no event log. The forged ticket is valid for the lifetime set in the golden ticket (default: 10 years). The judge validates this by attempting a Kerberos authentication to the target service using the submitted ticket.

**Golden Ticket (scenario 07)**

The `krbtgt` account NT hash is the root trust anchor for all Kerberos in the domain. With it, any TGT can be forged for any user, with any PAC, valid for any duration. Obtaining `krbtgt` requires DCSync or NTDS.dit extraction. The judge validates this by checking for a DCSync replication event in the DC security event log (Event ID 4662 with `Replicating Directory Changes All` rights).

### AD CS Abuse

**ESC1 — AD CS Template Abuse (scenario 08)**

An AD CS template with `CT_FLAG_ENROLLEE_SUPPLIES_SUBJECT` enabled and enrollment rights granted to domain users allows any user to request a certificate with an arbitrary subject — including `administrator@corp.local`. That certificate authenticates to the KDC via PKINIT for a TGT as the impersonated user.

---

## Judge Architecture

The judge is an async Python service (`aiohttp`) that runs the scoring engine: attack verification, flag generation, submission validation, hint delivery, and score tracking.

### Polling Loop

```
every 10 seconds:
  for each scenario:
    verify(scenario_id) -> VerifyResult(success: bool, artefact: str, detail: str)
    if success:
      generate_flag(scenario_id, current_time_window)
      update state
```

Each scenario has a dedicated verification function matched to the attack artefact:

| Scenario | Verification method |
|---|---|
| AS-REP Roasting | Monitor `${LOOT_DIR}/asrep_hashes/` — new `.hash` file with `$krb5asrep$23$` prefix |
| Kerberoasting | Monitor `${LOOT_DIR}/tgs_hashes/` — new `.hash` file with `$krb5tgs$23$` prefix AND cracked in `cracked.txt` |
| Unconstrained Delegation | Monitor `${LOOT_DIR}/tickets/` — `.ccache` file containing a DA TGT (LDAP check: group membership) |
| Constrained Delegation | Kerberos auth test: attempt `kinit` with submitted S4U2Proxy ticket to target service |
| RBCD | Kerberos auth test: impersonated DA ticket valid for target host CIFS service |
| Silver Ticket | Kerberos auth test: forged TGS accepted by target service (no LDAP interaction logged) |
| Golden Ticket | DC Event ID 4662: `Replicating Directory Changes All` in security log within last 30s |
| AD CS ESC1 | LDAP auth with submitted certificate: `whoami` returns `corp.local\administrator` |

### HMAC-SHA256 Time-Windowed Flags

```python
FLAG_TTL = 600  # 10 minutes

def generate_flag(scenario_id: str, window: int = None) -> str:
    if window is None:
        window = int(time.time()) // FLAG_TTL
    payload = f"{scenario_id}:{window}".encode()
    sig = hmac.new(SECRET.encode(), payload, sha256).hexdigest()[:24]
    return f"KERBLAB{{{scenario_id.upper()}_{sig}}}"

def verify_flag(scenario_id: str, flag: str) -> bool:
    cur = int(time.time()) // FLAG_TTL
    for w in [cur, cur - 1]:  # accept current and previous window
        if hmac.compare_digest(flag, generate_flag(scenario_id, w)):
            return True
    return False
```

A valid submission also requires that the judge confirmed the attack artefact within the last 10 seconds — preventing flag reuse after the artefact expires from the loot directory.

---

## Network Topology

```
10.0.1.0/24  Attacker Network
  10.0.1.10  attacker      Kali — impacket, Rubeus, certipy, bloodhound-python, hashcat

10.0.2.0/24  Domain — corp.local
  10.0.2.10  dc01          Samba 4 DC — LDAP:389, Kerberos:88, DNS:53
  10.0.2.20  srv01         Member server — SMB:445, CIFS, HTTP:80 (IIS stub)
  10.0.2.21  srv02         Member server — MSSQL:1433 (Samba-backed auth stub)
  10.0.2.30  ws01          Workstation — RDP:3389 (FreeRDP server stub)
  10.0.2.31  ws02          Workstation — unconstrained delegation enabled

10.0.3.0/24  PKI Network
  10.0.3.10  ca01          AD CS stub — certsrv HTTP:80 — ESC1 template enabled

10.0.99.0/24 Management Network
  10.0.99.10 attacker      (also on this subnet)
  10.0.99.20 judge         :8888 — KerbLab Judge API
  10.0.99.21 loot-watcher  monitors ${LOOT_DIR} for new artefacts
  10.0.99.22 grafana        :3000 — admin / kerblab
```

The domain and PKI networks have `com.docker.network.bridge.enable_ip_masquerade: "false"` — no external traffic from inside the lab.

---

## Project Structure

```
KerbLab/
├── Makefile                       up / down / clean / shell / status / reset
├── README.md
│
├── judge/
│   ├── judge.py                   Async polling + HMAC flag generation + REST API
│   ├── verify/
│   │   ├── asrep.py               Hash file watcher
│   │   ├── kerberoast.py          Hash + cracked file watcher
│   │   ├── delegation.py          Kerberos auth test (kinit)
│   │   ├── dcsync.py              DC event log reader via impacket
│   │   └── adcs.py                LDAP auth with PKINIT cert
│   ├── hints.py                   3-level progressive hints
│   ├── writeups.py                Unlocked on correct submission
│   └── Dockerfile
│
├── ui/
│   └── cli.py                     Player terminal interface (Rich TUI)
│
├── docker/
│   ├── docker-compose.yml         Full lab: 14 containers, 4 networks
│   ├── attacker/                  Kali-based — all tools pre-installed
│   ├── dc01/                      Samba 4 — pre-provisioned domain corp.local
│   │   ├── provision.sh           Creates users, groups, SPNs, misconfigs
│   │   └── smb.conf
│   ├── srv01/                     Member server — CIFS, constrained delegation config
│   ├── srv02/                     MSSQL stub — SPN `MSSQLSvc/srv02.corp.local`
│   ├── ws01/                      Workstation — low-priv user `jdupont` RDP session
│   ├── ws02/                      Unconstrained delegation workstation
│   ├── ca01/                      AD CS stub — certsrv + ESC1 template
│   ├── loot-watcher/              Inotify watcher → judge notification
│   └── monitoring/                Prometheus + node-exporter
│
└── terraform/aws/
    ├── main.tf                    VPC + EC2 (Windows AD + Kali)
    ├── variables.tf
    └── user_data/
        ├── dc.ps1                 AD provisioning PowerShell
        └── attacker.sh            Kali toolset install
```

---

## Quick Start — Docker

### Requirements

- Docker 20.10+ and Docker Compose v2
- Linux host recommended — Kerberos raw socket operations require `NET_RAW`
- 8 GB RAM minimum, 15 GB free disk
- GNU `make` (optional)

### Deploy

```bash
git clone https://github.com/franckferman/kerblab
cd kerblab
make up           # Build images, provision AD, start all services (~3 min)
make status       # Verify all containers are healthy
make shell        # Open attacker container shell
```

Without `make`:

```bash
docker compose -f docker/docker-compose.yml up -d --build
docker compose -f docker/docker-compose.yml exec attacker bash
docker compose -f docker/docker-compose.yml ps
```

### Web Interfaces

| Interface | URL | Credentials |
|---|---|---|
| Judge API | http://localhost:8888/status | — |
| Grafana | http://localhost:3000 | admin / kerblab |
| Prometheus | http://localhost:9090 | — |

### Inside the Attacker Container

```bash
# List all scenarios and current state
python3 /opt/scenarios/run.py list

# Show instructions for a scenario
python3 /opt/scenarios/run.py run 01

# Submit a flag
python3 /opt/ui/cli.py submit 01

# Hint (costs points)
python3 /opt/ui/cli.py hint 01 --level 2

# Full writeup (unlocked after solve)
python3 /opt/ui/cli.py writeup 01

# Scoreboard
python3 /opt/ui/cli.py scoreboard
```

### Loot Directory

Place your attack artefacts in `/opt/loot/` inside the attacker container. The loot-watcher service monitors this directory and notifies the judge:

```bash
# AS-REP hash → /opt/loot/asrep_hashes/
impacket-GetNPUsers corp.local/ -dc-ip 10.0.2.10 -no-pass -usersfile /opt/wordlists/users.txt \
  -outputfile /opt/loot/asrep_hashes/asrep.hash

# Kerberoast hashes → /opt/loot/tgs_hashes/
impacket-GetUserSPNs corp.local/jdupont:'Password123' -dc-ip 10.0.2.10 \
  -outputfile /opt/loot/tgs_hashes/spn.hash

# Crack TGS hash → /opt/loot/cracked.txt (judge checks both files)
hashcat -m 13100 /opt/loot/tgs_hashes/spn.hash /opt/wordlists/rockyou.txt \
  --outfile /opt/loot/cracked.txt
```

---

## Scenarios

| # | Name | Target | Technique | Difficulty | Points |
|---|---|---|---|---|---|
| 01 | AS-REP Roasting | dc01 (Kerberos:88) | Pre-auth disabled accounts | Easy | 100 |
| 02 | Kerberoasting | dc01 (Kerberos:88) | SPN account hash offline crack | Easy | 100 |
| 03 | Unconstrained Delegation | ws02 + coerce DA | TGT capture via printer coercion | Medium | 150 |
| 04 | Constrained Delegation | srv01 → srv02 | S4U2Self + S4U2Proxy | Medium | 150 |
| 05 | RBCD | srv01 (GenericWrite) | msDS-AllowedToActOnBehalfOfOtherIdentity | Hard | 200 |
| 06 | Silver Ticket | srv01 (CIFS) | Forged TGS with service account hash | Medium | 150 |
| 07 | Golden Ticket / DCSync | dc01 | krbtgt hash extraction → TGT forge | Hard | 200 |
| 08 | AD CS ESC1 | ca01 | Certificate template — enrollee supplies SAN | Hard | 200 |
| | | | **Total** | | **1,250** |

### Domain Configuration

The lab domain `corp.local` is pre-provisioned with deliberate misconfigurations. Key accounts:

```
jdupont       / Password123    — standard user, member of Domain Users
svc_sql       / Sql@2024!      — service account, SPN: MSSQLSvc/srv02.corp.local
svc_backup    / Backup#1       — DONT_REQUIRE_PREAUTH = true (AS-REP Roasting)
adminweb      / Summer2024     — DONT_REQUIRE_PREAUTH = true
srv01$                         — TrustedToAuthForDelegation (constrained delegation)
                                 msDS-AllowedToDelegateTo: cifs/srv02.corp.local
ws02$                          — TrustedForDelegation (unconstrained delegation)
Administrator / Corp@dm1n!     — Domain Admin (target for delegation/Golden Ticket)
```

Full user configuration in [docker/dc01/provision.sh](docker/dc01/provision.sh).

---

## Flag System

```
Format:  KERBLAB{SCENARIO_ID_hmac_truncated_hex}
Example: KERBLAB{01_a3f9c2d18e4b7f3d2c1b}
```

**Submission rules:**

1. The attack artefact must be current — the judge must have verified it within the last 10 seconds
2. Flags expire after the current and previous time window (~20 minutes grace period)
3. A correct submission unlocks the full technical writeup immediately
4. Hints cost points: Level 1 = -10, Level 2 = -25, Level 3 = -50
5. Each scenario can only be solved once per player

**Hint system:**

- **Level 1** — direction nudge, identifies the misconfigured attribute or account class
- **Level 2** — specific LDAP attribute or tool flag to investigate
- **Level 3** — near-explicit command showing the exact impacket/certipy invocation

---

## Judge API

| Endpoint | Method | Body / Params | Description |
|---|---|---|---|
| `/status` | GET | — | All scenarios, artefact state, active flags |
| `/submit` | POST | `{"player", "scenario", "flag"}` | Flag submission |
| `/hint/<id>` | GET | `?level=1\|2\|3` | Progressive hint for scenario |
| `/writeup/<id>` | GET | — | Full writeup (unlocked after solve) |
| `/scoreboard` | GET | — | Ranked player list |
| `/health` | GET | — | Service health |

```bash
# Check all targets and current artefact state
curl http://localhost:8888/status | python3 -m json.tool

# Submit a flag
curl -X POST http://localhost:8888/submit \
  -H "Content-Type: application/json" \
  -d '{"player":"yourname","scenario":"02","flag":"KERBLAB{02_...}"}'

# Get hint
curl "http://localhost:8888/hint/05?level=2"
```

---

## Quick Start — Terraform AWS

### Requirements

- Terraform 1.x+
- AWS CLI configured with VPC + EC2 + IAM permissions
- An existing EC2 key pair in the target region

```bash
cat > terraform/aws/terraform.tfvars << EOF
key_pair = "your-keypair-name"
your_ip  = "$(curl -s ifconfig.me)/32"
region   = "eu-west-1"
EOF

make tf-init
make tf-apply

# SSH to attacker node
ssh kali@<attacker_ip> -i ~/.ssh/your-keypair.pem

# Tear down
make tf-destroy
```

**Estimated AWS cost:** ~$0.50/hour with default instance types (t3.medium Kali + t3.small DC + t3.small×5 targets).

---

## Monitoring

```bash
# Kerberos ticket traffic
tcpdump -i eth0 -n 'port 88'

# LDAP queries from attacker
tcpdump -i eth0 -n 'port 389 or port 3268'

# Watch loot directory for new artefacts
watch -n2 'find /opt/loot -type f -newer /opt/loot/.stamp'

# Judge poll log
docker compose -f docker/docker-compose.yml logs -f judge

# Grafana dashboard (host)
open http://localhost:3000   # admin/kerblab
```

---

## Troubleshooting

### Kerberos clock skew error (`KRB_AP_ERR_SKEW`)

Kerberos tolerates a 5-minute clock skew between client and KDC. If your host clock drifts:

```bash
# Sync attacker container clock with DC
docker compose exec attacker bash -c "ntpdate 10.0.2.10"
```

### AS-REP hash file not detected by judge

The judge watches for files matching `*.hash` containing `$krb5asrep$23$`. Verify the format:

```bash
head -1 /opt/loot/asrep_hashes/asrep.hash
# expected: $krb5asrep$23$user@CORP.LOCAL:...
```

### DCSync not triggering Event ID 4662

The Samba DC must have auditing enabled. If the scenario was reset, re-run:

```bash
docker compose exec dc01 bash /opt/enable_auditing.sh
```

### Container exits immediately on `make up`

```bash
docker compose -f docker/docker-compose.yml logs dc01
```

Most commonly: Samba provision failed due to previous partial state. Reset fully:

```bash
make clean
make up
```

---

## Bibliography

[1] Duckwall, B., Campbell, C. (2014). *Abusing Microsoft Kerberos — Sorry You Guys Don't Get It*. Black Hat USA.

[2] Metcalf, S. (2015). *Sneaky Active Directory Persistence Tricks*. ADSecurity.org.

[3] Harmj0y. (2018). *A Case Study in Wagging the Dog: Computer Takeover*. harmj0y.net — RBCD original research.

[4] Wadhwa, A. (2021). *Certified Pre-Owned: Abusing Active Directory Certificate Services*. SpecterOps. — AD CS ESC1–8 taxonomy.

[5] Dirkjanm. (2020). *S4U2Pwnage — Revisiting Constrained Delegation*. dirkjanm.io.

[6] Gentilkiwi (Benjamin Delpy). (2014). *mimikatz: Golden Ticket*. GitHub / blog.gentilkiwi.com.

[7] RFC 4120. (2005). *The Kerberos Network Authentication Service (V5)*. IETF.

[8] RFC 4556. (2006). *Public Key Cryptography for Initial Authentication in Kerberos (PKINIT)*. IETF.

---

## License

GNU Affero General Public License v3.0. See [LICENSE](LICENSE) for full terms.

<p align="right">(<a href="#top">Back to top</a>)</p>

---

## Contact

[![ProtonMail](https://img.shields.io/badge/ProtonMail-8B89CC?style=flat-square&logo=protonmail&logoColor=white)](mailto:contact@franckferman.fr)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-0077B5?style=flat-square&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/franckferman)
[![X](https://img.shields.io/badge/X-000000?style=flat-square&logo=x&logoColor=white)](https://www.twitter.com/franckferman)

<p align="right">(<a href="#top">Back to top</a>)</p>
