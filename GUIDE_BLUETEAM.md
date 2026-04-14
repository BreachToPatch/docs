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

## 6. Writing your patch

A patch is a set of files placed inside your own folder:

```
{machine-slug}/writeups/patch/{your-username}/
├── writeup.md            ← required, use templates/writeup_patch.md
├── docker-compose.yml    ← required, the patched service definition
└── <any other files>     ← any other files you modified (Dockerfile, configs, source code, etc.)
```

The CI rebuilds and starts the services using **your** `docker-compose.yml`
and any files it references. Anything you don't include keeps its archived
version.

### Example: patching Bee-Path (CVE-2021-41773)

The vulnerability comes from Apache 2.4.49's broken path normalization.
Two valid approaches:

1. **Upgrade Apache** — change the base image in `app/apache/Dockerfile` from
   `httpd:2.4.49` to `httpd:2.4.51` or later. Include the modified `Dockerfile`
   in your patch folder.
2. **Disable mod_cgi** — the RCE part of the exploit needs `mod_cgi` enabled.
   Comment out the `LoadModule cgi_module` line in `httpd.conf`. Note: this
   downgrades functionality, so the SLA check must still pass.

Both are valid as long as services stay UP and the exploit returns exit code 1.

---

## 7. Testing your patch locally

Before opening a PR, verify your patch yourself.

### Quick test (Docker only — what the CI does)

From inside your patch folder:

```bash
cd {machine-slug}/writeups/patch/{your-username}
docker compose up -d --build
```

Wait for services to start (5 to 30 seconds), then check they respond:

```bash
curl http://localhost:80/    # should return HTTP 200 or 403
```

Now run the locked exploit against your patched version:

```bash
TARGET_IP=localhost TARGET_PORT=80 \
  python3 ../../../../{machine-slug}/exploit/exploit.py
echo "Exit code: $?"
```

| Exit code | Meaning |
|-----------|---------|
| `1` | ✅ Vulnerability is patched. You're ready to submit. |
| `0` | ❌ Exploit still works. Your patch is insufficient. |
| `2` | ❌ Service unreachable. Your patch broke something. |

Cleanup:

```bash
docker compose down --volumes
```

### Full test (Vagrant — closer to real player environment)

Optional but recommended for complex patches. Modify the archived
`docker-compose.yml` to point to your patched files, then `vagrant up` from
the archive folder. Run the exploit against `192.168.56.10`.

---

## 8. Submitting your patch

### Step 1 — Fork machines-public

If you haven't already, fork `https://github.com/BreachToPatch/machines-public`
and clone it.

```bash
git clone https://github.com/{your-username}/machines-public.git
cd machines-public
```

### Step 2 — Create your branch

```bash
git checkout -b patch/{your-github-username}/{machine-slug}
```

Example:

```bash
git checkout -b patch/bob/vuln-apache-path-traversal
```

The CI rejects any branch that does not exactly match `patch/{username}/{machine-slug}`.

### Step 3 — Add your files

```
{machine-slug}/writeups/patch/{your-username}/
├── writeup.md            ← your patch writeup
├── docker-compose.yml    ← your patched service definition
└── <any modified files>  ← e.g., Dockerfile, httpd.conf, app source files
```

You may **only** create files inside your own folder. The CI rejects PRs that
touch anything outside `{machine-slug}/writeups/patch/{your-username}/`.

### Step 4 — Commit and push

```bash
git add {machine-slug}/writeups/patch/{your-username}/
git commit -m "patch: {machine-slug} by {your-username}"
git push origin patch/{your-username}/{machine-slug}
```

### Step 5 — Open the Pull Request

Open the PR on GitHub from your branch into `BreachToPatch/machines-public:main`.

---

## 9. What the CI does to your patch

In sequence:

1. **Branch name check** — must be `patch/{username}/{machine-slug}`
2. **Folder ownership check** — folder must match your GitHub username
3. **Scope check** — PR must touch only files inside your own folder
4. **Required files check** — `writeup.md` and `docker-compose.yml` must exist
5. **Secret scan** — TruffleHog scans your patch for accidentally committed secrets
6. **Build and start** — `docker compose up -d --build` from your folder
7. **SLA check** — services must respond within 60 seconds
8. **Regression test** — the locked `exploit/exploit.py` is replayed
9. **Comment posted** — the result lands on your PR

The four possible outcomes:

| Outcome | Meaning |
|---------|---------|
| ✅ PATCH ACCEPTED | Exploit blocked + services UP. Maintainer reviews and merges. |
| ❌ Vulnerability not fixed | Exploit still returns exit `0` — patch insufficient |
| ❌ SLA Violation | Services stopped responding after applying the patch |
| ⚠️ Validation Error | Unexpected CI error — a maintainer will investigate |

Push a fix to the same branch — the CI reruns automatically.

---

## 10. Scoring

| Action | Points |
|--------|--------|
| Patch validated and merged | **75 pts** |
| Writeup quality bonus (maintainer-awarded) | up to 25 pts |

Earned badges:

| Badge | How to earn it |
|-------|---------------|
| 🛡️ Defender | Get a patch validated |
| 🏰 Fort Knox | Your patch resists 3 consecutive re-pwn attempts |

---

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