# 🔵 Blue Team Guide — How to patch a BreachToPatch machine

> Audience: players who want to defend a machine by proposing a security patch.
> Prerequisites: Docker installed locally, basic understanding of the vulnerable technology, and a GitHub account.

---

## 1. How a patch works on BreachToPatch

A patch can only be submitted **after** a machine has been pwned at least once.
Until First Blood, the source code of the machine stays private — you have
nothing to study. Once a Red Teamer cracks it:

```
1. Sources are revealed in machines-archive (flag stripped out)
2. The Red Teamer's exploit is locked at {machine}/exploit/exploit.py in machines-public
3. You fork machines-archive and study the vulnerable code
4. You write a patch — typically a fixed docker-compose.yml + modified service files
5. You test locally that the exploit no longer works against your patch
6. You open a PR on machines-public with your patch
7. The CI replays the locked exploit against your patched version:
      Exit 1 (exploit blocked) + services still UP → patch ACCEPTED
      Exit 0 (exploit still works)                 → patch REJECTED
      Exit 2 or services down                       → SLA violation, REJECTED
8. A maintainer merges → the machine is reset to Red Team Active for the next cycle
```

A patch that holds against three consecutive re-pwn attempts earns the
🏰 **Fort Knox** badge.

---

## 2. What you need installed

| Tool | Why | Install |
|------|-----|---------|
| **Docker Engine + Compose plugin** | Run the patched services locally for testing | https://docs.docker.com/engine/install/ |
| **Git** | Clone your fork, push your patch | https://git-scm.com/downloads |
| **Python 3** | Run the locked exploit against your patched version | Pre-installed on macOS/Linux; https://www.python.org/downloads/ for Windows |

