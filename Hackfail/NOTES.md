# Hack The Box Fortress: Hackfail Notes

> In-Progress-Notizen, kein vollständiges Writeup. Flags, TOTP-Seeds, private Schlüssel und kurzlebige OTP-Codes sind absichtlich nicht enthalten.

## Überblick

Hackfail ist eine Fortress-/Multi-Host-Chain mit einem öffentlichen Symfony/PHP-Blog, einem internen Java/RMI-Monitoring-Service und einer weiteren VPN-/APK-/TOTP-Stufe. Der bisherige Fortschritt führte von Web-Enumeration über LFI und Symfony `_fragment`-RCE zu Codeausführung als `www-data`, danach über interne Monitoring-Artefakte zu RMI-Codeausführung als Benutzer `monitoring` auf einem zweiten Host. Zwei Flags wurden gefunden, werden hier aber nicht veröffentlicht.

Status: nicht abgeschlossen. Bisher 2 von 7 Flags gefunden.

## Scope und Ziel

```text
Fortress-Ziel: 10.13.37.13
Primärer VHost: hackfail.htb
Weiterer VHost: dev.hackfail.htb
```

Lokaler Arbeitsordner:

```text
/Users/anderson/htb/10.13.37.13
```

## Recon

Der Einstiegspunkt war HTTP auf dem Fortress-Ziel. Über Host-Header-/VHost-Enumeration wurden mindestens diese Namen relevant:

```text
hackfail.htb
 dev.hackfail.htb
```

Die Web-App lief als Symfony/PHP-Anwendung. Aus der Apache-/Filesystem-Enumeration ergaben sich diese Pfade:

```text
/var/www/html
/var/www/blog/public        # hackfail.htb
/var/www/blog_dev/public    # dev.hackfail.htb
```

## Web-Foothold: Account-Trick und LFI

Die Anwendung hatte einen Admin-/Elon-spezifischen Zugriffspfad. Der relevante Bypass basierte auf einem Username-Trailing-Space-Problem:

```text
registrierter Username: "elonmusk "
DB-Vergleich: behandelt den Wert wie "elonmusk"
PHP strict compare/session: behält den Space-Unterschied bei
```

Damit ließ sich ein Upload-/Download-Pfad erreichen. Der Download-Pfad war als LFI nutzbar. Nach Bestimmung des Upload-Prefixes konnten lokale Dateien gelesen werden, u. a.:

```text
/var/www/blog/.env
/etc/passwd
Symfony-Controller und weitere App-Quellen
```

## Symfony `_fragment` RCE

Aus der Symfony-Konfiguration wurde der `APP_SECRET` extrahiert. Damit ließ sich die Symfony-Fragment-Funktion missbrauchen, um signierte `_fragment`-Requests zu erzeugen und Kommandos auszuführen.

Proof-of-execution:

```text
uid=33(www-data)
hostname=blog
pwd=/var/www/blog/public
```

Lokaler Helper:

```text
/Users/anderson/htb/10.13.37.13/rce.py
```

Über diese RCE wurde die erste Flag-Datei im Blog-Kontext gefunden. Der Flag-Wert ist hier absichtlich entfernt.

## Backup-/Log-Leak

Über den Webpfad `backup_b64.txt` wurde ein Backup-/Log-Artefakt gefunden und lokal dekodiert:

```text
/Users/anderson/htb/10.13.37.13/backup_logs.txt
```

Das Artefakt enthielt Prozess- und Netzwerkhistorie. Wichtige Hinweise daraus:

```text
Java-Prozess: /home/elonmusk/monitoringClient.jar
Interne Verbindung zu RMI/rmiregistry und TCP 1337
Lokaler MySQL-Service
Weitere Hinweise auf FTP/VPN/APK-Artefakte
```

## Datenbank-Enumeration

Die Web-App-Konfiguration verwies auf die Datenbank `appli` und die Tabellen:

```text
bans
users
```

Aus der `users`-Tabelle wurde ein echter `elonmusk`-Account identifiziert. Der exakte Passwortwert ist in diesen öffentlichen Notizen nicht enthalten.

## Interner RMI-/Monitoring-Pivot

Aus den Logdaten ergab sich ein interner Monitoring-Host:

```text
172.22.1.250
```

Relevante Ports:

```text
1099/tcp  RMI registry
1337/tcp  Java/RMI-Service
```

Mit einem lokalen Java/RMI-Exploit wurde Codeausführung auf dem Monitoring-Host erreicht.

Ausführungskontext:

```text
uid=1000(monitoring)
hostname=watcher
home=/home/monitoring
```

Lokaler Helper:

```text
/Users/anderson/htb/10.13.37.13/mon_rce.py
```

