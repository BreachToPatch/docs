# ⚙️ Creator Guide — How to build a new BreachToPatch machine

> Audience: community members who want to design and submit a new vulnerable machine.
> Prerequisites: solid grasp of Linux, Docker, at least one CVE or vulnerable application you want to ship, and access to the private `btop-sources` organization (you must be invited as a Maintainer first — request access via a GitHub Issue on `BreachToPatch/docs`).

---

## 1. The mindset of a good machine

A good BreachToPatch machine teaches **one** clear vulnerability cleanly. It is:

- **Reproducible**: defined entirely by code (Vagrantfile + Ansible + Docker Compose), no binary blobs.
- **Realistic**: based on a real CVE or a believable misconfiguration, not a contrived puzzle.
- **Patchable**: there must exist at least one valid patch a Blue Teamer can apply that fixes the root cause without breaking the service.
- **Bounded**: one main vulnerability, one flag at `/root/flag.txt`. Side puzzles or rabbit holes are discouraged for Easy/Medium machines.
- **CI-compatible**: the vulnerability must be exploitable from outside the container (web, SSH, FTP, application-level RCE, etc.). Kernel-level or Layer 2 attacks need a self-hosted runner — out of scope for first-time contributors.

---

## 2. The 3-layer architecture

Every machine has the same architecture:

| Layer | Tool | Purpose |
|-------|------|---------|
| **VM** | Vagrant + VirtualBox | Creates an isolated Ubuntu VM on the player's machine |
| **Provisioning** | Ansible (`ansible_local` provisioner) | Installs Docker, places the flag, starts services |
| **Services** | Docker Compose | Runs the actual vulnerable application |

This means a machine is **100% defined by text files** — no `.ova`, no binary blobs.

---

## 3. File structure

A new machine in `btop-sources/machines-sources` (private) follows this structure exactly:

```
{machine-slug}/
├── README.md                        # Internal notes for maintainers
├── CHANGELOG.md                     # Version history
├── Vagrantfile                      # VM definition (OS, network, resources)
├── Dockerfile.ci                    # CI-only image (flag baked in)
├── provision/
│   ├── playbook.yml                 # Main Ansible playbook (defines flag)
│   └── roles/
│       ├── docker/tasks/main.yml    # Installs Docker Engine
│       └── flag/tasks/main.yml      # Writes /root/flag.txt
├── app/
│   ├── docker-compose.yml           # Orchestrates the services
│   └── {service-folder}/            # One folder per service (apache, db, backend, etc.)
│       ├── Dockerfile
│       └── ...service files...
├── tests/
│   ├── health_check.sh              # SLA: services respond
│   └── service_test.py              # Functional endpoint tests
└── exploit/
    └── exploit.py                   # MAINTAINER reference exploit (private — never published)
```

The reference machine `bee-path/` in `btop-sources/machines-sources` is the canonical template. Copy its structure when starting a new machine.

---

## 4. Naming conventions

| Item | Convention | Example |
|------|-----------|---------|
| Internal name | Short, evocative, lowercase-hyphenated | `bee-path` |
| Public slug | `vuln-{technology}-{vuln-type}` | `vuln-apache-path-traversal` |
| Private folder | `{internal-name}/` | `bee-path/` |
| Public folder | `{public-slug}/` | `vuln-apache-path-traversal/` |
| Archive folder | `{public-slug}-v{version}/` | `vuln-apache-path-traversal-v1.0/` |
| CI image | `ghcr.io/btop-sources/{internal-name}-ci:latest` | `ghcr.io/btop-sources/bee-path-ci:latest` |
| Version tag | `{public-slug}/v{N}.{M}` | `vuln-apache-path-traversal/v1.0` |

---

## 5. The Vagrantfile

Use Ubuntu 22.04 (`ubuntu/jammy64`) as the base. Use the IP `192.168.56.10`
unless you have a strong reason otherwise (some firewalls block other ranges).

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/jammy64"
  config.vm.hostname = "{machine-name}"
  config.vm.network "private_network", ip: "192.168.56.10"

  config.vm.provider "virtualbox" do |vb|
    vb.name   = "btop-{machine-name}"
    vb.memory = "1024"   # bump only if your services need more
    vb.cpus   = 1
    vb.gui    = false
  end

  config.vm.provision "ansible_local" do |ansible|
    ansible.playbook          = "playbook.yml"
    ansible.provisioning_path = "/vagrant/provision"
    ansible.extra_vars = { ansible_python_interpreter: "/usr/bin/python3" }
  end