Optional (for full VM-level testing identical to a real player):
- Vagrant + VirtualBox — see [GUIDE_REDTEAM.md](GUIDE_REDTEAM.md#2-setting-up-your-environment)

---

## 3. Finding the source code

Every machine that has been pwned at least once has its sanitized source code
in `machines-archive`. Folder naming convention:

```
{machine-slug}-v{version}/
```

Examples:

| Folder | Meaning |
|--------|---------|
| `vuln-apache-path-traversal-v1.0` | Bee-Path, original version (revealed after first pwn) |
| `vuln-apache-path-traversal-v1.1` | Bee-Path after first patch (revealed after second pwn) |

Browse the archive: https://github.com/BreachToPatch/machines-archive

> The flag value is stripped from the archived sources — you don't need it to
> patch the vulnerability. The locked exploit (in `machines-public`) is what
> the CI uses to validate your patch.

---

## 4. Reading the locked exploit

Before writing your patch, read the locked exploit at:

```
https://github.com/BreachToPatch/machines-public/blob/main/{machine-slug}/exploit/exploit.py
```

This script is your enemy. Your patch is valid if and only if this exact
script returns exit code `1` (failure) against your patched version, while all
services still respond.

---

## 5. Studying the vulnerability

Fork `machines-archive` to your own GitHub account, then clone:

```bash
git clone https://github.com/{your-username}/machines-archive.git
cd machines-archive/{machine-slug}-v{version}
```

Read in this order:

1. `README.md` — context and history of the vulnerability
2. `app/` — the actual vulnerable application code and configuration
3. `app/docker-compose.yml` — how services are wired together
4. The locked exploit (in `machines-public`) — to know exactly what attack you must block

Identify the **root cause**. Patches that treat the symptom (e.g. blocking one
specific URL pattern) usually fail because the locked exploit can be slightly
adjusted by future Red Teamers — but more importantly, the next iteration of
the machine will likely keep the same root cause exposed if you only fix the
symptom on this version.

---

## 6. Understanding how patch validation works
 
Before writing your patch, understand what happens when you submit:
 
```
Your patched files (in writeups/patch/{username}/)
        │
        ▼
CI clones your branch (private runner — you can't see this)
        │
        ▼
CI injects a validation flag it controls via --build-arg FLAG_VALUE
        │
        ▼
CI builds your Docker image with that flag written to /root/flag.txt
        │
        ▼
CI replays the locked exploit.py against your patched container
        │
    ┌───┴───┐
   exit 1  exit 0
    │        │
    ✅       ❌
 Exploit   Exploit retrieved the injected flag
 blocked   → your patch doesn't fix the root cause
```
 
**Why you don't need to know the flag value:** The CI injects its own flag — you never see it. Your patch just needs to make CVE-2021-41773 unexploitable. If the exploit can't traverse to `/root/flag.txt`, it returns exit `1` regardless of what's in that file.
 
---
 
## 7. Preparing your patch
 
### Step 1 — Get the source code
 
The sanitized source code is in `machines-archive`:
 
```
https://github.com/BreachToPatch/machines-archive/tree/main/{machine-slug}-v{version}
```
 
Clone or download this folder. The flag value has been replaced with `FLAG{PLACEHOLDER}` — that's intentional.
 
### Step 2 — Understand the vulnerability
 
Read the locked exploit at:
```
https://github.com/BreachToPatch/machines-public/blob/main/{machine-slug}/exploit/exploit.py
```
 
For `vuln-apache-path-traversal`: the exploit uses CVE-2021-41773 to traverse outside the document root via URL-encoded dots (`%2e`), then executes arbitrary commands via `mod_cgi`. Two valid fixes:
1. **Upgrade Apache** to 2.4.51 or later (fixes the path normalization bug)
2. **Disable `mod_cgi`** (removes the RCE escalation vector)
3. **Fix the `<Directory />` directive** (remove `Require all granted` on the root)
Address the root cause — not just the specific payload.
 
### Step 3 — Write your fix locally
 
Edit the relevant files from `machines-archive`:
- `app/apache/httpd.conf` — Apache configuration
- `app/apache/Dockerfile` — base image version or module config
- Any other file contributing to the vulnerability
Test locally:
```bash
# Build with a test flag
FLAG_VALUE="FLAG{test}" docker compose build --build-arg FLAG_VALUE="FLAG{test}"
docker compose up -d
 
# Check the service responds
curl http://localhost:80/     # should return HTTP 200 or 403
 
# Run the locked exploit against your patched version
TARGET_IP=localhost TARGET_PORT=80 python3 ../../../exploit/exploit.py
echo "Exit code: $?"
# Must be 1 (exploit blocked) — NOT 0 (exploit worked)
 
# Cleanup
docker compose down --volumes
```
 
---
 
## 8. Submission structure
 
Your submission folder must follow this exact structure:
 
```
{machine-slug}/writeups/patch/{your-username}/
├── writeup.md                ← required — explains root cause and fix
├── docker-compose.yml        ← required — CI test compose file
└── patch/                    ← required — your patched files, mirrors machine structure
    └── app/
        └── apache/
            ├── Dockerfile    ← your patched Dockerfile (MUST have ARG FLAG_VALUE)
            └── httpd.conf    ← your patched config (if you changed it)
```
 
Everything inside `patch/` is copied on top of the machine's current version folder when your patch is promoted. Only include files you actually modified.
 
---
 
## 8.1 Required docker-compose.yml format
 
Your `docker-compose.yml` MUST pass `FLAG_VALUE` to your Dockerfile as a build arg.
The CI sets this environment variable before running your compose file.
 
```yaml
services:
  apache:
    build:
      context: ./patch/app/apache/
      args:
        FLAG_VALUE: ${FLAG_VALUE}   # reads from environment — DO NOT hardcode
    ports:
      - "80:80"
    restart: unless-stopped
```
 
> **Do NOT** hardcode a flag value in your `docker-compose.yml`.
> The CI injects its own flag. Your job is to block access to it, not predict it.
 
---
 
## 8.2 Required Dockerfile format
 
Your `Dockerfile` inside `patch/` MUST:
1. Accept `FLAG_VALUE` as a build arg
2. Write it to `/root/flag.txt` with `chmod 400`
```dockerfile
FROM httpd:2.4.49   # or httpd:2.4.51 if upgrading
 
# --- YOUR SECURITY FIX HERE ---
# Example: copy a fixed httpd.conf
COPY httpd.conf /usr/local/apache2/conf/httpd.conf
 
# --- REQUIRED: flag injection pattern (do not remove) ---
# The CI uses this to inject its validation flag.
ARG FLAG_VALUE=FLAG{PLACEHOLDER}
RUN mkdir -p /root \
    && echo "$FLAG_VALUE" > /root/flag.txt \
    && chmod 400 /root/flag.txt
 
EXPOSE 80
```
 
> **Important:** You may modify anything in the Dockerfile — base image, configuration,
> installed modules — as long as you keep the `ARG FLAG_VALUE` block.
> Removing it will cause the CI to fail with a configuration error, not an acceptance.
 
---
 
## 9. Submitting your patch
 
### Step 1 — Fork and clone machines-public
 
```bash
git clone https://github.com/{your-username}/machines-public.git
cd machines-public
```
 
### Step 2 — Create your branch
 
```bash
git checkout -b patch/{your-github-username}/{machine-slug}
```
 
### Step 3 — Add your files
 
Create the folder structure as described above inside `{machine-slug}/writeups/patch/{your-username}/`.
 
### Step 4 — Commit and push
 
```bash
git add {machine-slug}/writeups/patch/{your-username}/
git commit -m "patch: {machine-slug} — fix CVE description"
git push origin patch/{your-username}/{machine-slug}
```
 
### Step 5 — Open the Pull Request
 
Open a PR from your branch into `BreachToPatch/machines-public:main`.
 
The CI will:
1. Check branch name, files, scope, and secrets (< 30 seconds)
2. Post "validation in progress" 🕐
3. Run the full test privately (~3 minutes): builds your image with the injected flag, replays the exploit
4. Post the final result comment ✅ or ❌
---
 
## 10. What the CI checks
 
```
Structural checks (public CI):
  ✓ Branch name: patch/{username}/{machine}
  ✓ Folder name matches PR author
  ✓ PR only touches your own folder
  ✓ writeup.md present
  ✓ docker-compose.yml present
  ✓ patch/ subfolder present
  ✓ locked exploit.py exists (machine was pwned)
  ✓ No secrets accidentally committed (TruffleHog)
 
Security test (private CI — invisible to players):
  ✓ docker compose build --build-arg FLAG_VALUE={real_flag} succeeds
  ✓ Service responds on port 80 (SLA)
  ✓ Locked exploit.py returns exit 1 (exploit blocked)
       If exit 0: hash of obtained flag verified against real flag
       If hash matches → patch insufficient ❌
       If hash mismatch → docker-compose misconfiguration ❌
```

## 11. After your patch is merged

- Your username appears in `leaderboard.json` (auto-updated within seconds).
- The machine is **automatically tagged** with the next version
  (e.g. `vuln-apache-path-traversal/v1.1`) by the CI.
- A new image is built on the private `btop-sources` side, and the machine
  resets to **Red Team Active** for the next cycle.
- Your patch writeup remains in
  `machines-public/{machine-slug}/writeups/patch/{your-username}/` forever — a
  public proof of your defensive work, linked to your GitHub profile.

---

## 12. Anti-patterns — patches that always get rejected

- **Hardcoding the locked exploit's payload as forbidden input.** The CI runs
  the exact same payload, so you would block it — but a maintainer reviewing
  your writeup will reject the PR for being a dishonest patch (treating the
  symptom, not the root cause).
- **Disabling the entire service.** The SLA check fails — the service must
  still respond.
- **Adding a backdoor for "future debugging".** TruffleHog and maintainer
  review will catch it. This is grounds for permanent ban.
- **Submitting on a fork without opening a PR.** Submissions live or die in
  the PR — there is no other path.

---

## 13. Templates

- [`templates/writeup_patch.md`](templates/writeup_patch.md) — patch writeup structure

---

Defend wisely. From shadow to light.