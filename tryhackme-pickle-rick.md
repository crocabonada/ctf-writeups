# TryHackMe — Pickle Rick

| | |
|---|---|
| **Platform** | TryHackMe |
| **Difficulty** | Easy |
| **OS** | Linux (Ubuntu) |
| **Date** | 2026-11-24 |
| **Skills** | Web recon, source code review, command injection via a web panel, filter bypass, Linux file handling, sudo misconfiguration |

> The three "ingredients" (flags) and the password are redacted, following TryHackMe's write-up guidelines.

## Summary

The web server leaks a **username in an HTML comment** and a **password in `robots.txt`**. Together they unlock a "command panel" that runs OS commands as `www-data`. The panel blocks a few commands like `cat`, but other file readers get through. `www-data` can run **anything as root without a password** (`(ALL) NOPASSWD: ALL`), so the last ingredient in `/root` is one `sudo` away.

## 1. Reconnaissance

```bash
sudo nmap -p- --min-rate 1000 -T4 -oN ports.txt <TARGET>
nmap -sV -sC -p 22,80 -oN services.txt <TARGET>
```

| Port | Service | Notes |
|---|---|---|
| 22 | OpenSSH 8.2p1 (Ubuntu) | |
| 80 | Apache 2.4.41 (Ubuntu) | Title "Rick is sup4r cool" |

The site's stack wasn't obvious (no tell-tale cookie), so gobuster tried several extensions:

```bash
gobuster dir -u http://<TARGET> -w /usr/share/wordlists/dirb/common.txt -x php,html,txt -o web.txt
```

The results included a **login portal** (`login.php`) and `robots.txt`.

## 2. Credentials from the web root

- **Page source** (Ctrl+U): an HTML comment gives the username: `Username: <redacted>`.
- **`robots.txt`**: there are no `Disallow` rules, only a single odd word. At first I wrote it off as "nothing interesting". It turned out to be the **password**.

The username plus the `robots.txt` word log in to the portal.

## 3. Command panel as www-data

The portal (`portal.php`) runs OS commands on the server:

```bash
whoami     # www-data
ls -la     # Sup3rS3cretPickl3Ingred.txt, clue.txt, assets, denied.php, login.php, portal.php, robots.txt ...
```

**Filter bypass:** `cat` returns *"Command disabled to make it hard for future PICKLEEEE RICCCKKKK"*, and `nano` is blocked as well. The filter blocklists specific command names, but Linux has plenty of other ways to print a file:

```bash
less Sup3rS3cretPickl3Ingred.txt      # ingredient 1
```

Other options would have been `head`, `tail`, `tac`, `grep . file`, `strings` or `base64`. `clue.txt` says to look around the file system for the next ingredient.

**Panel gotcha:** every command runs in a **fresh shell**, so `cd ..` does nothing for the next command. Either pass absolute paths or chain commands in one line (`cd /home/rick; ls -la`).

```bash
ls /home                 # rick  ubuntu
ls -la /home/rick        # -rwxrwxrwx ... second ingredients
less "/home/rick/second ingredients"     # ingredient 2
```

The file name contains a **space**. The shell splits words on spaces, so the name has to be quoted (`"second ingredients"`) or the space escaped (`second\ ingredients`). Replacing it with `_` would point to a file that doesn't exist.

## 4. Privilege escalation — sudo without a password

```bash
sudo -l
# User www-data may run the following commands on <host>:
#     (ALL) NOPASSWD: ALL
```

`www-data` can run any command as root with no password. A few things that **don't** work in this panel, and why:

- `sudo su` opens an interactive root shell, but the panel is one-shot, so it just hangs or exits.
- `sudo cd /root` fails because `cd` is a shell builtin, not a program, so `sudo` can't run it.
- Changing root's password would work, but on a real engagement it locks the client out of their own server and leaves an obvious trace. It's also unnecessary.

Simply put `sudo` in front of each command:

```bash
sudo ls -la /root
sudo less /root/<ingredient-file>      # ingredient 3
```

## Attack chain

```
nmap → Apache on :80
     → page source → username
     → robots.txt → password
     → login.php → command panel (www-data)
     → cat blocked → less → ingredient 1
     → /home/rick/"second ingredients" → ingredient 2
     → sudo -l = (ALL) NOPASSWD: ALL → sudo less /root/... → ingredient 3
```

## Root causes and remediation

| Issue | Fix |
|---|---|
| Username in an HTML comment | Strip comments and debug notes from production HTML |
| Password in `robots.txt` | `robots.txt` is public by design; never put anything sensitive in it (and don't rely on it to hide paths) |
| Web panel that runs OS commands | Remove it; if it really must exist, use a strict allowlist of fixed actions instead of free-form shell input |
| Command blocklist (`cat`, `nano`…) | Blocklists always miss alternatives (`less`, `tac`, `grep`…); use an allowlist or remove the feature |
| `www-data` with `(ALL) NOPASSWD: ALL` | The web server user should have no sudo rights; grant only the specific commands needed |

## Lessons learned

- **Read every recon artefact properly.** I dismissed `robots.txt` because it had no paths, and it held the password.
- **The page source is part of recon.** Ctrl+U takes two seconds.
- **One-shot command panels aren't shells:** no persistent `cd`, no interactive programs (`nano`, `su`). Use absolute paths and chain commands with `;`.
- **Builtins vs binaries:** `sudo cd` fails because `cd` belongs to the shell itself.
- **Filenames with spaces:** quote them or escape the space.
- **Least damage:** having root access doesn't mean you should change passwords. Read what you need and leave everything else as it was.
