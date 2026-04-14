# 🔴 Red Team Guide — How to pwn a BreachToPatch machine

> Audience: players who want to attack a machine, capture its flag, and submit a validated pwn.
> Prerequisites: a GitHub account, a computer that can run a virtual machine, and basic command-line familiarity.

---

## 1. How a pwn works on BreachToPatch

The full flow, end to end:

```
1. Install Vagrant + VirtualBox on your machine
2. Download a machine from BreachToPatch (one Vagrantfile + provisioning files)
3. Boot the VM locally with `vagrant up`
4. Attack the running machine — find /root/flag.txt
5. Verify your flag against the public SHA-256 hash
6. Write your exploit.py following the standard format
7. Open a Pull Request on machines-public with your writeup + exploit
8. The CI automatically validates your submission and posts a comment on your PR
9. A maintainer merges → you appear on the leaderboard
```

If you are the **first** player to validate a pwn on a given machine, you earn the
🩸 **First Blood** badge and your `exploit.py` is automatically locked as the
regression test for every future Blue Team patch.

---

## 2. Setting up your environment

You need two tools installed locally:

| Tool | What it does | Download |
|------|--------------|----------|
| **VirtualBox** | The hypervisor — actually runs the VM | https://www.virtualbox.org/wiki/Downloads |
| **Vagrant** | Automates VM creation from a `Vagrantfile` | https://developer.hashicorp.com/vagrant/install |

Verify both are installed:

```bash
VBoxManage --version
vagrant --version
```

You should see version numbers, not errors. If you get errors, reinstall and reboot.

---

## 3. Downloading and booting a machine

Each machine has a public page at:

```
https://github.com/BreachToPatch/machines-public/tree/main/{machine-slug}
```

Example for the first machine: `vuln-apache-path-traversal`.

**Step 1 — Get the machine code.** The download link lives in the machine's
README under the **Download** section (it points to a GitHub Release of the
private `btop-sources/machines-sources` repo containing the `Vagrantfile` and
provisioning files).

**Step 2 — Boot the VM.** From the folder containing the `Vagrantfile`:

```bash
vagrant up
```

The first run takes 5 to 10 minutes — Vagrant downloads the Ubuntu 22.04 base
image, creates the VM, and Ansible installs Docker and the vulnerable services
inside it.

**Step 3 — Confirm it's running.** Each machine documents its target IP and
port. For `vuln-apache-path-traversal`:

```bash
curl http://192.168.56.10/
```

You should see a response. If you see "connection refused", wait one more
minute (services are still starting) then retry.

**Useful Vagrant commands:**

| Command | What it does |
|---------|--------------|
| `vagrant up` | Boot the VM (first run also provisions it) |
| `vagrant halt` | Shut down the VM cleanly |
| `vagrant destroy` | Delete the VM completely |
| `vagrant reload` | Restart the VM |
| `vagrant ssh` | SSH into the VM as the `vagrant` user |
| `vagrant status` | Show current state |

---

## 4. Rules of engagement

- The vulnerability lives **inside the VM**. Do not attack `vagrant.com`,
  `github.com`, or any other infrastructure — they are not part of the challenge.
- The flag file is always at `/root/flag.txt` with mode `0400` (readable only by root).
  Reading it requires achieving root access on the VM.
- The flag string always follows the format `FLAG{...}`.
- You may use any tool you want (Burp, sqlmap, Metasploit, custom scripts).
  But your final `exploit.py` submission must be a self-contained Python script
  that reproduces the attack without manual steps.

---

## 5. Verifying your flag

Once you have the flag, verify its SHA-256 hash matches the public hash listed
in the machine's README:

```bash
echo -n "FLAG{your_flag_value_here}" | sha256sum
```

> Use `echo -n` (no trailing newline). The hash will not match if you include a newline.

If the hash matches the public one, you have the right flag. If not, you either
copied it wrong or read the wrong file.

---

## 6. Writing your exploit script

<a id="exploit-format"></a>

### The standard format (mandatory)

The CI runs your exploit with two environment variables, `TARGET_IP` and
`TARGET_PORT`, then inspects its exit code and stdout. The script must:

| Outcome | Exit code | Stdout | Meaning |
|---------|-----------|--------|---------|
| Flag obtained | `0` | `FLAG_OBTAINED:FLAG{...}` | Pwn validated |
| Vulnerability absent (patched) | `1` | (any) | Used by patch validation — exploit is blocked |
| Service unreachable | `2` | (any) | Network/SLA error |

