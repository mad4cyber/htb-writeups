# Hack The Box: DevArea Writeup

> Öffentliches Writeup für GitHub. Flags, API-Credentials und Secrets sind absichtlich nicht enthalten.

## Überblick

DevArea ist eine Linux-Maschine mit mehreren zusammenhängenden Services: FTP, Apache, ein Apache-CXF-SOAP-Service, Hoverfly und eine interne SysWatch-Anwendung. Der initiale Zugriff erfolgt über eine Apache-CXF-XOP/MTOM-LFI-Schwachstelle, mit der Service-Units und Hoverfly-Credentials ausgelesen werden. Danach ermöglicht die Hoverfly-Middleware-API Codeausführung als `dev_ryan`. Die Privilege Escalation kombiniert eine lokale SysWatch-Web-Command-Injection als `syswatch` mit einem sudo-erlaubten Root-Log-Reader und einer Symlink-Kette.

## Recon

Initialer Scan:

```bash
nmap -Pn -sC -sV -oN nmap.txt 10.129.57.23
nmap -Pn -p- --min-rate 5000 -oN nmap-allports.txt 10.129.57.23
```

Interessante Dienste:

```text
21/tcp    FTP / vsftpd 3.0.5, anonymous erlaubt
22/tcp    SSH
80/tcp    Apache, Redirect auf devarea.htb
8080/tcp  Jetty / Apache CXF SOAP Service
8500/tcp  Hoverfly Proxy
8888/tcp  Hoverfly Dashboard/API
```

Der Webdienst auf Port 80 leitete auf `devarea.htb` weiter. Der wichtigste erste Fund lag allerdings auf FTP.

## FTP: Java-Service-Artefakt

Anonymous FTP enthielt ein Java-Artefakt:

```text
employee-service.jar
```

Das JAR wurde lokal entpackt und inspiziert:

```bash
jar tf employee-service.jar
jar xf employee-service.jar
```

Relevante Erkenntnisse:

```text
Apache CXF 3.2.14
Jetty SOAP Service
Endpoint: /employeeservice
WSDL: /employeeservice?wsdl
```

Der SOAP-Service bot eine `submitReport`-Methode mit einem Report-Objekt an. Wichtige Felder waren unter anderem `employeeName`, `department`, `content` und `confidential`.

## Foothold Teil 1: Apache CXF XOP/MTOM LFI

Apache CXF 3.2.14 ist gegen CVE-2022-46364 anfällig. Über XOP/MTOM `xop:Include` konnte der SOAP-Service lokale Dateien lesen.

Beispielhafte Ziele für die LFI-Enumeration:

```text
/etc/passwd
/proc/self/environ
/proc/self/cmdline
/etc/systemd/system/employee-service.service
/etc/systemd/system/hoverfly.service
/etc/systemd/system/syswatch-web.service
```

Wichtige Ergebnisse:

- Der SOAP-Service lief im Kontext `dev_ryan`.
- Die User-Flag war für den SOAP-Service explizit über systemd `InaccessiblePaths` blockiert.
- Die Hoverfly-Service-Unit enthielt die Startparameter und API-Zugangsdaten.
- SysWatch-Service-Units gaben Hinweise auf eine lokale Admin-/Monitoring-Anwendung.

Secrets aus der Service-Unit wurden für das öffentliche Writeup redigiert.

## Foothold Teil 2: Hoverfly Middleware RCE

Hoverfly war auf zwei Ports erreichbar:

```text
8500/tcp  Proxy
8888/tcp  Dashboard/API
```

Mit den per LFI gefundenen Credentials konnte ein API-Token über den Hoverfly-Token-Endpoint geholt werden:

```text
POST /api/token-auth
```

Die API zeigte Hoverfly v1.11.3. Interessant war besonders die Middleware-Funktionalität. Über die API konnte lokale Python-Middleware gesetzt und Hoverfly in einen Modus gebracht werden, in dem Proxy-Requests durch die Middleware liefen.

Minimaler Test der Codeausführung:

```bash
id
```

Ausführungskontext:

```text
uid=1001(dev_ryan) gid=1001(dev_ryan) groups=1001(dev_ryan)
```

Damit war der Foothold als `dev_ryan` erreicht. Die User-Flag wurde anschließend aus dem Home-Verzeichnis gelesen; der Wert ist im öffentlichen Writeup absichtlich nicht enthalten.

## Enumeration als dev_ryan

`sudo -l` zeigte eine sehr relevante Berechtigung:

```text
NOPASSWD: /opt/syswatch/syswatch.sh
```

Einige gefährliche Subcommands waren explizit ausgeschlossen, der Hauptwrapper durfte aber mit erlaubten Argumenten via sudo ausgeführt werden.

Im Home-Verzeichnis von `dev_ryan` lag außerdem ein SysWatch-Quellcode-Archiv:

```text
/home/dev_ryan/syswatch-v1.zip
```

