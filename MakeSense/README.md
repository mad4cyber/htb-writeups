# Hack The Box: MakeSense Writeup

> Public-safe Writeup: Flags und wiederverwendbare Secrets sind absichtlich nicht enthalten.

## Überblick

MakeSense ist eine Linux-Maschine aus HTB Active Season 11. Exponiert waren SSH und ein HTTPS-VHost mit einer WordPress-Seite. Der Einstieg gelang über einen Stored-XSS-Pfad in der Kontaktfunktion, der von einem Admin-Bot besucht wurde. Aus dem Admin-Kontext wurde ein temporärer WordPress-Admin erzeugt, anschließend führte ein Plugin-Upload zu Code Execution als `www-data`.

Die User-Eskalation basierte auf einem wiederverwendeten Passwort aus der WordPress-Konfiguration. Root wurde über einen internen OCR-Service erreicht, der als root auf `127.0.0.1:8001` lief und OCR-Ergebnisse mit beliebiger Dateiendung in ein web-served Verzeichnis speichern konnte.

## Recon

Der initiale Scan zeigte nur wenige relevante Dienste:

```text
22/tcp   OpenSSH 9.6p1 Ubuntu
443/tcp  Apache httpd 2.4.58 Ubuntu, HTTPS
```

Der TLS-Common-Name und die Webantwort zeigten auf den VHost:

```text
makesense.htb
```

Die HTTPS-Seite war eine WordPress-Installation mit Theme `webagency`. Relevante Hinweise im Frontend waren:

```text
/wp-content/themes/webagency/assets/js/whisper/whisper-wrapper.js
/wp-admin/admin-ajax.php
```

Im JavaScript wurden mehrere AJAX-Actions sichtbar:

```text
submit_contact_form
save_voice_raw
save_voice_results
```

Zusätzlich waren einige WordPress-Verzeichnisse browsbar, unter anderem Upload-/Include-Pfade. Das war für die Orientierung hilfreich, der eigentliche Einstieg lag aber in der Admin-Bot-/XSS-Chain.

## Foothold: Stored XSS gegen Admin-Bot

Die Kontaktfunktion speicherte eingereichte Inhalte so, dass sie später im WordPress-Admin-Kontext gerendert wurden. Ein Headless-Admin-Bot besuchte regelmäßig diese Inhalte. Auf dem Ziel war später auch der Prozess sichtbar:

```text
/home/admin/.process/prey-unified.py
```

Für den Proof wurde eine externe JavaScript-Datei über die Kontaktform eingebunden. Der Admin-Bot lud diese Datei und führte sie im eingeloggten WordPress-Admin-Kontext aus.

Der XSS-Payload machte im Prinzip:

1. `/wp-admin/user-new.php` im Admin-Kontext abrufen.
2. `_wpnonce_create-user` aus der Seite extrahieren.
3. Per `POST` einen temporären WordPress-Benutzer mit Rolle `administrator` anlegen.
4. Ergebnis an den lokalen Listener zurückmelden.

Damit stand ein WordPress-Admin-Zugang zur Verfügung, ohne WordPress-Credentials zu erraten.

## Code Execution als www-data

Mit dem temporären Admin-Zugang war Plugin-Upload verfügbar. Ein minimales WordPress-Plugin wurde hochgeladen und aktiviert. Es enthielt nur einen kleinen Command-Wrapper für Laborzwecke.

Die Ausführung bestätigte den Kontext:

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Anschließend wurde die lokale WordPress-Konfiguration gelesen. Die Installation nutzte SQLite:

```php
define( 'DB_DIR', __DIR__ . '/wp-content/database/' );
define( 'DB_FILE', '.ht.sqlite' );
```

In derselben Konfiguration war ein wiederverwendetes Passwort für den Linux-Benutzer `walter` hinterlegt. Das Passwort ist hier absichtlich nicht enthalten.

## User: SSH als walter

Mit dem wiederverwendeten Passwort war SSH als `walter` möglich:

```text
uid=1000(walter) gid=1000(walter) groups=1000(walter)
```

Damit konnte `user.txt` unter `/home/walter/user.txt` gelesen werden.

## Enumeration als walter

`sudo -l` war kein direkter Weg:

```text
Sorry, user walter may not run sudo on makesense.
```

Die Prozessliste war deutlich interessanter. Neben Apache und SSH lief ein lokaler PHP-Service als root:

```text
root  php -S 127.0.0.1:8001 -t /root/ocr4/
```

