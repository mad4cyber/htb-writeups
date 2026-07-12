# Hack The Box: Paperwork — Writeup

> Flags und Zugangsdaten sind absichtlich nicht enthalten.

## Überblick

**Paperwork** ist eine Linux Release-Arena-Maschine mit einem Archivierungs-/Drucksystem. Die vollständige Kette verbindet einen unsicheren LPD-Dienst, einen localhost-exponierten PJL/JetDirect-Emulator und einen rootlaufenden Unix-Socket-Dienst.

## Recon

| Port | Dienst | Hinweis |
|---:|---|---|
| 22 | OpenSSH | Nach dem Pivot als `archivist` nutzbar |
| 80 | nginx | Intranet-Portal; stellt ein Archiv mit LPD-Quellcode bereit |
| 1515 | Custom LPD | Druckjob-Annahme und verwundbare Verarbeitung des Control-Files |

Das Portal verweist auf einen RFC-1179-kompatiblen Legacy-Intake und erlaubt den Download des dazugehörigen Python-Quellcodes.

## Foothold: LPD → `lp`

Der LPD-Server übernimmt den Namen eines Druckjobs aus einer `J`-Zeile des Control-Files und interpoliert ihn über `subprocess.Popen(..., shell=True)` in einen Shell-Befehl. Das ergibt Command Injection im Kontext von `lp`.

## Pivot: PJL/JetDirect → `archivist`

Ein nur lokal erreichbarer JetDirect-Service auf Port 9100 läuft als `archivist` und implementiert PJL-Dateisystembefehle. Seine Pfadübersetzung verwendet effektiv `normpath(join(root, supplied_path))`, prüft aber nicht, ob das Resultat noch innerhalb des vorgesehenen Roots liegt.

Relative Traversals erlauben damit Datei-Lese- und Schreibzugriffe über PJL. Ein Upload in die SSH-Autorisierungsdatei des Benutzers ermöglicht den stabilen Pivot nach `archivist`.

## Privilege Escalation: Root-Daemon / Unix Socket

`/usr/bin/paperwork-daemon` läuft als root und stellt `/run/paperwork/mgmt.sock` bereit. Der Socket gehört zur Gruppe von `archivist`. Erkennt der Daemon PJL-Dateisystem-Kommandos in `commands.log`, sendet er mit `SCM_RIGHTS` privilegierte File Descriptors an den Client.

Der FD-Leak liefert die für den lokalen, authentifizierten Root-Übergang erforderliche Berechtigung. Der Root-Kontext und der Zugriff auf die Root-Flag wurden anschließend validiert.

## Kill Chain

1. LPD-Control-File-Injection als `lp`.
2. Zugriff auf localhost:9100 und PJL-Pfadtraversal.
3. Datei-Schreibprimitive für den Pivot nach `archivist`.
4. PJL-Logeintrag triggern und den root-Daemon-FD-Leak über den Unix Socket empfangen.
5. Lokalen Root-Übergang mit der resultierenden Berechtigung validieren.

## Takeaways

- Control-Dateien und Metadaten müssen wie untrusted input behandelt werden; Shell-Interpolation ist bei Print-Workflows besonders gefährlich.
- `normpath()` ersetzt keine Pfad-Grenzprüfung. Nach der Normalisierung muss geprüft werden, dass der Pfad innerhalb des erlaubten Root-Verzeichnisses liegt.
- `SCM_RIGHTS` über gruppenbeschreibbare Unix Sockets kann vertrauliche root-Dateien an weniger privilegierte Clients weitergeben.
- Ein Symlink-Race gegen die geloggte Datei wäre unnötig riskant, weil der Daemon sie später als root trunciert.
