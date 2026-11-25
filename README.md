# CTF Write-ups

Write-ups of the machines I solve while practising offensive security (eJPT certified).

Each one covers recon, exploitation, privilege escalation, root causes with fixes, and what I learned. That includes the dead ends and the shortcuts I took. Flags and passwords are redacted.

| Machine | Platform | Difficulty | Key techniques |
|---|---|---|---|
| [Simple CTF](tryhackme-simple-ctf.md) | TryHackMe | Easy | Anonymous FTP, CMS Made Simple SQLi (CVE-2019-9053), hash cracking, `sudo vim` |
| [RootMe](tryhackme-rootme.md) | TryHackMe | Easy | Upload filter bypass (`.phtml`), PHP reverse shell, SUID python |
| [Pickle Rick](tryhackme-pickle-rick.md) | TryHackMe | Easy | Creds in HTML source and robots.txt, command panel filter bypass, `sudo` NOPASSWD ALL |
| [Startup](tryhackme-startup.md) | TryHackMe | Easy | Writable anonymous FTP → web shell, PCAP analysis (Wireshark), cron + writable script |
| [Bounty Hacker](tryhackme-bounty-hacker.md) | TryHackMe | Easy | Anonymous FTP, SSH brute force with a found wordlist, `sudo tar` |
