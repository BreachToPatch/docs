# docs 📚

> **Public repository — `BreachToPatch` organization**
> The knowledge base. Everything a player, contributor, or maintainer needs to know about BreachToPatch — in one place.

---

## What this repo is

`docs` is the **documentation hub** of the BreachToPatch platform. It contains guides for every type of participant (players, machine creators, maintainers), templates for standardized contributions, and the platform's foundational manifesto.

Good documentation is what makes an open-source project self-sustaining. The goal: any person with zero prior BreachToPatch experience should be able to pick any guide and know exactly what to do, step by step, without asking for help.

---

## What lives here

```
docs/
├── README.md                    ← You are here — index of all docs
├── MANIFESTO.md                 ← Core principles and philosophy of BreachToPatch
│
├── GUIDE_REDTEAM.md             ← How to download, run, and pwn a machine
├── GUIDE_BLUETEAM.md            ← How to analyze and patch a machine
├── GUIDE_CREATOR.md             ← How to build and submit a new machine
├── GUIDE_MAINTAINER.md          ← Internal ops guide (CI, releases, secrets)
│
└── templates/
    ├── writeup_attack.md        ← Template for Red Team writeups
    ├── writeup_remediation.md   ← Template for Blue Team patch writeups
    └── exploit_script.py        ← Template for exploit scripts (with correct exit codes)
```

---

## Guides at a glance

### 🔴 [Red Team Guide](GUIDE_REDTEAM.md)
*For players who want to pwn machines.*

Covers:
- Installing Vagrant and VirtualBox
- Downloading and booting a machine
- The rules of engagement
- How to write a valid exploit script (the required format with exit codes)
- How to prove your flag (sha256sum steps)
- How to submit a writeup via Pull Request

### 🔵 [Blue Team Guide](GUIDE_BLUETEAM.md)
*For players who want to patch machines.*

Covers:
- Understanding the source code structure
- How to fork `machines-archive`
- What makes a valid patch (SLA + exploit must fail)
- How to test your patch locally
- How to submit a patch PR

### ⚙️ [Creator Guide](GUIDE_CREATOR.md)
*For community members who want to build new machines.*

Covers:
- What makes a good CTF machine
- The full file structure (Vagrantfile, Ansible, Docker Compose)
- How to hardcode a flag and generate its hash
- How to write the reference exploit
- The checklist before submitting

### 🔧 [Maintainer Guide](GUIDE_MAINTAINER.md)
*Internal — for platform maintainers only.*

Covers:
- How to review and merge machine submissions
- How to cut a new machine release (Vagrant box + GitHub Release)
- How to archive sources after first pwn
- Managing GitHub Actions secrets
- Handling abuse reports

---

## Templates

### [writeup_attack.md](templates/writeup_attack.md)
The standard template all Red Teamers must use for their writeup PR. Includes sections for:
- Recon / enumeration steps
- Vulnerability identification
- Exploitation walkthrough
- Proof (flag hash verification)

### [writeup_remediation.md](templates/writeup_remediation.md)
The standard template for Blue Team patch writeups. Includes:
- Root cause analysis
- What was changed and why
- How to verify the patch works

### [exploit_script.py](templates/exploit_script.py)
A commented Python template with the correct exit code behavior:
- `exit(0)` — exploit succeeded, flag obtained
- `exit(1)` — exploit failed (vulnerability is patched)
- `exit(2)` — service unreachable (network error / SLA violation)

This interface is what the CI pipeline expects. Exploits that don't follow it will be automatically rejected.

---

## The Manifesto

[MANIFESTO.md](MANIFESTO.md) — *"From shadow to light, from breach to patch."*

The foundational document that explains what BreachToPatch is, why it exists, and the principles it will never compromise on (free core, open source, no pay-to-win, GitHub-native).

---

## How this repo interacts with other repos

```
docs (this repo)
   └── Referenced from every other repo's README
   └── Templates are used by players when opening PRs in machines-public
   └── Linked from the web-site navigation

machines-public (BreachToPatch, public)
   └── PR templates will reference the guides here

web-site (BreachToPatch, public)
   └── "Docs" nav link points here
```

---

## Contributing to the docs

See something unclear or missing? Open a PR. Documentation improvements are some of the most valuable contributions to any open-source project — they don't require hacking skills, just clear writing.
