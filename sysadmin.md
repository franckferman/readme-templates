# attack-box-setup

Post-install script for a hardened Debian 12 pentest workstation — installs and configures the core offensive toolset, shell environment, and network setup used on Red Team engagements.

> [!WARNING]
> Run this only on a dedicated machine or VM. It modifies system-wide configurations, installs tools from third-party sources, and adjusts kernel parameters. Do not run on production systems.

---

## Table of Contents

1. [What it installs](#1-what-it-installs)
2. [Requirements](#2-requirements)
3. [Installation](#3-installation)
4. [Profile system](#4-profile-system)
5. [Configuration](#5-configuration)
6. [Post-install steps](#6-post-install-steps)

---

## 1. What it installs

**Core tooling:**

| Tool | Source | Purpose |
|---|---|---|
| [Exegol](https://github.com/ThePorgs/Exegol) | pip | Docker-based pentest environment |
| [Impacket](https://github.com/fortra/impacket) | pip | SMB/LDAP/Kerberos protocol library |
| [CrackMapExec](https://github.com/byt3bl33d3r/CrackMapExec) | pip | Network protocol attack automation |
| [Responder](https://github.com/lgandx/Responder) | git | LLMNR/NBT-NS/MDNS poisoner |
| [BloodHound.py](https://github.com/fox-it/BloodHound.py) | pip | AD graph ingestor |
| [Burp Suite Community](https://portswigger.net/burp) | deb | Web proxy |
| `nmap`, `masscan`, `rustscan` | apt / cargo | Port scanning |
| `john`, `hashcat` | apt | Password cracking |

**Shell environment:**

- zsh + Oh My Zsh + zsh-autosuggestions + zsh-syntax-highlighting
- tmux with hardened `.tmux.conf` (persistent sessions, mouse support, vim keybindings)
- Custom `.zshrc` with pentest aliases (`cmx`, `bh`, `resp`, `msfconsole`)

**Network:**

- OpenVPN client config templates
- Proxychains4 pre-configured
- `iptables` rules: drop inbound by default, allow established, allow lo

---

## 2. Requirements

- Debian 12 (Bookworm) — x86_64
- Root or sudo access
- Internet access during setup (~2-4 GB download)
- At least 20 GB free disk space

**Does not support:** Ubuntu, Kali, Arch, Windows WSL2. Use Exegol containers for those.

---

## 3. Installation

```bash
git clone https://github.com/franckferman/attack-box-setup.git
cd attack-box-setup
sudo bash setup.sh
```

The script is non-interactive by default and uses the `full` profile. To select a specific profile:

```bash
sudo bash setup.sh --profile minimal
sudo bash setup.sh --profile redteam
sudo bash setup.sh --profile full
```

---

## 4. Profile system

Profiles are cumulative — each inherits from the previous:

```
minimal
  └── core (nmap, impacket, responder, zsh)
       └── redteam (+ bloodhound, crackmapexec, exegol, burp)
            └── full (+ hashcat, john, metasploit, custom kernel params)
```

| Profile | Install time | Disk | Use case |
|---|---|---|---|
| `minimal` | ~5 min | ~1 GB | Quick VM, CTF, single engagement |
| `redteam` | ~15 min | ~5 GB | Standard Red Team workstation |
| `full` | ~30 min | ~12 GB | Primary dedicated machine |

---

## 5. Configuration

Copy the example config and adjust before running:

```bash
cp config.example.env config.env
```

| Variable | Default | Description |
|---|---|---|
| `INSTALL_PROFILE` | `full` | Profile: `minimal`, `redteam`, `full` |
| `VPN_CONFIG_DIR` | `/etc/openvpn/client` | Where to copy `.ovpn` templates |
| `EXEGOL_IMAGE` | `full` | Exegol Docker image: `light`, `full`, `nightly` |
| `BLOODHOUND_VERSION` | `4.3.1` | BloodHound desktop app version |
| `ENABLE_KERNEL_HARDENING` | `true` | Apply `/etc/sysctl.d/99-pentest.conf` |
| `SETUP_IPTABLES` | `true` | Install default deny-inbound iptables rules |
| `ZSH_DEFAULT` | `true` | Set zsh as default shell for current user |

---

## 6. Post-install steps

These require manual action and cannot be automated:

1. **Import BloodHound databases** — start Neo4j (`neo4j start`) and import `.json` from previous engagements
2. **Configure VPN profiles** — copy `.ovpn` files to `/etc/openvpn/client/` and test with `sudo openvpn --config client.ovpn`
3. **Set up Exegol workspace** — run `exegol start full mybox` to pull and launch your first container
4. **Configure proxychains** — edit `/etc/proxychains4.conf` with your SOCKS5 proxy if needed
5. **Install Burp cert** — export CA from Burp and add to system + browser trust stores

---

## License

Licensed under the **GNU General Public License v3.0**.
See [LICENSE](LICENSE) for full terms.

---

## Contact

[![ProtonMail](https://img.shields.io/badge/ProtonMail-8B89CC?style=flat-square&logo=protonmail&logoColor=white)](mailto:contact@franckferman.fr)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-0077B5?style=flat-square&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/franckferman)
[![X](https://img.shields.io/badge/X-000000?style=flat-square&logo=x&logoColor=white)](https://www.twitter.com/franckferman)
