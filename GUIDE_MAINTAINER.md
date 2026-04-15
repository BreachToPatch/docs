# 🔧 Maintainer Guide — Internal operations of BreachToPatch

> Audience: members of the `btop-sources` and `BreachToPatch` GitHub organizations with Maintainer or Owner roles.
> This guide covers the operational side of running the platform — reviewing PRs, managing secrets, archiving sources, releasing new versions, and handling abuse.

---

## 1. Roles and access

| Org | Role | What it grants |
|-----|------|---------------|
| `btop-sources` (private) | Owner | Full admin (orgs, billing, repos) — keep to 2 people maximum |
| `btop-sources` (private) | Maintainer | Read/write on `machines-sources`, can merge PRs there |
| `BreachToPatch` (public) | Owner | Full admin |
| `BreachToPatch` (public) | Maintainer | Can merge PRs in any public repo, can push directly to `main` (use sparingly) |
| `BreachToPatch` (public) | Member | No special privileges (community contributors don't need to be members) |

**Onboarding a new maintainer:**

1. Owner invites them to both orgs as Maintainer
2. They accept both invitations
3. They generate a Personal Access Token (classic, scope `read:packages`) so they can pull CI images locally if needed
4. Owner adds their GitHub username to relevant `CODEOWNERS` files

---

## 2. Reviewing a pwn PR (Red Team submission)

**The CI does the heavy lifting.** When a player opens a pwn PR in `machines-public`:

1. `validate-pwn.yml` runs automatically
2. It posts an `✅ ACCEPTED` or `❌ REJECTED` comment

**Your job:**

| If CI says... | Do this |
|---------------|---------|
| ✅ ACCEPTED | Read the writeup. If quality is reasonable, merge. Optionally award 1-25 pts writeup quality bonus by adding the label `bonus:N` (e.g. `bonus:15`) before merging. |
| ❌ REJECTED | Do nothing — the player must fix and push again. The CI will re-validate. |
| ⚠️ Validation Error | Investigate the failed Action run. Usually a transient ghcr.io issue or runner timeout. Re-run the workflow. If it fails again, ping the on-call owner. |

### Writeup quality criteria

A good writeup explains the **kill chain** (recon → vuln identification → exploitation → flag), not just the final payload. Reject low-effort writeups (one-liners, dump of `nmap` output without explanation) by closing the PR with a short comment pointing to `templates/writeup_attack.md`. The exploit can stay locked even if the writeup is rejected — the `exploit.py` validation is independent.

### First Blood handling

The CI automatically:
- Copies the player's `exploit.py` to `{machine-slug}/exploit/exploit.py`
- Adds a header comment identifying the First Blood author
- Commits to the PR branch with `[skip ci]` to avoid loops

You do **not** need to do this manually. After merge, the locked exploit is on `main` and becomes the regression test for all future patches.

---

## 3. Reviewing a patch PR (Blue Team submission)

**The CI does the heavy lifting.** When a player opens a patch PR:

1. `validate-patch.yml` runs automatically
2. It rebuilds and starts the patched services from the player's folder
3. It runs the SLA check + replays the locked exploit
4. It posts `✅ PATCH ACCEPTED` or `❌ PATCH REJECTED`

**Your job (when ACCEPTED):**

1. Read the patch writeup — does it explain the **root cause** correctly?
2. Read the diff — does it look honest, or does it just block the specific exploit payload?
3. Run a quick mental test: "Could a slightly different exploit defeat this patch?" If yes, comment on the PR asking the player to address the root cause.
4. If the patch is solid, merge.

After merge:
- `auto-tag.yml` creates the next version tag (e.g. `vuln-apache-path-traversal/v2`)
- You then perform the [post-patch release flow](#5-post-patch-release-flow)

---

## 4. Releasing a new machine

When a new machine is ready in `btop-sources/machines-sources`:

### Step 1 — Build and verify the CI image

After your PR is merged, `build-and-push.yml` automatically builds and pushes
`ghcr.io/btop-sources/{machine-name}-ci:latest`. Verify it works locally:

```bash
docker pull ghcr.io/btop-sources/{machine-name}-ci:latest
docker run --rm -p 80:80 ghcr.io/btop-sources/{machine-name}-ci:latest
# In another terminal:
TARGET_IP=localhost TARGET_PORT=80 \
  python3 btop-sources/machines-sources/{machine-name}/exploit/exploit.py
# Should print FLAG_OBTAINED:FLAG{...} and exit 0
```

### Step 2 — Create the public folder

In `BreachToPatch/machines-public`, create `{public-slug}/` with:

```
{public-slug}/
├── README.md            # Public description with flag hash
├── ci/docker-compose.yml # Pulls the CI image
└── exploit/.gitkeep     # Empty until First Blood
```

Use `vuln-apache-path-traversal/` as the structural template.

### Step 3 — Publish the Vagrant box

Players need a downloadable Vagrant box. From a freshly provisioned VM:

```bash
cd btop-sources/machines-sources/{machine-name}
vagrant up
vagrant package --output {machine-name}.box
```

Upload `{machine-name}.box` as an asset on a new GitHub Release in
`btop-sources/machines-sources` tagged `{machine-name}/v1`.

> Boxes are large (200 MB to 2 GB). Use Git LFS only if absolutely necessary —
> GitHub Releases handle files up to 2 GB without LFS, which is preferred.

### Step 4 — Publish the download link

Update the public `README.md` `### Download` section to point to the release URL.

### Step 5 — Add the flag value as a GitHub Secret

For patch validation, the CI needs the flag value as a build-arg secret. In
`BreachToPatch/machines-public` settings → Secrets and variables → Actions:

```
Name:  {MACHINE_NAME_UPPERCASE}_FLAG  (e.g., BEE_PATH_FLAG)
Value: FLAG{your_unique_flag_value_here}
```

Update `validate-patch.yml` to inject this secret if the patch CI needs it (the bee-path workflow already does so via `env.FLAG_VALUE`).

### Step 6 — Announce

Post a short announcement in the project Discord/Twitter pointing players to the new machine. Status on the website automatically becomes **🔴 Red Team Active**.

---

## 5. Post-patch release flow

When a patch PR is merged in `machines-public`:

### Step 1 — Acknowledge the auto-tag

`auto-tag.yml` creates `{public-slug}/v{N+1}` automatically. No action needed.

### Step 2 — Update the private machine

In `btop-sources/machines-sources/{machine-name}/`:

1. Apply the same patch to the source files (or cherry-pick the diff from the player's PR)
2. Generate a **new** flag value (`FLAG{...}`)
3. Update `provision/playbook.yml` `vars.flag` and `Dockerfile.ci`
4. Compute the new SHA-256 hash
5. Bump version in `CHANGELOG.md`
6. Open a PR, merge it
7. `build-and-push.yml` rebuilds the CI image with the new flag

### Step 3 — Update the public README

In `machines-public/{public-slug}/README.md`:
- Update the **Flag Hash (SHA-256)** section with the new hash
- Bump the version header (e.g. "Bee-Path (v2)")
- Move status back to **🔴 Red Team Phase Active**
- Update the **Download** link to the new Release

### Step 4 — Reset the locked exploit

The locked `exploit/exploit.py` was the Red Teamer's First Blood exploit for
v1. After a patch is merged, **it is now obsolete** — the new version is no
longer vulnerable to it. Delete it (or move it to a `history/v1/` folder)
so the machine is once again open for First Blood on v2.

### Step 5 — Update the GitHub Secret

Update `{MACHINE_NAME_UPPERCASE}_FLAG` in `machines-public` Secrets to the
new flag value.

### Step 6 — Publish the new Vagrant box

Same as [Step 3 of release flow](#step-3--publish-the-vagrant-box) — but tag
`{machine-name}/v2`.

---

## 6. Archiving sources after First Blood

When a machine is pwned for the first time:

### Step 1 — Sanitize the sources

Copy the entire `btop-sources/machines-sources/{machine-name}/` folder.
Remove or replace:

- The flag value in `provision/playbook.yml` → replace with `FLAG{REDACTED}` placeholder
- The flag in `Dockerfile.ci` → replace with placeholder
- The reference exploit in `exploit/exploit.py` → **remove entirely** (the
  player's locked exploit is already in `machines-public` for that purpose)
- Any internal-only comments referencing maintainer Slack channels, internal
  URLs, etc.

### Step 2 — Push to machines-archive

Push the sanitized folder to `BreachToPatch/machines-archive/{public-slug}-v{version}/`:

```bash
git checkout -b archive/{public-slug}-v{version}
mkdir -p {public-slug}-v{version}
cp -r /path/to/sanitized/{machine-name}/* {public-slug}-v{version}/
git add {public-slug}-v{version}
git commit -m "archive: {public-slug} v{version} sources after first pwn"
git push origin archive/{public-slug}-v{version}
```

Open a PR, merge.

### Step 3 — Confirm the README

Each archived folder must have a `README.md` explaining:
- Which version this is
- When First Blood happened (link to the merged pwn PR)
- The vulnerability category and a short description (the public README hash field is removed)

---

## 7. Managing GitHub Actions secrets

### `btop-sources/machines-sources`

| Secret | Purpose |
|--------|---------|
| `GITHUB_TOKEN` | Auto-provided by GitHub. Used to push CI images to ghcr.io |

### `BreachToPatch/machines-public`

| Secret | Purpose |
|--------|---------|
| `GHCR_PULL_TOKEN` | Personal Access Token (classic, scope `read:packages`) generated by an Owner of `btop-sources`. Lets the public CI pull the private CI images. |
| `{MACHINE_NAME}_FLAG` | The flag value of each machine, used by `validate-patch.yml` to inject into the patched container build. One per machine. |

### `BreachToPatch/leaderboard`

No secrets needed — the leaderboard update workflow lives in `machines-public` and pushes to `leaderboard` using a deploy key or GITHUB_TOKEN with `contents: write` permission across orgs.

### Rotation policy

- `GHCR_PULL_TOKEN`: rotate every 90 days. When you rotate, also delete the old token from the issuing user's GitHub settings.
- `{MACHINE_NAME}_FLAG`: rotate at every machine version bump (Step 5 of the post-patch release flow).
- Never log secrets in workflow output. The workflows here use `set +e` and explicit redaction — review any new workflow you add for the same hygiene.

---

## 8. Branch protection settings

| Repo | `main` protection |
|------|-------------------|
| `btop-sources/machines-sources` | Require PR before merge, require CI to pass, no direct push |
| `BreachToPatch/machines-public` | Require PR before merge, require `validate-pwn` and `validate-patch` to pass when applicable |
| `BreachToPatch/machines-archive` | Require PR before merge |
| `BreachToPatch/leaderboard` | No protection — auto-update workflow commits directly |
| `BreachToPatch/web-site` | Require PR before merge |
| `BreachToPatch/docs` | Require PR before merge |

For solo work during early development, "Required reviewers" can stay at 0 — branch protection still enforces "PR before merge", which keeps history clean.

---

## 9. Handling abuse and bad-faith submissions

| Situation | Response |
|-----------|----------|
| Someone submits a pwn PR for another player | The CI rejects automatically (folder/PR-author mismatch). No action needed. |
| Someone submits a writeup with no real explanation | Close the PR with a polite comment pointing to the writeup template. Do not merge — the locked exploit can be rolled back if it was already locked by an unmerged PR (revert the auto-commit). |
| Someone submits a patch that hardcodes blocking the exact CI exploit payload | Reject. Comment that the patch must address the root cause. If the player insists or repeats this on multiple machines, escalate to org Owner. |
| Someone submits a backdoor in a patch | TruffleHog usually catches this. If you spot it manually, immediately close the PR, ban the user from the org, and write a short post-mortem in `docs/incidents/`. |
| A player publishes a flag publicly to grief a fresh machine | Rotate the flag (Step 5 of post-patch release flow) and announce v2. Optionally ban the publisher. |
| GitHub Action minutes exhaustion | Public repos get unlimited free minutes for public workflows. If we hit limits on private workflows in `machines-sources`, switch the build job to use `paths` filters or trigger only on tag push instead of every commit. |

---

## 10. Operational runbook

### Daily

- Skim new PRs in `machines-public`. CI does the work; just merge what's green.
- Check Discord/Twitter for player questions.

### Weekly

- Audit org membership in both orgs. Remove inactive collaborators.
- Verify ghcr.io image storage usage stays within free tier.

### Monthly

- Review the open Issues across all repos. Triage and label.
- Check that all `*_FLAG` secrets are still valid (machines released ≥ 1 year ago should be considered for retirement or refresh).

### Quarterly

- Rotate `GHCR_PULL_TOKEN`.
- Audit `CODEOWNERS` files across all repos.
- Update this guide with anything that changed.

---

## 11. Emergency contacts

> Maintained in private — see the `#maintainers` channel.

If you're the only maintainer awake and you can't resolve a critical issue
(e.g. a leaked private repo, a sustained DoS on an Action runner), the
nuclear option is: **archive the affected public repo** (Settings → Archive).
This freezes it instantly and stops all CI activity. You can un-archive later
once the issue is resolved.

---

## 12. References

- [GUIDE_REDTEAM.md](GUIDE_REDTEAM.md) — what players follow when attacking
- [GUIDE_BLUETEAM.md](GUIDE_BLUETEAM.md) — what defenders follow when patching
- [GUIDE_CREATOR.md](GUIDE_CREATOR.md) — what creators follow when building machines
- `MANIFESTO.md` — the project's foundational principles
- All workflow files in `BreachToPatch/machines-public/.github/workflows/` and `btop-sources/machines-sources/.github/workflows/`