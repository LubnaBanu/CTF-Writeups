# CTF Writeups

Penetration test writeups for a set of vulnerable CTF-style lab machines, taken from reconnaissance through to full root compromise. Each writeup includes methodology, step-by-step proof-of-concept commands, evidence, findings, and the captured flags.

All targets are intentionally-vulnerable lab/training VMs on private, host-only networks (`192.168.56.0/24`) — not production systems. Flags, lab credentials and IP addresses are included as-is, since they carry no value outside these lab environments and disclosing them is standard practice for CTF writeups.

---

## Writeups

| Machine | Target OS | Vulnerability Chain | Result |
|---|---|---|---|
| [SunsetNoonTide](./sunsetnoontide-report.md) | Debian 10 "Buster" | UnrealIRCd 3.2.8.1 backdoor (CVE-2010-2075) → weak root password | Full root compromise |
| [First Stage](./first-stage-report.md) | Ubuntu 18.04.4 LTS | Leaked SSH credentials via exposed web file → PwnKit (CVE-2021-4034) | Full root compromise |
| [Dina](./dina-report.md) | Ubuntu 12.04 LTS | Leaked backup archive → playSMS RCE (CVE-2017-9909) → sudo misconfiguration | Full root compromise |

A PDF version of each report (e.g. `dina-report.pdf`) is included alongside its Markdown version, with the original formatted layout and terminal screenshots.

---

## Summary

### SunsetNoonTide
An exposed, trojaned UnrealIRCd 3.2.8.1 service allowed fully unauthenticated remote command execution via CVE-2010-2075. Outbound egress filtering blocked the default reverse-shell payload, requiring a pivot to a bind shell. Despite an otherwise well-hardened filesystem and kernel, a trivially weak root password (`root`) allowed instant privilege escalation via `su`.

### First Stage
Extension-aware directory brute-forcing uncovered an internal notes file referencing a second, still-exposed file that leaked plaintext SSH credentials. Standard local privilege-escalation checks (sudo, SUID, LXD group membership) came up empty, but a vulnerable `pkexec`/Polkit install (CVE-2021-4034, "PwnKit") provided a direct path to root once a statically-linked exploit resolved a glibc version mismatch.

### Dina
A chain of individually moderate weaknesses — leaked password hints via `robots.txt`, a weakly-protected backup archive, and a weak application password — led to authenticated access to an outdated playSMS 1.4 install, vulnerable to CVE-2017-9909 remote code execution. Post-exploitation enumeration uncovered a severe sudo misconfiguration (`www-data` permitted to run `perl` as root with NOPASSWD), providing instant escalation to full root.

---

## Methodology (common across all three)

1. **Reconnaissance** — host discovery, port scanning, service fingerprinting (Nmap, arp-scan)
2. **Enumeration** — web/service enumeration to find an initial entry point
3. **Initial Access** — exploiting the discovered vulnerability or leaked credentials
4. **Post-Exploitation** — enumerating the compromised host for privilege-escalation vectors
5. **Privilege Escalation** — exploiting the identified weakness to obtain root
6. **Proof of Compromise** — capturing and documenting the flag(s)

---

## Disclaimer

These writeups document authorized, isolated lab/CTF engagements only. All targets are deliberately vulnerable training machines. Nothing here targets or was performed against production systems, and none of this constitutes an invitation or instruction to attack systems you don't own or have explicit permission to test.

---

*Prepared by Lubna Banu*
