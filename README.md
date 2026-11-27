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
| [Agent Sudo](tryhackme-agent-sudo.md) | TryHackMe | Easy | User-Agent fuzzing, FTP brute force, steganography (binwalk, steghide), ZIP cracking, sudo `!root` bypass (CVE-2019-14287) |
| [Ignite](tryhackme-ignite.md) | TryHackMe | Easy | Fuel CMS unauthenticated RCE (CVE-2018-16763), reverse shell through PHP `system()`, DB creds in config file, password reuse |
| [Wgel CTF](tryhackme-wgel-ctf.md) | TryHackMe | Easy | HTML comment leak, nested directory enumeration, exposed SSH private key, `sudo wget` file read |
| [Brooklyn Nine Nine](tryhackme-brooklyn-nine-nine.md) | TryHackMe | Easy | Two paths: anonymous FTP + SSH brute force → `sudo less`; steganography cracking (stegseek) → `sudo nano` |
