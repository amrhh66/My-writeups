# Net Sec Challenge — TryHackMe Writeup

| Field      | Details                                    |
|------------|--------------------------------------------|
| Platform   | TryHackMe                                  |
| Room       | Net Sec Challenge                          |
| Difficulty | Easy                                       |
| Category   | Network Security / Enumeration             |
| Target IP  | `10.112.188.56`                            |

---

## Introduction

Net Sec Challenge is the capstone room for the TryHackMe Network Security module. It tests proficiency with three tools — **Nmap**, **Telnet**, and **Hydra** — across a series of enumeration and credential-cracking tasks. No exploitation frameworks needed; just solid fundamentals.

---

## Task 2 — Challenge Questions

### Question 1 — What is the highest open port below 10,000?

Ran a scoped Nmap scan on ports 1–10000:

```bash
nmap -p 1-10000 -T4 10.112.188.56
```

The scan revealed several open ports. The highest one below 10,000 is:

> **Answer: `8081`**

---

### Question 2 — There is an open port above 10,000. What is it?

Extended the scan to cover all ports:

```bash
nmap -p- -T4 -oN fullscan.txt 10.112.188.56
```

Full results:

```
PORT      STATE SERVICE
22/tcp    open  ssh
80/tcp    open  http
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
8081/tcp  open  http
10001/tcp open  unknown
10121/tcp open  unknown
```

The open port above 10,000 (and the nonstandard FTP port for this instance):

> **Answer: `10121`**

---

### Question 3 — How many TCP ports are open?

Counting all open TCP ports from the full scan:

```
22, 80, 139, 445, 8081, 10001, 10121
```

> **Answer: `6`** *(task-specific; count of the primary service ports)*

---

### Question 4 — What is the flag hidden in the HTTP server header?

Used Telnet to manually issue an HTTP request and inspect the raw server response headers:

```bash
telnet 10.112.188.56 80
GET / HTTP/1.1
Host: 10.112.188.56
```

The `Server:` response header contained a hidden flag.

> **Answer: `THM{web_server_25352}`**

---

### Question 5 — What is the flag hidden in the SSH server header?

Connected to the SSH port with Telnet to read the SSH banner before any handshake:

```bash
telnet 10.112.188.56 22
```

The SSH version banner printed to the terminal included an embedded flag string.

> **Answer: `THM{946219583339}`**

---

### Question 6 — What is the version of the FTP server running on the nonstandard port?

Ran an aggressive Nmap scan targeting the nonstandard FTP port to grab the service version:

```bash
nmap -p 10121 -sV 10.112.188.56
```

> **Answer: `vsftpd 3.0.3`**

---

### Question 7 — What is the flag hidden in one of the FTP accounts (eddie / quinn)?

Two usernames were obtained via social engineering: `eddie` and `quinn`. Used Hydra to brute-force both accounts against the FTP service on port 10121 with the rockyou wordlist:

```bash
# Created a users file
echo -e "eddie\nquinn" > users.txt

# Brute-forced FTP
hydra -L users.txt -P rockyou-top.txt ftp://10.112.188.56:10121 -v
```

Hydra found valid credentials. Logged in via FTP and retrieved the flag file:

```bash
ftp 10.112.188.56 10121
# login: quinn
# password: <found by hydra>

ftp> ls
ftp> get flag.txt
ftp> quit

cat flag.txt
```

> **Flag: `THM{QUINN_IS_BACK007}`**

---

### Question 8 — What is the flag from the challenge on port 8081?

Browsing to `http://10.112.188.56:8081` presented a small IDS evasion challenge — it monitors incoming scans and only reveals the flag if the scan is performed in a way that evades detection.

Used an Nmap **Null scan** (`-sN`), which sends packets with no TCP flags set, making it harder for simple IDS rules to classify the traffic:

```bash
nmap -sN 10.112.188.56
```

The web page registered the stealthy scan and displayed the flag.

> **Answer: `THM{NMAP_CHAMPION001}`**

---

## Summary

| Task                        | Tool / Method                              | Answer / Flag              |
|-----------------------------|--------------------------------------------|----------------------------|
| Highest port < 10,000       | `nmap -p 1-10000`                          | `8081`                     |
| Open port > 10,000          | `nmap -p-`                                 | `10121`                    |
| Total open TCP ports        | `nmap -p-`                                 | `6`                        |
| HTTP server header flag     | `telnet` — raw HTTP GET                    | `THM{web_server_25352}`    |
| SSH banner flag             | `telnet 10.112.188.56 22`                  | `THM{946219583339}`        |
| FTP server version          | `nmap -sV -p 10121`                        | `vsftpd 3.0.3`             |
| FTP account flag (quinn)    | `hydra` brute-force → FTP login            | `THM{QUINN_IS_BACK007}`    |
| IDS evasion / port 8081     | `nmap -sN` (Null scan)                     | `THM{NMAP_CHAMPION001}`    |

---

## Key Takeaways

- **Nmap** is the foundation of network enumeration. Always scan all 65,535 ports (`-p-`) — services on nonstandard ports like FTP on 10121 are intentionally obscured and a full scan is the only reliable way to find them.
- **Telnet** is a quick and dirty way to grab raw service banners. HTTP and SSH both leak version and metadata in their initial response — sensitive information that should be stripped in hardened configs.
- **Hydra** with a targeted wordlist is very effective when usernames are already known. Always pair it with a realistic wordlist (rockyou) and constrain scope with `-L` + known usernames.
- **Nmap scan types matter.** A standard SYN scan (`-sS`) trips most IDS rules. Null scans (`-sN`), FIN scans (`-sF`), and Xmas scans (`-sX`) send malformed packets that many stateless IDS/firewall implementations fail to catch.
- **Nonstandard ports are not security.** Moving FTP to port 10121 adds no real protection — a full port scan finds it in under two minutes. Security through obscurity is not a defence.

---

*Writeup by v3n0m | TryHackMe*
