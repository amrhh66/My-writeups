# Summit — TryHackMe Writeup

| Field      | Details                              |
|------------|--------------------------------------|
| Platform   | TryHackMe                            |
| Room       | Summit                               |
| Difficulty | Easy                                 |
| Category   | Threat Intelligence / Malware Defence / Security Operations |

---

## Introduction

Summit is a guided blue-team room that puts you in the shoes of a security analyst at PicoSecure. The scenario: a threat actor is actively trying to compromise the organisation, and your job is to work through the **Pyramid of Pain** — a threat intelligence framework — to progressively disrupt the attacker by blocking their indicators at each level of the pyramid. Each task builds on the last, forcing the attacker to adapt and costing them more effort with every step.

The room is structured around a simulated malware sandbox and a firewall/blocklist management portal. No exploitation required — this is pure defensive security and threat intelligence work.

---

## Task 2 — Analysing the Sample (Hash)

### Flag 1

The first task involves submitting a malware sample to the sandbox and reviewing its report. The goal is to identify the file hash and add it to the blocklist.

Opened the malware sandbox portal and uploaded the provided sample. The sandbox generated a report including the SHA-256 hash of the file.

Added the hash to the hash blocklist in the security portal. The system confirmed the block and issued the flag.

> **Flag: `THM{f3cbf08151a11a6a331db9c6cf5f4fe4}`**

---

## Task 3 — IP Address Blocking

### Flag 2

With the hash blocked, the attacker adapted and attempted to establish a C2 (command-and-control) connection. The sandbox report showed outbound network connections — specifically a C2 callback to a hard-coded IP address embedded in the sample.

Reviewed the **Network Connections** section of the sandbox report to extract the C2 IP. Added the IP to the IP blocklist in the firewall portal.

> **Flag: `THM{2ff48a3421a938b388418be273f4806d}`**

---

## Task 4 — Domain Blocking

### Flag 3

Blocking the IP forced the attacker to register a domain for their C2 instead of relying on a bare IP. The new sample in the sandbox resolved a domain for its callback rather than connecting directly to an IP.

Extracted the domain name from the sandbox's DNS resolution log. Added it to the domain blocklist.

> **Flag: `THM{4eca9e2f61a19ecd5df34c788e7dce16}`**

---

## Task 5 — Network Artefacts (Host-based)

### Flag 4

Domain blocked. The attacker shifted to using dynamic DNS and started relying on network-level artefacts — specifically, a distinctive URI pattern and User-Agent string embedded in the malware's HTTP requests to blend into legitimate traffic.

Reviewed the **HTTP Requests** section of the sandbox report and identified the suspicious User-Agent string and URL pattern used for C2 beaconing. Created a detection rule in the portal to flag and block traffic matching those network artefacts.

> **Flag: `THM{c956f455fc076aea829799c0876ee399}`**

---

## Task 6 — Host Artefacts

### Flag 5

The attacker modified the network-level indicators but left behind host-based artefacts — registry keys, file paths, and process names created by the malware during execution.

Reviewed the **Host Activity** section of the sandbox report, including:
- Registry keys created/modified by the sample
- Files dropped to disk (paths and filenames)
- Suspicious process spawning chains

Added the relevant host artefacts (registry keys and file paths) to the detection rules in the portal.

> **Flag: `THM{46b21c4410e47dc5729ceadef0fc722e}`**

---

## Task 7 — TTPs (Techniques, Tactics, and Procedures)

### Flag 6

At the top of the Pyramid of Pain sit **TTPs** — the attacker's actual behaviour and methodology. These are the hardest indicators to change because they represent *how* the attacker operates, not just *what tools* they use.

Mapped the attacker's observed behaviour to MITRE ATT&CK techniques using the sandbox report and previous analysis. Identified the specific ATT&CK technique IDs covering the persistence mechanism, the C2 communication method, and the defence evasion approach used throughout the campaign.

Submitted the mapped TTPs in the portal to complete the Pyramid of Pain exercise.

> **Flag: `THM{c8951b2ad24bbcbac60c16cf2c83d92c}`**

---

## Summary

| Task                       | Level of Pyramid     | Action Taken                            | Flag                                       |
|----------------------------|----------------------|-----------------------------------------|--------------------------------------------|
| Malware hash               | Trivial              | Hash blocklist                          | `THM{f3cbf08151a11a6a331db9c6cf5f4fe4}`   |
| C2 IP address              | Easy                 | IP blocklist / firewall rule            | `THM{2ff48a3421a938b388418be273f4806d}`   |
| C2 domain                  | Simple               | Domain blocklist / DNS sinkhole         | `THM{4eca9e2f61a19ecd5df34c788e7dce16}`   |
| Network artefacts          | Annoying             | URI/User-Agent detection rule           | `THM{c956f455fc076aea829799c0876ee399}`   |
| Host artefacts             | Annoying             | Registry key / file path detection rule | `THM{46b21c4410e47dc5729ceadef0fc722e}`   |
| TTPs (MITRE ATT&CK)        | Tough                | ATT&CK technique mapping                | `THM{c8951b2ad24bbcbac60c16cf2c83d92c}`   |

---

## Key Takeaways

- The **Pyramid of Pain** is a practical framework for prioritising threat intelligence. Blocking a hash is trivial for an attacker to bypass (recompile the binary) — blocking TTPs forces a full re-architecture of their attack methodology.
- **Malware sandboxes** are a core blue-team tool. They surface IOCs (hashes, IPs, domains, URI patterns, registry changes) without risking live infrastructure.
- **MITRE ATT&CK** provides a common language for describing attacker behaviour. Mapping detections to technique IDs means your rules survive tooling changes by the adversary — you're detecting *behaviour*, not artefacts.
- Every level up the pyramid costs the attacker more time, money, and operational security risk. The goal isn't necessarily to stop every attack forever — it's to make attacking *you* expensive enough that the attacker moves on.
- **Defence in depth** mirrors the pyramid: hash blocks, network rules, and behavioural detections should all be layered together, not treated as alternatives.

---

*Writeup by v3n0m | TryHackMe*
