# Patch Writeup — {Machine Name}

> **Player:** @{your-github-username}
> **Machine:** {machine-slug}
> **Target version:** v{X.Y} (the version you are patching)
> **Date:** YYYY-MM-DD

---

## 1. Summary

One paragraph (2-4 sentences). What was the vulnerability, what is your fix, and why does it work?

> Example: "Bee-Path v1 ships Apache 2.4.49, vulnerable to CVE-2021-41773 (path traversal + RCE via mod_cgi). My patch upgrades the base image to `httpd:2.4.51`, which contains the upstream fix for path normalization, neutralizing both the traversal and the RCE escalation."

---

## 2. Root cause analysis

Why does the vulnerability exist? Go beyond the symptom.

- The **specific** code path or configuration that creates the weakness
- Why the original design or implementation made this possible
- What an attacker needs (preconditions, access, etc.) to exploit it

> A patch that only blocks the exact payload of the locked exploit will **not**
> count as a real fix during maintainer review, even if the CI accepts it.
> Address the root cause.

---

## 3. The patch

What did you change, file by file?

### `{file-1}` — {what was changed}

What was the issue:

```
... show the original problematic snippet ...
```

What you changed it to:

```
... show your fix ...
```

Why this works: ...

### `{file-2}` — {what was changed}

(repeat for each file)

---

## 4. Why this patch is correct

Argue your case:

- It addresses the **root cause** ({...})
- It does not break any existing service (the SLA check passes)
- It does not introduce new attack surface
- It is consistent with upstream / vendor recommendations

If you considered alternative fixes, briefly say why you picked this one.

---

## 5. Local verification

Show that you tested before submitting.

### Services start cleanly

```bash
cd writeups/patch/{your-username}
docker compose up -d --build
```

Output (or screenshot description): all services are `Up`.

### SLA check passes

```bash
curl -i http://localhost:80/
# HTTP/1.1 200 OK   (or 403 — both pass the CI SLA check)
```

### Locked exploit fails

```bash
TARGET_IP=localhost TARGET_PORT=80 \
  python3 ../../../exploit/exploit.py
echo "Exit code: $?"
# EXPLOIT_FAILED: ...
# Exit code: 1
```

Exit code `1` is the only acceptable outcome here. Exit `0` means the patch
is insufficient. Exit `2` means you broke the service.

---

## 6. Trade-offs

Be honest about what your patch sacrifices.

- Functionality removed: ...
- Performance impact: ...
- Configuration changes that could affect downstream users: ...

If the patch is a clean upstream upgrade with no downsides, just say so.

---

## 7. References

- Original CVE: https://nvd.nist.gov/vuln/detail/CVE-XXXX-YYYY
- Upstream fix commit / release notes: ...
- Related advisories: ...

---

## 8. Files in this patch

List every file in your patch folder and what it is:

```
{machine-slug}/writeups/patch/{your-username}/
├── writeup.md            ← this file
├── docker-compose.yml    ← patched service definition
├── Dockerfile            ← (if modified)
└── ...                   ← any other modified files
```

---

> Reminder: your patch folder must contain `writeup.md` and `docker-compose.yml`
> at minimum. Any file you don't include keeps its original archived version.
> The CI rejects PRs that touch files outside `{machine-slug}/writeups/patch/{your-username}/`.