end
```

---

## 6. The Ansible playbook — where the flag lives

The flag is hardcoded in `provision/playbook.yml` under `vars.flag`. It must
**never** appear in any public repository.

```yaml
---
- name: Provision {machine-name}
  hosts: all
  become: true

  vars:
    # ----------------------------------------------------------------------
    # FLAG — never commit this to a public repository
    # SHA-256 of this value is published in machines-public README only.
    # ----------------------------------------------------------------------
    flag: "FLAG{your_unique_flag_value_here}"

    app_src:  "/vagrant/app"
    app_dest: "/opt/{machine-name}"

  roles:
    - docker
    - flag

  tasks:
    - name: Create app directory
      file:
        path: "{{ app_dest }}"
        state: directory
        mode: "0755"

    - name: Copy app files into the VM
      copy:
        src:  "{{ app_src }}/"
        dest: "{{ app_dest }}/"

    - name: Start the vulnerable service
      shell: "docker compose up -d --build"
      args:
        chdir: "{{ app_dest }}"
```

The reusable `docker` and `flag` roles live in `provision/roles/`. Copy them
verbatim from `bee-path` — they are stable.

### Generating the flag hash

The public README displays only the SHA-256 hash of the flag. Generate it:

```bash
echo -n "FLAG{your_unique_flag_value_here}" | sha256sum
```

> Use `echo -n` (no trailing newline). The CI compares the hash of the
> player-submitted flag against this exact value. A trailing newline will
> never match.

Copy the resulting 64-character hash into the public README under the
**Flag Hash (SHA-256)** section.

---

## 7. The Docker Compose stack

Inside `app/docker-compose.yml`, define every service the machine needs.
Mount `/root/flag.txt` read-only into any container that needs to expose it
(typically just one service):

```yaml
services:
  {service-name}:
    build:
      context: ./{service-folder}
      dockerfile: Dockerfile
    container_name: {machine-name}-{service-name}
    ports:
      - "80:80"
    volumes:
      - /root/flag.txt:/root/flag.txt:ro
    restart: unless-stopped
```

Use any number of services (frontend, backend, db, etc.) — but keep the
attack surface focused on the one vulnerability you want to teach.

---

## 8. The reference exploit (private, maintainer-only)

`exploit/exploit.py` lives in the **private** repo. It proves the vulnerability
exists and works as intended at the moment the machine is published. It is
**never** shared with players — they write their own.

It must respect the same exit code contract as player exploits:

| Exit code | Meaning |
|-----------|---------|
| `0` | Vulnerability present, flag obtained |
| `1` | Vulnerability absent or patched |
| `2` | Service unreachable |

See [`templates/exploit_script.py`](templates/exploit_script.py) for the
reference structure.

---

## 9. Health checks (SLA)

`tests/health_check.sh` is what the patch-validation CI uses to check that a
patch did not break the service. It must exit `0` if all services are healthy.

```bash
#!/usr/bin/env bash
set -euo pipefail

TARGET_IP="${TARGET_IP:-192.168.56.10}"
TARGET_PORT="${TARGET_PORT:-80}"

HTTP_CODE=$(curl --silent --max-time 5 \
  --output /dev/null --write-out "%{http_code}" \
  "http://${TARGET_IP}:${TARGET_PORT}/")

if [[ "${HTTP_CODE}" == "200" || "${HTTP_CODE}" == "403" ]]; then
  echo "[+] Service UP (HTTP ${HTTP_CODE})"
  exit 0
else
  echo "[-] Service DOWN (HTTP ${HTTP_CODE})"
  exit 1
fi
```

Add functional tests for each service in `tests/service_test.py` (Python, no
external dependencies — use `urllib.request` from the standard library).

---

## 10. The CI image — `Dockerfile.ci`

For CI speed, every machine ships a CI-only Docker image with the flag
**baked in**. This image is built and pushed to the private GitHub Container
Registry by `.github/workflows/build-and-push.yml` in `machines-sources`.

```dockerfile
# Dockerfile.ci — CI-only image
FROM {base-image}

# ... same setup as the player image ...

# Bake the flag into the image (CI only — never use this pattern in production)
RUN mkdir -p /root \
    && echo "FLAG{your_unique_flag_value_here}" > /root/flag.txt \
    && chmod 400 /root/flag.txt

