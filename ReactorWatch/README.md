# Hack The Box: ReactorWatch Writeup

Flags sind absichtlich nicht enthalten.

## Überblick

ReactorWatch war eine Linux-Maschine aus der EU Release Arena. Der Einstieg lief über eine verwundbare Next.js/React-Anwendung auf Port 3000. Nach RCE wurden lokale App-Artefakte und eine SQLite-Datenbank ausgelesen. Ein geknackter Benutzer-Hash ermöglichte SSH als `engineer`. Die Privilege Escalation erfolgte anschließend über einen als root laufenden Node.js-Service mit lokal aktiviertem Inspector-Port.

## Recon

Der initiale Scan zeigte nur wenige offene Dienste:

```text
22/tcp    ssh
3000/tcp  http / Next.js
```

Die Web-App zeigte sich als ReactorWatch / Core Monitoring System. Klassische Web-Enumeration brachte kaum verwertbare Endpunkte. Interessanter waren die statischen Next.js-Artefakte und die verwendeten Framework-Versionen.

Wichtige Hinweise:

```text
Next.js 15.0.3
React 19.0.0
React DOM 19.0.0
```

Diese Kombination war anfällig für React Server Components / React2Shell.

## Foothold

Die Anwendung auf Port 3000 war gegen CVE-2025-55182, auch bekannt als React2Shell, verwundbar. Ein kontrollierter Command-Execution-Test bestätigte RCE im Kontext des App-Benutzers:

```text
uid=999(node) gid=988(node)
pwd: /opt/reactor-app
hostname: reactor
```

Nach der RCE wurden die App-Dateien in `/opt/reactor-app` geprüft. Besonders relevant waren:

```text
/opt/reactor-app/package.json
/opt/reactor-app/.env
/opt/reactor-app/reactor.db
```

Die `.env` zeigte eine SQLite-Konfiguration:

```text
DB_PATH=/opt/reactor-app/reactor.db
DB_TYPE=sqlite3
NODE_ENV=production
```

## Enumeration

Die SQLite-Datenbank enthielt unter anderem eine `users`-Tabelle:

```sql
CREATE TABLE users (
    id INTEGER PRIMARY KEY,
    username TEXT NOT NULL,
    password_hash TEXT NOT NULL,
    role TEXT NOT NULL,
    email TEXT
);
```

Die Einträge enthielten einen Administrator-Account und den Benutzer `engineer`. Die Hashes sahen nach MD5 aus. Der `engineer`-Hash ließ sich mit einer Standard-Wortliste knacken. Mit dem gefundenen Passwort war SSH als `engineer` möglich.

Der Benutzer gehörte zu mehreren Gruppen, unter anderem `lxd`:

```text
engineer adm cdrom dip plugdev lxd
```

LXD war jedoch auf der Maschine nicht brauchbar, da der Snap/LXD-Installer fehlschlug. Deshalb wurde die Enumeration auf lokale Services und Custom-Systemd-Units fokussiert.

## Privilege Escalation

Unter `/etc/systemd/system` fiel ein eigener Service auf:

```ini
[Service]
Type=simple
User=root
ExecStart=/usr/bin/node --inspect=127.0.0.1:9229 /opt/uptime-monitor/worker.js
Restart=on-failure
RestartSec=3
```

Der Service `uptime-monitor.service` lief als root und startete Node.js mit aktiviertem Inspector auf `127.0.0.1:9229`. Von außen war dieser Port nicht erreichbar, aber über SSH konnte ein lokaler Portforward gesetzt werden.

Nach dem Forward war der Node Inspector über die DevTools-/Inspector-API erreichbar:

```text
/json/list -> webSocketDebuggerUrl
```

Über das Chrome DevTools Protocol konnte `Runtime.evaluate` genutzt werden, um JavaScript im root-Node-Prozess auszuführen. Daraus ergab sich root-Code-Execution via Node.js `child_process.execSync()`.

Ein `id`-Proof bestätigte die Eskalation:

```text
uid=0(root) gid=0(root) groups=0(root)
```

Danach konnte die Root-Flag gelesen werden.

## Kill Chain

```text
nmap -> Port 3000 Next.js
Next.js/React-Versionen prüfen
React2Shell RCE als node
/opt/reactor-app/.env und reactor.db lesen
MD5-Hash aus users-Tabelle cracken
SSH als engineer
Custom systemd service finden
root-Node-Prozess mit --inspect auf 127.0.0.1:9229
SSH-Portforward zum Inspector
Chrome DevTools Protocol Runtime.evaluate
root command execution
```

## Takeaways

- Bei sehr sparsamen Next.js-Apps lohnt sich Framework-/Version-Enumeration mehr als blindes Directory-Fuzzing.
- Nach Web-RCE sofort App-Verzeichnis, `.env`, lokale Datenbanken und systemd-Konfigurationen prüfen.
- SQLite-Dateien in `/opt` oder App-Verzeichnissen sind oft direkte Pivot-Punkte zu echten Systembenutzern.
- Die Gruppe `lxd` ist ein guter Lead, aber nicht immer der intended path.
- Lokale Debug-Interfaces sind auf Linux-Maschinen sehr wertvoll:
  - Node `--inspect`
  - JVM/JMX
  - gdbserver
  - Flask/Werkzeug Debugger
  - lokale Admin-APIs
- Ein root-Prozess mit Node Inspector ist praktisch root-RCE, wenn ein lokaler Benutzer per SSH tunneln kann.

## Empfehlungen

- React/Next.js-Versionen aktualisieren und CVE-2025-55182 patchen.
- Secrets und Benutzer-Hashes nicht in lokal lesbaren App-Datenbanken speichern.
- Passwörter nicht als schnelle MD5-Hashes speichern; stattdessen Argon2id/bcrypt/scrypt mit Salt verwenden.
- Debug-Ports wie Node Inspector niemals in produktiven systemd-Services aktivieren.
- Falls Debugging nötig ist, nur temporär und mit zusätzlicher Zugriffskontrolle.
- Services mit minimalen Rechten ausführen; Monitoring-Worker benötigen in der Regel keinen root-Kontext.
