# Apprendre le Reverse Engineering — Les Fondamentaux

**Guide libre et complet pour débuter et maîtriser l'analyse de binaires Windows et Linux.**

[![License](https://img.shields.io/badge/license-AGPL--3.0-blue?style=flat-square)](LICENSE)
[![Niveau](https://img.shields.io/badge/niveau-débutant%20→%20intermédiaire-green?style=flat-square)]()
[![Langue](https://img.shields.io/badge/langue-français-blue?style=flat-square)]()

> [!NOTE]
> Document classé **TLP:CLAIR** — diffusion publique autorisée sans restriction.

---

## Table of Contents

1. [À propos](#1-à-propos)
2. [Prérequis](#2-prérequis)
3. [Structure du cours](#3-structure-du-cours)
4. [Comment lire ce cours](#4-comment-lire-ce-cours)
5. [Exercices](#5-exercices)
6. [Contribuer](#6-contribuer)
7. [Licence](#7-licence)
8. [Contact](#8-contact)

---

## 1. À propos

Ce cours couvre les fondamentaux du reverse engineering : lecture de binaires PE et ELF, désassemblage statique avec IDA et Ghidra, analyse dynamique avec x64dbg et GDB, et introduction à l'analyse de malware.

Il est conçu pour être lu dans l'ordre, chaque chapitre s'appuyant sur le précédent. Les exercices sont inclus à la fin de chaque partie.

Si tu démarres de zéro en C, commence par [Apprendre le C](https://github.com/franckferman/apprendre_le_c) avant ce cours.

---

## 2. Prérequis

| Prérequis | Niveau attendu |
|---|---|
| Programmation C | Comprendre les pointeurs, la pile, les structures |
| Système d'exploitation | Savoir ce qu'est un processus, une adresse mémoire, un EXE/ELF |
| Ligne de commande | Linux basique — `file`, `strings`, `xxd`, `objdump` |

---

## 3. Structure du cours

```
cours/
├── 01-fondamentaux/
│   ├── 01-format-pe-elf.md          Format PE (Windows) et ELF (Linux) — anatomie d'un binaire
│   ├── 02-representations.md        Hex, assembleur x86-64, calling conventions
│   └── 03-outils-essentiels.md      file, strings, objdump, xxd, readelf
│
├── 02-analyse-statique/
│   ├── 04-ida-free.md               Interface, navigation, nommage, types
│   ├── 05-ghidra.md                 Décompilation, scripting, diff de binaires
│   ├── 06-structures-controle.md    if/else, boucles, switch en assembleur
│   ├── 07-structures-donnees.md     Tableaux, structs, vtables C++
│   └── 08-strings-obfusquees.md     Détection et décodage de strings XOR
│
├── 03-analyse-dynamique/
│   ├── 09-x64dbg-windbg.md          Breakpoints, step over/into, dump mémoire
│   ├── 10-gdb-pwndbg.md             Linux, analyse de la pile, registres
│   ├── 11-api-monitor.md            API Monitor et Process Monitor — tracer les appels Win32
│   └── 12-frida.md                  Hook dynamique sans recompiler
│
├── 04-cas-pratiques/
│   ├── 13-crackme-niveau1.md        Reverse d'un serial checker
│   ├── 14-crackme-niveau2.md        Anti-debug et obfuscation légère
│   ├── 15-unpacking-upx.md          OEP finding, dump, import reconstruction
│   └── 16-intro-malware.md          Comportement, persistence, réseau
│
├── annexes/
│   ├── A-instructions-x86-64.md     Référence des instructions fréquentes
│   ├── B-calling-conventions.md     Windows x64 et Linux System V
│   └── C-ressources.md              Lectures complémentaires et CTF recommandés
│
└── exercices/
    ├── binaires/                     Binaires compilés pour les exercices
    ├── src/                          Sources — à lire APRÈS avoir tenté l'exercice
    └── solutions/                    Solutions commentées
```

---

## 4. Comment lire ce cours

**Clone le repo et lis dans l'ordre :**

```bash
git clone https://github.com/franckferman/apprendre_le_reverse_engineering
cd apprendre_le_reverse_engineering
```

Chaque fichier `.md` est autonome et lisible directement dans n'importe quel éditeur ou sur GitHub. Commence par `cours/01-fondamentaux/` et avance linéairement.

**Outils à installer avant de commencer :**

```bash
# Linux
sudo apt install binutils xxd gdb file
pip install pwntools

# Windows (télécharger manuellement)
# IDA Free    → hex-rays.com/ida-free
# x64dbg      → x64dbg.com
# Ghidra      → ghidra-sre.org
```

---

## 5. Exercices

Les binaires des exercices sont dans `exercices/binaires/`. Les sources dans `exercices/src/` — ne pas lire avant d'avoir tenté.

```bash
# Exemple — exercice chapitre 6
file exercices/binaires/ch06_control_flow
strings exercices/binaires/ch06_control_flow | grep -i "flag\|password\|key"
```

Solutions dans `exercices/solutions/` avec explication pas à pas.

---

## 6. Contribuer

**Contributions acceptées :**
- Corrections factuelles — erreur technique, commande incorrecte, typo
- Exercices supplémentaires — binaire compilé + source + solution commentée
- Clarifications sans changer le fond

**Contributions refusées :**
- Outils propriétaires ou nécessitant une licence payante
- Chapitres entiers sans discussion préalable — ouvrir une issue d'abord

```bash
git clone https://github.com/franckferman/apprendre_le_reverse_engineering
git checkout -b fix/chapter-06-typo
# éditer
git commit -m "fix: correct objdump flag in chapter 6"
git push origin fix/chapter-06-typo
# ouvrir une Pull Request
```

---

## 7. Licence

GNU Affero General Public License v3.0. Voir [LICENSE](LICENSE) pour les termes complets.

---

## 8. Contact

[![ProtonMail](https://img.shields.io/badge/ProtonMail-8B89CC?style=flat-square&logo=protonmail&logoColor=white)](mailto:contact@franckferman.fr)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-0077B5?style=flat-square&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/franckferman)
[![X](https://img.shields.io/badge/X-000000?style=flat-square&logo=x&logoColor=white)](https://www.twitter.com/franckferman)
