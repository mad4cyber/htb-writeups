# Hack The Box: TwoMillion Writeup

Flags sind absichtlich nicht enthalten.

## Überblick

TwoMillion ist eine Easy-Linux-Maschine mit einer nostalgischen Hack-The-Box-Weboberfläche. Die Chain führt von Invite-Code-API-Analyse über eine fehlerhafte Admin-Settings-API zu Command Injection, anschließend über Passwort-Wiederverwendung zu SSH und abschließend über eine OverlayFS/FUSE-Kernel-LPE zu root.

## Recon

Nmap zeigte nur zwei offene Ports:

```text
22/tcp open  ssh   OpenSSH 8.9p1 Ubuntu
80/tcp open  http  nginx -> http://2million.htb/
```

Der HTTP-Service redirectete auf den VHost:

```text
2million.htb
```

## Invite-Code Flow

Die Seite `/invite` lud ein JavaScript:

```text
/js/inviteapi.min.js
```

Darin waren die relevanten API-Endpunkte versteckt:

```text
/api/v1/invite/how/to/generate
/api/v1/invite/generate
/api/v1/invite/verify
```

`/api/v1/invite/how/to/generate` gab ROT13-kodierte Hinweise zurück. Danach lieferte ein POST auf `/api/v1/invite/generate` einen base64-kodierten Invite-Code. Mit diesem Code ließ sich ein normaler Account registrieren.

## API Enumeration

Nach Login war `/api/v1` erreichbar und gab eine vollständige Routenliste zurück. Besonders interessant waren die Admin-Endpunkte:

```text
GET  /api/v1/admin/auth
POST /api/v1/admin/vpn/generate
PUT  /api/v1/admin/settings/update
```

## Admin Bypass

Der Settings-Endpunkt akzeptierte JSON. Als eingeloggter Benutzer konnte ich den eigenen Account per E-Mail auf Admin setzen:

```json
{
  "email": "<eigene-email>",
  "is_admin": 1
}
```

Danach bestätigte `/api/v1/admin/auth`, dass der Account Adminrechte hatte.

## Foothold: Command Injection

Der Admin-Endpunkt `/api/v1/admin/vpn/generate` generiert VPN-Konfigurationen für angegebene User. Der JSON-Parameter `username` wurde unsicher in einen Shell-Befehl eingebaut.

Ein Payload nach diesem Muster führte Kommandos als `www-data` aus:

```json
{
  "username": "x;id;"
}
```

Damit war Remote Code Execution auf dem Webserver möglich.

## Pivot zu SSH

Über die RCE konnte die Web-App-Konfiguration gelesen werden:

```text
/var/www/html/.env
```

Dort lagen Datenbank-Zugangsdaten. Das Datenbankpasswort wurde auch für den Linux-User `admin` verwendet. Damit war SSH-Zugriff als `admin` möglich.

Nach dem Login war die User-Flag im Home-Verzeichnis des Users erreichbar.

## Privilege Escalation

Im Postfach des Users lag ein Hinweis:

```text
/var/mail/admin
```

Die Mail erwähnte eine kritische OverlayFS/FUSE-Kernel-Schwachstelle. Das System lief mit einem verwundbaren Ubuntu-Kernel:

```text
Linux 5.15.70-051570-generic
```

Ein CVE-2023-0386 OverlayFS/FUSE PoC wurde auf dem Ziel kompiliert und ausgeführt. Dadurch entstand ein root-owned SUID Helper, der eine Root-Shell ermöglichte. Damit konnte die Root-Flag gelesen werden.

Nach der Verifikation wurden temporäre PoC-Dateien und der FUSE-Prozess entfernt.

## Kill Chain

```text
HTTP vhost 2million.htb
-> /invite JavaScript analysieren
-> Invite-Code generieren
-> Account registrieren und einloggen
-> /api/v1 route list lesen
-> eigenen Account via JSON is_admin=1 setzen
-> Admin VPN generator command injection
-> RCE als www-data
-> /var/www/html/.env lesen
-> DB-Passwortreuse für SSH admin
-> user.txt
-> /var/mail/admin Kernel-CVE-Hinweis
-> CVE-2023-0386 OverlayFS/FUSE LPE
-> root.txt
```

## Takeaways

- API-Root-Endpunkte wie `/api/v1` immer prüfen; sie können versteckte Routenlisten offenlegen.
- Bei API-Tests Content-Type wechseln: hier war `application/json` entscheidend.
- Self-service Admin-Settings-Endpunkte sind gute Kandidaten für Access-Control-Bugs.
- VPN-/Certificate-Generatoren sind häufig gefährlich, wenn Usernamen in Shell-Befehle eingebaut werden.
- `.env`-Dateien bleiben ein klassischer Pivot-Punkt zu SSH durch Passwort-Wiederverwendung.
- Lokale Mailboxen können gezielt auf den intended Privesc-Pfad hinweisen.