Use the official template at [`templates/exploit_script.py`](templates/exploit_script.py)
as your starting point. Do not change the `__main__` block — exit codes are
contractual with the CI.

### Local testing

Before opening a PR, test your script against your local VM:

```bash
TARGET_IP=192.168.56.10 TARGET_PORT=80 python3 exploit.py
echo "Exit code: $?"
```

You should see `FLAG_OBTAINED:FLAG{...}` and exit code `0`.

The CI will run the same script with `TARGET_IP=localhost` against a Docker
container that mirrors the VM. If your script depends on something specific to
your local network, it will fail there.

---

## 7. Submitting your pwn

Every submission is a Pull Request to the `machines-public` repository.

### Step 1 — Fork and clone

Fork `https://github.com/BreachToPatch/machines-public` to your own account,
then clone your fork:

```bash
git clone https://github.com/{your-username}/machines-public.git
cd machines-public
```

### Step 2 — Create your branch

The branch name is **strict**. The CI rejects any PR whose branch does not
match the convention.

```bash
git checkout -b attack/{your-github-username}/{machine-slug}
```

Example:

```bash
git checkout -b attack/alice/vuln-apache-path-traversal
```

### Step 3 — Add your two files

You may only create files inside your own folder. The CI verifies that:
- The branch name matches `attack/{your-username}/{machine-slug}`
- The folder name matches your GitHub username exactly
- The PR touches **only** files inside your folder

```
{machine-slug}/writeups/attack/{your-username}/
├── writeup.md       ← use templates/writeup_attack.md
└── exploit.py       ← use templates/exploit_script.py
```

For Bee-Path with username `alice`:

```
vuln-apache-path-traversal/writeups/attack/alice/
├── writeup.md
└── exploit.py
```

### Step 4 — Commit and push

```bash
git add vuln-apache-path-traversal/writeups/attack/alice/
git commit -m "pwn: vuln-apache-path-traversal by alice"
git push origin attack/alice/vuln-apache-path-traversal
```

### Step 5 — Open the Pull Request

Open the PR on GitHub from your branch into `BreachToPatch/machines-public:main`.
The CI starts within seconds.

---

## 8. What the CI does to your PR

Within ~2 minutes you will see one of these comments:

**✅ ACCEPTED**
```
🐝 BtoP — PWN Validation: ✅ ACCEPTED
Machine: `vuln-apache-path-traversal`
Player: @alice
Your exploit ran successfully and the flag hash matches.
🩸 First Blood! Your exploit has been locked as the regression test for all future patches.
```
A maintainer reviews your writeup and merges. You appear on the leaderboard.

**❌ REJECTED**
The comment lists the exact reason. Common rejections:

| Reason | Fix |
|--------|-----|
| Invalid branch name | Rename branch to `attack/{username}/{machine-slug}` |
| PR author does not match folder name | Use your own GitHub username for the folder |
| PR touches files outside your allowed folder | Remove changes to other files |
| Missing `writeup.md` or `exploit.py` | Add the missing file |
| Exit code is not `0` | Your exploit failed against the CI environment — debug locally |
| `FLAG_OBTAINED:` line not found | Make sure your script prints the flag in the exact format |
| Hash mismatch | Wrong flag value submitted |

Push a fix to the same branch — the CI reruns automatically.

---

## 9. Scoring

| Action | Points |
|--------|--------|
| First Blood (first pwn on a machine) | **100 pts** |
| Subsequent pwn | 50 pts |
| Writeup quality bonus (maintainer-awarded) | up to 25 pts |

Earned badges:

| Badge | How to earn it |
|-------|---------------|
| 🩸 First Blood | First player to pwn a machine |
| ✍️ Scribe | Submit 5 writeups |
| 🐝 Queen Bee | Top of the leaderboard for 30 consecutive days |

Full scoring table: see [`leaderboard/README.md`](https://github.com/BreachToPatch/leaderboard).

---

## 10. After your PR is merged

- Your username appears in `leaderboard.json` (auto-updated within seconds).
- The website at `https://breachtopatch.github.io` reflects your new score.
- If you got First Blood, the machine moves from **Red Team Active** to
  **Blue Team Active** — Blue Teamers can now fork the sources from
  `machines-archive` and propose patches.
- Your writeup stays in `machines-public/{machine-slug}/writeups/attack/{your-username}/`
  forever — a public proof of your work, linked to your GitHub profile.

---

## 12. Templates

- [`templates/writeup_attack.md`](templates/writeup_attack.md) — your writeup structure
- [`templates/exploit_script.py`](templates/exploit_script.py) — your exploit skeleton

---

Good luck. Hack ethically.