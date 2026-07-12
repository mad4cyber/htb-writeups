# Hack The Box: Paperwork — Notes

> Status: unvollständig. Flags und Zugangsdaten sind absichtlich nicht enthalten.

## Überblick

**Paperwork** ist eine Linux Release-Arena-Maschine mit einer Weboberfläche für ein Archivierungs-/Drucksystem. Die Kette verbindet einen unsicheren LPD-Dienst, einen localhost-exponierten PJL/JetDirect-Emulator und einen rootlaufenden Unix-Socket-Dienst.

## Recon

| Port | Dienst | Hinweis |
|---:|---|---|
| 22 | OpenSSH | Nach dem Pivot als `archivist` nutzbar |
| 80 | nginx | Intranet-Portal; stellt ein Archiv mit LPD-Quellcode bereit |
| 1515 | Custom LPD | Druckjob-Annahme und verwundbare Verarbeitung des Control-Files |

Das Portal verweist auf einen RFC-1179-kompatiblen Legacy-Intake und erlaubt den Download des dazugehörigen Python-Quellcodes.

## Bisheriger Fortschritt

### LPD → `lp`

Im Quellcode des LPD-Servers wird der Name eines Druckjobs aus einer `J`-Zeile des Control-Files übernommen und über `subprocess.Popen(..., shell=True)` in einen Shell-Befehl interpoliert. Damit entsteht Command Injection im Kontext von `lp`.

### PJL/JetDirect → `archivist`

Der nur lokal erreichbare JetDirect-Service auf Port 9100 läuft als `archivist` und implementiert PJL-Dateisystembefehle. Seine Pfadübersetzung verwendet effektiv `normpath(join(root, supplied_path))`, prüft danach aber nicht, ob der resultierende Pfad noch innerhalb des vorgesehenen Roots liegt.

Dadurch sind relative Traversals möglich. Als praktischer Pivot kann ein PJL-Datei-Upload eine SSH-Autorisierungsdatei im Home-Verzeichnis von `archivist` schreiben; anschließend ist SSH als dieser Benutzer möglich.

### Root-Daemon / Unix Socket

`/usr/bin/paperwork-daemon` läuft als root und stellt `/run/paperwork/mgmt.sock` bereit. Der Socket ist für die Gruppe von `archivist` beschreibbar. Erkennt der Daemon PJL-Dateisystem-Kommandos in `commands.log`, sendet er mit `SCM_RIGHTS` File Descriptors an den Client. Das demonstriert einen privilegierten FD-Leak aus dem root-Kontext.

## Blocker

Der finale Root-Schritt ist noch nicht dokumentiert. Besonders riskant wäre ein Symlink-Race auf die geloggte Datei: Der Daemon öffnet und trunciert den Pfad später als root. Diese Variante wurde deshalb nicht eingesetzt, um keine Flag- oder Systemdatei zu beschädigen.

## Wahrscheinliche Chain

1. LPD-Control-File-Injection als `lp`.
2. Zugriff auf localhost:9100 und PJL-Pfadtraversal.
3. Pivot nach `archivist`.
4. Trigger des root-Daemons über PJL-Logeinträge und Analyse des FD-Leaks.
5. Nicht-destruktiven finalen root-Pivot identifizieren.

## Lokale Artefakte

- Session-Zusammenfassung: `~/htb/10.129.46.91/session-summary.md`
- LPD-Client: `~/htb/10.129.46.91/lpd_submit.py`
- FD-Empfangs-Helfer: `~/htb/10.129.46.91/trigger_and_recv.py`

## Status

In Bearbeitung; nur veröffentlichungssichere technische Notizen.