Nach dem Entpacken wurde klar, dass SysWatch aus mehreren Komponenten bestand:

- `/opt/syswatch/syswatch.sh` als CLI-/sudo-Wrapper
- `/opt/syswatch/syswatch_gui/app.py` als lokale Flask-Web-GUI
- `/opt/syswatch/logs` als Log-Verzeichnis
- `/etc/syswatch.env` als Konfigurationsdatei

## Privilege Escalation Teil 1: SysWatch Web Command Injection

Die SysWatch-Web-GUI lief lokal auf Port 7777 als Benutzer `syswatch`.

Die Flask-Konfiguration nutzte ein Secret aus:

```text
/etc/syswatch.env
```

Da dieses Secret aus dem `dev_ryan`-Kontext lesbar war, konnte eine gültige Flask-Session für den Admin-Bereich erzeugt werden.

In der Route `/service-status` wurde ein Service-Name validiert und anschließend in einem Shell-Befehl verwendet:

```text
systemctl status --no-pager <service>
```

Die Validierung blockte zwar Zeichen wie Slash, Punkt, Semikolon, Ampersand und Großbuchstaben, erlaubte aber Command Substitution mit `$()`.

Damit war Codeausführung als `syswatch` möglich.

## Privilege Escalation Teil 2: Root-Log-Reader und Symlink-Kette

Der sudo-erlaubte SysWatch-Wrapper konnte Logs als root lesen:

```bash
sudo /opt/syswatch/syswatch.sh logs <logfile>
```

Die Log-Funktion enthielt eine Symlink-Prüfung, aber die Implementierung konnte über eine zweistufige Symlink-Kette umgangen werden:

```text
/opt/syswatch/logs/root.txt       -> /root/root.txt
/opt/syswatch/logs/log-alerts.log -> root.txt
```

Warum das funktionierte:

- `syswatch` konnte im Log-Verzeichnis schreiben.
- Der sudo-Wrapper las die angeforderte Logdatei als root.
- Die Prüfung erkannte nicht zuverlässig, dass die zweite Symlink-Auflösung letztlich auf `/root/root.txt` zeigte.

Anschließend gab der erlaubte Root-Log-Reader den Inhalt der Root-Flag aus. Der konkrete Flag-Wert ist im öffentlichen Writeup nicht enthalten.

## Kill Chain

```text
Anonymous FTP
    -> employee-service.jar
    -> Apache CXF 3.2.14 SOAP Service
    -> CVE-2022-46364 XOP/MTOM LFI
    -> Read systemd service units
    -> Recover Hoverfly API credentials
    -> Hoverfly middleware API RCE as dev_ryan
    -> Read user.txt
    -> sudo /opt/syswatch/syswatch.sh
    -> Forge Flask session using SysWatch secret
    -> /service-status command injection as syswatch
    -> Create log symlink chain
    -> sudo SysWatch log reader follows chain as root
    -> Read root.txt
```

## Cleanup

Nach der Ausnutzung wurden die Änderungen zurückgenommen:

- Hoverfly-Middleware geleert
- Hoverfly-Modus zurück auf den ursprünglichen Simulationsmodus gesetzt
- Temporäre Dateien entfernt
- SysWatch-Logdatei aus Backup wiederhergestellt
- Temporäre Symlink-Kette entfernt

## Takeaways

- Build-Artefakte auf FTP können interne Framework-Versionen, Endpoints und Datenmodelle offenlegen.
- Systemd-Service-Units enthalten oft kritische Informationen wie User-Kontext, Schutzmechanismen und Startparameter.
- Eine LFI wird deutlich gefährlicher, wenn sie Service-Credentials oder lokale Admin-Oberflächen offenlegt.
- Middleware-/Proxy-Systeme sollten nicht mit weitreichenden API-Rechten und ausführbarer lokaler Middleware exponiert sein.
- Blocklists für Shell-Metacharacter sind schwer korrekt zu bauen; `$()` reichte hier trotz Filterung für Codeausführung.
- Symlink-Schutz muss die finale kanonische Zielauflösung prüfen, nicht nur die erste Ebene.

## Empfehlungen

- Keine Build-Artefakte oder Service-JARs per anonymous FTP bereitstellen.
- Apache CXF aktualisieren und XOP/MTOM-LFI-Schwachstellen patchen.
- Secrets nicht in global lesbaren Service-Units oder Environment-Dateien speichern.
- Hoverfly Dashboard/API nur lokal oder stark eingeschränkt erreichbar machen.
- Shell-Aufrufe in Web-Apps vermeiden; falls nötig, Argumente ohne `shell=True` übergeben.
- Bei Log-Readern Symlinks vollständig mit `realpath()` auflösen und auf erlaubte Basispfade begrenzen.
- Sudo-Regeln so eng wie möglich fassen und Wrapper mit Dateizugriff besonders kritisch prüfen.
