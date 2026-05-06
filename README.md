# HTB Writeups — Josef Basner

![Platform](https://img.shields.io/badge/Platform-Hack%20The%20Box-9FEF00)
![Focus](https://img.shields.io/badge/Focus-Pentesting-red)
![Language](https://img.shields.io/badge/Lang-Deutsch-blue)
![Machines](https://img.shields.io/badge/Machines-7-success)

> Sammlung meiner deutschsprachigen Writeups zu retired HackTheBox-Maschinen.
> Fokus auf vollständig dokumentierte Angriffsketten — Recon, Foothold, Privilege
> Escalation, plus dokumentierter Methodik und genutzter Tools.

---

## Maschinen-Index

| Machine                       | OS      | Difficulty | Highlights                                                              |
|-------------------------------|---------|------------|-------------------------------------------------------------------------|
| [Support](Support.md)         | Windows | Easy       | SMB Anonymous · .NET Reverse Engineering · LDAP Recon · RBCD / S4U      |
| [Browsed](Browsed.md)         | Linux   | Medium     | Browser-Extension Abuse · Localhost Pivot · Python `__pycache__`-Hijack |
| [Phantom](Phantom.md)         | Windows | Medium     | SMB Recon · Password Spraying · VeraCrypt · RBCD / S4U                  |
| [Sendai](Sendai.md)           | Windows | Medium     | RID Bruteforce · `STATUS_PASSWORD_MUST_CHANGE` · GMSA · ADCS ESC4 → ESC1 |
| [Dump](Dump.md)               | Linux   | Hard       | PHP Argument Injection · SQLite Credential Leak · AppArmor / `tcpdump` Abuse |
| [Snapped](Snapped.md)         | Linux   | Hard       | Nginx-UI CVE-2026-27944 · TOCTOU Race (`snap-confine`)                  |
| [JobTwo](Job_Two.md)          | Windows | Hard       | Macro Phishing · hMailServer Config Decrypt · Veeam B&R Exploit         |

---

## Aufbau eines Writeups

Jeder Writeup folgt derselben Struktur — Konsistenz schlägt Stilistik:

1. **Synopsis** — Kurzfassung der Angriffskette in 2-3 Sätzen
2. **Enumeration** — Nmap, Service-Discovery, Recon
3. **Foothold** — Initial Access, mit kompletten PoC-Schritten
4. **Privilege Escalation** — Pfad zur User → Root / SYSTEM-Eskalation
5. **Angriffskette Kurzüberblick** — Bullet-Liste der Schritte für schnelles Wiederfinden

---

## Demonstrierte Skill-Areas

**Active Directory:**
RBCD / S4U2Self / S4U2Proxy · GMSA-Abuse · ADCS ESC4 → ESC1 · Password Spraying ·
RID Bruteforce · LDAP-Enumeration · BloodHound Path Analysis

**Linux Exploitation:**
Argument Injection · AppArmor Bypass · Sudo-Misconfiguration (`tcpdump`) ·
Python Bytecode Hijacking · TOCTOU Race Conditions · `snap-confine`-Exploit

**Web & Application:**
Browser-Extension Abuse als Recon-Werkzeug · Localhost-only Service Pivot ·
Command Injection · Phishing mit Office-Macros

**Cryptanalysis & Reverse Engineering:**
.NET-Decompiling (ILSpy) · XOR-Reverse · VeraCrypt Cracking · SQLite-/SDF-Cred-Extraction

**CVE-Exploitation:**
CVE-2026-27944 (Nginx-UI Auth-Bypass) · CVE-2026-3888 (`snap-confine` TOCTOU) ·
Veeam B&R RCE

---

## Tooling

| Phase                    | Tools                                                              |
|--------------------------|--------------------------------------------------------------------|
| Recon                    | Nmap, NSE-Scripts, FFUF, gobuster                                  |
| Web                      | Burp Suite Pro, sqlmap, ffuf                                       |
| AD-Enumeration           | nxc / crackmapexec, BloodHound, ldapsearch, kerbrute               |
| Kerberos                 | Impacket (GetUserSPNs, GetNPUsers, getST, secretsdump), Rubeus     |
| ADCS                     | Certipy                                                             |
| Linux PrivEsc            | LinPEAS, pspy, GTFOBins-Lookup                                     |
| Windows PrivEsc          | WinPEAS, PowerUp, Seatbelt, PrintSpoofer / GodPotato               |
| Reverse Engineering      | ILSpy, dnSpy, Ghidra                                               |
| Lateral Movement         | Evil-WinRM, psexec.py, wmiexec.py, smbexec.py                      |

---

## Methodisches Prinzip

```
Recon → Enumerate → Hypothesize → Validate → Escalate → Report
```

Jeder Writeup zeigt nicht nur **was** funktioniert hat, sondern **warum** ich
diesen Weg gewählt habe und welche Hypothesen falsch waren. Das ist der Unterschied
zwischen einem Walkthrough und einer Methodik-Dokumentation.

---

## Verwandte Repositories

- [Study-Notes](https://github.com/JBMTP07/Study-Notes) — strukturierte Methodik-Notes (Linux/Windows PrivEsc, AD, Web, Reporting)
- [JBMTP07.github.io](https://github.com/JBMTP07/JBMTP07.github.io) — Portfolio

---

## Hinweis

Alle Writeups beziehen sich ausschließlich auf **retired** HTB-Maschinen
und dienen Ausbildungs- und Methodik-Dokumentationszwecken.

---

**Josef Basner** · Penetration Tester · Red Team Operator · Security Researcher
[github.com/JBMTP07](https://github.com/JBMTP07) · [jbmtp07.github.io](https://jbmtp07.github.io)