Über diesen Pivot wurde eine zweite Flag-Datei im Monitoring-Home gefunden. Der Flag-Wert ist hier absichtlich entfernt.

## FTP/VPN/APK-Stufe

Weitere Artefakte wurden aus der internen Kette gezogen:

```text
hackfail-auth.apk
hackfail.ovpn
```

Lokale Kopien:

```text
/Users/anderson/htb/10.13.37.13/hackfail-auth.apk
/Users/anderson/htb/10.13.37.13/hackfail.ovpn
```

Die APK wurde dekompiliert:

```text
/Users/anderson/htb/10.13.37.13/apk_decomp/
```

Wichtige Klassen:

```text
com/hackfail/authenticator/MainActivity.java
com/hackfail/authenticator/TOTP.java
com/hackfail/authenticator/ChaCha20.java
```

Die App-Logik:

```text
1. Erwartet einen 32-Byte-Key.
2. Prüft den Key über ChaCha20 gegen bekannten Plain-/Ciphertext.
3. Entschlüsselt bei korrektem Key einen TOTP-Seed.
4. Generiert 8-stellige HMAC-SHA1-TOTP-Codes mit 30-Sekunden-Step.
```

Ein lokaler Solver wurde erstellt, um den Key aus Known-Plaintext abzuleiten und daraus den Seed/OTP zu berechnen. Sensible Werte sind nicht veröffentlicht.

```text
/Users/anderson/htb/10.13.37.13/solve_apk_key.py
/Users/anderson/htb/10.13.37.13/check_apk_key.py
```

Mit VPN-User und TOTP wurde anschließend eine weitere interne VPN-Stufe erreicht. Der aktuelle OTP ist kurzlebig und wird nicht dokumentiert.

## Neue VPN-/Proxy-Ebene

Nach der internen VPN-Stufe wurden weitere Hosts gescannt. Bisher relevante Funde:

```text
172.22.43.1    3128/tcp Squid HTTP proxy 4.6
172.22.43.142  Host erreichbar, keine offenen Top-1000-TCP-Ports
```

Nmap-Artefakte:

```text
/Users/anderson/htb/10.13.37.13/nmap-hackfailvpn-hosts.nmap
/Users/anderson/htb/10.13.37.13/nmap-squid-scripts.txt
```

## Bisherige Kill Chain

```text
HTTP/VHost enum
  -> Symfony/PHP Blog auf hackfail.htb
  -> Username-Trailing-Space-Bug für privilegierten Webzugriff
  -> Upload-/Download-LFI
  -> .env und App-Source lesen
  -> APP_SECRET extrahieren
  -> Symfony _fragment RCE als www-data auf blog
  -> erste Flag-Datei finden
  -> Backup-/Prozesslogs lesen
  -> internen RMI-Monitoring-Host identifizieren
  -> RMI-Codeausführung als monitoring auf watcher
  -> zweite Flag-Datei finden
  -> FTP/VPN/APK-Artefakte einsammeln
  -> APK analysieren: ChaCha20 + TOTP
  -> interne VPN-Stufe betreten
  -> Squid-Proxy auf 172.22.43.1 entdecken
```

## Offene Punkte

- Squid-Proxy auf `172.22.43.1:3128` weiter prüfen.
- Proxy eventuell für weitere interne Web-/Service-Enumeration nutzen.
- `172.22.43.142` und weitere Hosts hinter der VPN-Stufe enumerieren.
- Restliche 5 Flags finden.
- Nach Abschluss diese Notizen in ein vollständiges `README.md` umwandeln.

## Lokale Artefakte

```text
/Users/anderson/htb/10.13.37.13/rce.py
/Users/anderson/htb/10.13.37.13/mon_rce.py
/Users/anderson/htb/10.13.37.13/lfi_probe.py
/Users/anderson/htb/10.13.37.13/lfi_read.py
/Users/anderson/htb/10.13.37.13/backup_logs.txt
/Users/anderson/htb/10.13.37.13/hackfail-auth.apk
/Users/anderson/htb/10.13.37.13/hackfail.ovpn
/Users/anderson/htb/10.13.37.13/apk_decomp/
/Users/anderson/htb/10.13.37.13/solve_apk_key.py
/Users/anderson/htb/10.13.37.13/nmap-hackfailvpn-hosts.nmap
/Users/anderson/htb/10.13.37.13/nmap-squid-scripts.txt
```

## Status

Nicht abgeschlossen. Diese Datei ist eine öffentliche, redigierte Zwischenablage. Sie enthält keine Flag-Werte, keine TOTP-Seeds und keine kurzlebigen OTP-Codes.