EXPOSE {port}
```

Add a job to `.github/workflows/build-and-push.yml` to build and push your
machine's image:

```yaml
build-and-push-{machine-name}:
  name: Build {machine-name} CI image
  runs-on: ubuntu-latest
  permissions:
    contents: read
    packages: write
  steps:
    - uses: actions/checkout@v4
    - uses: docker/login-action@v3
      with:
        registry: ghcr.io
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}
    - uses: docker/build-push-action@v5
      with:
        context: ./{machine-name}
        file: ./{machine-name}/Dockerfile.ci
        push: true
        tags: |
          ghcr.io/btop-sources/{machine-name}-ci:latest
          ghcr.io/btop-sources/{machine-name}-ci:${{ github.sha }}
```

---

## 11. The public counterpart in `machines-public`

Once the private side is ready, create the public folder in `BreachToPatch/machines-public/{public-slug}/`:

```
{public-slug}/
├── README.md           # Public description, target info, flag hash, download link
├── ci/
│   └── docker-compose.yml   # Pulls the private CI image (referenced by validate-pwn.yml)
└── exploit/
    └── .gitkeep        # Empty until First Blood — CI auto-locks the exploit here
```

The `ci/docker-compose.yml` pulls the CI image:

```yaml
services:
  {service-name}:
    image: ghcr.io/btop-sources/{machine-name}-ci:latest
    container_name: {machine-name}-ci
    ports:
      - "80:80"
    restart: unless-stopped
```

The public README must include the SHA-256 hash on a line of its own (the
CI extracts it via the regex `^[a-f0-9]{64}$`):

```markdown
### Flag Hash (SHA-256)

```
{the-64-char-hash-here}
```
```

Use `bee-path` as a structural template — the section ordering is what the
website parses.

---

## 12. Submission checklist

Before opening your PR on `btop-sources/machines-sources`:

- [ ] Folder `{machine-name}/` follows the canonical structure
- [ ] `Vagrantfile` boots cleanly on a fresh machine (`vagrant destroy -f && vagrant up`)
- [ ] `provision/playbook.yml` runs without errors and writes `/root/flag.txt` mode 0400
- [ ] `app/docker-compose.yml` brings all services UP after `vagrant up`
- [ ] `tests/health_check.sh` returns 0 against the running VM
- [ ] `exploit/exploit.py` returns exit 0 and the correct flag against the VM
- [ ] `Dockerfile.ci` builds and the resulting image, run with `docker run -p 80:80`, also yields the flag via `exploit.py`
- [ ] `.github/workflows/build-and-push.yml` has a job for the new machine
- [ ] SHA-256 of the flag is generated and ready to publish
- [ ] Public README drafted for `machines-public/{public-slug}/`
- [ ] No flag value appears anywhere in the public README, public docker-compose, or public exploit folder

---

## 13. The submission flow

1. Open a PR on `btop-sources/machines-sources` with your new machine folder
2. A maintainer reviews the design, code quality, and educational value
3. If accepted: maintainer merges → `build-and-push.yml` builds and publishes the CI image
4. Maintainer creates the public folder in `machines-public` and publishes the Vagrant box as a GitHub Release on `machines-sources`
5. Machine goes live with status **🔴 Red Team Active** on the website
6. You earn ⚙️ **Creator** badge + **150 pts**

---

## 14. After your machine goes live

- **First pwn**: a Red Teamer validates a pwn → CI auto-locks their exploit at
  `machines-public/{public-slug}/exploit/exploit.py`.
- **Sources revealed**: a maintainer copies the sanitized source code (flag
  stripped) from `machines-sources/{machine-name}/` to
  `machines-archive/{public-slug}-v1.0/`.
- **Blue Team phase opens**: defenders fork the archive and propose patches.
- **Patches accepted**: each merge bumps the version
  (`{public-slug}/v1.1`, `v1.2`, ...) and triggers a new CI image build.
- You stay listed as the original author of the machine forever.

---

## 15. Tips for great machines

- **Pick a real CVE.** Players love learning about historical CVEs they can
  later recognize in the wild. Avoid fictional vulnerabilities.
- **Document the intended path.** In your private `README.md` and reference
  exploit, write the intended kill chain step by step. This helps maintainers
  review and helps future you when you patch your own machine three years later.
- **Test on a clean machine.** What works on your dev machine often fails on
  someone else's. `vagrant destroy -f && vagrant up` from a fresh clone is the
  real test.

---

Build with care. Every machine you ship will be hacked, then patched, then
re-hacked. That is the cycle.