Der Service war von außen nicht erreichbar, aber lokal über SSH als `walter` ansprechbar. Ohne Authentifizierung gab er HTTP Basic Auth zurück:

```text
WWW-Authenticate: Basic realm="OCR Protected"
```

Die bekannten `walter`-Credentials funktionierten auch für diesen internen Service.

## Privilege Escalation: OCR zu PHP als root

Die interne Anwendung war eine kleine OCR-/Canvas-App:

```text
Draw text. Read it back.
```

Der Ablauf:

1. Benutzer zeichnet Text in ein Canvas.
2. Das Canvas wird als PNG an den Service geschickt.
3. Der Service erkennt den Text per OCR.
4. Das erkannte Ergebnis kann unter einem frei wählbaren Dateinamen gespeichert werden.

Nach einem Test wurde klar, dass die Anwendung die Ausgabe unter einem web-served `saved/`-Pfad speichert und Dateiendungen wie `.php` nicht blockiert.

Der entscheidende Primitive war daher:

```text
Bild mit PHP-Code erzeugen
  -> OCR erkennt PHP-Code als Text
  -> Ergebnis als saved/shell.php speichern
  -> Datei wird vom root-PHP-Server ausgeführt
```

Ein gut lesbares Bild mit folgendem Text wurde erkannt:

```php
<?php system($_GET[1]);?>
```

Nach dem Speichern als `shell.php` wurde der Befehlskontext bestätigt:

```text
uid=0(root) gid=0(root) groups=0(root)
```

Damit konnte `root.txt` über den lokalen OCR-Service gelesen werden.

## Kill Chain

1. `makesense.htb` als HTTPS-VHost identifiziert.
2. WordPress mit Theme `webagency` erkannt.
3. Kontaktformular-/Admin-Bot-Flow untersucht.
4. Stored XSS im Admin-Kontext ausgelöst.
5. Aus dem XSS heraus temporären WordPress-Admin erstellt.
6. Plugin-Upload genutzt, um Code Execution als `www-data` zu erhalten.
7. WordPress-Konfiguration gelesen; wiederverwendetes `walter`-Passwort gefunden.
8. SSH als `walter` und User-Flag gelesen.
9. Internen root-PHP-Service auf `127.0.0.1:8001` gefunden.
10. OCR-App missbraucht, um PHP-Code als `saved/shell.php` zu speichern.
11. PHP-Datei lief als root und erlaubte das Lesen der Root-Flag.
12. Temporäre Artefakte entfernt.

## Cleanup

Während der XSS-Phase hat der Admin-Bot den Payload mehrfach ausgeführt. Dadurch entstanden mehrere temporäre `svc*`-WordPress-Admin-Benutzer. Nach Root wurden alle temporären Artefakte entfernt und verifiziert:

```text
svc_users=[]
cleanup_plugin_dir=removed
ocr_shell=404
```

Entfernt wurden:

- temporäre `svc*`-WordPress-Admin-Benutzer
- temporäres WordPress-Plugin für Command Execution
- temporäres Cleanup-Plugin
- OCR-Root-Shell unter `/root/ocr4/saved/`
- lokaler XSS-Listener

## Takeaways

- Stored XSS wird deutlich kritischer, wenn ein privilegierter Bot regelmäßig Admin-Ansichten besucht.
- WordPress-Admin-Zugang ist fast immer gleichbedeutend mit Code Execution, wenn Plugin-Upload aktiv ist.
- Secrets in Konfigurationsdateien sollten nie für lokale Benutzer wiederverwendet werden.
- Interne Services auf `127.0.0.1` sind nach User-Foothold oft der wichtigste Privesc-Hinweis.
- OCR-/Konvertierungs-Workflows sind gefährlich, wenn erkannte Inhalte ungefiltert als ausführbare Dateien gespeichert werden können.

## Empfehlungen

- Kontaktformular-Ausgaben strikt serverseitig encoden und im Admin-UI nur als Text rendern.
- Admin-Bots mit minimalen Rechten und isolierten Browserprofilen betreiben.
- WordPress-Plugin-Upload für nicht benötigte Rollen deaktivieren oder stark einschränken.
- Keine Wiederverwendung von Web-/DB-/Linux-Passwörtern.
- Interne root-Services nicht über dynamische Webserver ausführen.
- Upload-/Save-Funktionen auf erlaubte Dateiendungen und nicht-ausführbare Speicherorte beschränken.
