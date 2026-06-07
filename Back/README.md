# Hack The Box: Back Writeup

> Public-safe Writeup: Flags und wiederverwendbare Secrets sind absichtlich nicht enthalten.

## Überblick

Back ist eine Linux/Sangoma-Box aus HTB Active Season 11. Die exponierte Weboberfläche ist FreePBX auf einem CentOS/RHEL-ähnlichen System. Der Einstieg gelingt über eine pre-auth FreePBX-Schwachstelle, die zu SQL-Injection und anschließend zu Code Execution als `asterisk` führt. Die Privilege Escalation basiert auf einer unsicheren Root-Automation im FreePBX/Sangoma-Stack.

## Recon

Der initiale Scan zeigte nur wenige offene Ports:

```text
22/tcp   OpenSSH 7.4
80/tcp   Apache httpd 2.4.6 / PHP 7.4.16
443/tcp  Apache httpd 2.4.6 / PHP 7.4.16
```

HTTP leitete auf den Vhost `connected.htb` um. Unter `/admin/config.php` war die FreePBX-Administration erreichbar:

```text
FreePBX 16.0.40.7
Apache 2.4.6
PHP 7.4.16
OS: Sangoma Linux 7
```

Die Version war wichtig, weil FreePBX 16.0.40.7 in den verwundbaren Bereich für CVE-2025-57819 fällt.

## Foothold: FreePBX CVE-2025-57819

FreePBX-Versionen vor 16.0.89 sind anfällig für eine Kette aus Auth-Bypass und SQL-Injection im Endpoint-Ajax-Codepfad. Der relevante unauthentifizierte Einstiegspunkt war:

```text
/admin/ajax.php?module=FreePBX\modules\endpoint\ajax&command=model&template=x&model=model&brand=...
```

Über den `brand`-Parameter ließ sich SQL einschleusen. Auch wenn die Anwendung mit einem Fehler wie diesem antwortete, wurde die SQL-Anweisung trotzdem ausgeführt:

```text
Trying to access array offset on value of type bool
/var/www/html/admin/modules/endpoint/views/model.php
```

Für einen kontrollierten Proof of Code Execution wurde ein Eintrag in `cron_jobs` geschrieben, der einen kleinen PHP Command Wrapper in den Webroot legte. Wichtig war dabei, Base64-Payloads sauber zu URL-encoden, insbesondere `+` als `%2b`, damit die PHP-Datei nicht korrupt geschrieben wird.

Nach kurzer Wartezeit führte der Webshell-Pfad Befehle als `asterisk` aus:

```text
uid=999(asterisk) gid=1000(asterisk) groups=1000(asterisk)
```

Damit war der User-Kontext erreicht und `user.txt` konnte unter `/home/asterisk/user.txt` gelesen werden.

## Enumeration als asterisk

Die lokale Enumeration zeigte unter anderem:

```bash
sudo -l
getcap -r / 2>/dev/null
find / -perm -4000 -type f 2>/dev/null
ps auxww
```

Auffällig waren FreePBX/Sangoma-spezifische Automationen und ein SUID-Bash-Zustand auf dem System. Der eigentliche interessante Privesc-Pfad lag in einer root-ausgeführten FreePBX-Komponente.

## Privilege Escalation: sysadmin_manager Pipe Injection

Die entscheidende Komponente war:

```text
/usr/bin/sysadmin_manager
```

Dieser PHP-basierte Manager führt FreePBX-Modul-Hooks als root aus. Die relevanten Punkte:

- `sysadmin_manager` prüft FreePBX-Modulsignaturen.
- Die ionCube-verschlüsselte Logik nutzt ein hart kodiertes GPG-Homedir unter `/home/asterisk/.gnupg`.
- Dateien mit der Endung `.CONTENTS` werden speziell behandelt: Parameter werden aus dem Dateiinhalt gelesen.
- Die Parameter werden zwar gefiltert, aber das Pipe-Zeichen `|` wurde nicht blockiert.
- Anschließend wird ein Shell-Kommando ähnlich zu `$hookfile $params` ausgeführt.

Damit konnte ein Pipe-Zeichen im Parameterkontext Shell-Semantik erzeugen.

Zusätzlich gab es incron-Regeln, die bestimmte Pfade überwachten. Ein Pfad unter `/var/spool/asterisk/incron` war racy, weil `IN_MODIFY` beim Truncation-Event zu früh auslöste. Der zuverlässigere Pfad war der lokale `IN_CLOSE_WRITE`-Flow:

```text
/usr/local/asterisk/incron/
```

Der ausgenutzte Primitive war sinngemäß:

```bash
printf '| chmod 4755 /bin/bash' > /usr/local/asterisk/incron/sysadmin.dump-iptables.CONTENTS
sleep 3
/bin/bash -p -c 'id'
```

Nachdem `/bin/bash` SUID-root war, reichte:

```bash
/bin/bash -p -c 'id; cat /root/root.txt'
```

Der Befehl lief mit effektiver UID 0:

```text
euid=0(root)
```

Damit war Root erreicht.

## Kill Chain

1. Vhost `connected.htb` identifiziert.
2. FreePBX 16.0.40.7 erkannt.
3. CVE-2025-57819 gegen `/admin/ajax.php` genutzt.
4. Per SQL-Injection einen Cronjob für einen PHP Command Wrapper angelegt.
5. Code Execution als `asterisk` bestätigt.
6. User-Flag unter `/home/asterisk/user.txt` gelesen.
7. FreePBX/Sangoma-Root-Automation untersucht.
8. `sysadmin_manager`/incron Pipe Injection genutzt, um `/bin/bash` SUID-root zu setzen.
9. Root-Flag über `/bin/bash -p` gelesen.
10. Temporäre Artefakte entfernt.

## Cleanup

Nach der Verifikation wurden die temporären Webshells entfernt und die per SQLi erzeugten Cronjob-Einträge gelöscht. Danach wurden die Webshell-Pfade erneut abgefragt und lieferten `404 Not Found`.

## Takeaways

- Versionshinweise in Login-Oberflächen sind bei Appliance-Stacks oft direkt ausnutzbar.
- Fehlerantworten schließen erfolgreiche SQL-Ausführung nicht aus.
- Bei Cronjob-/Webshell-Payloads können URL-Encoding-Details wie `+` vs. `%2b` entscheidend sein.
- ionCube-Code ist nicht automatisch eine Blackbox: Prozessverhalten, Pfade, GPG-Aufrufe und Hook-Mechaniken lassen sich trotzdem über Dateien, Strings und Runtime-Verhalten nachvollziehen.
- File-Watcher unterscheiden sich stark: `IN_MODIFY` kann zu früh triggern, `IN_CLOSE_WRITE` ist oft zuverlässiger.
- Sanitizer-Blocklists sind fragil; ein einzelnes fehlendes Shell-Metazeichen wie `|` kann zur Root-Ausführung reichen.

## Empfehlungen

- FreePBX auf eine Version aktualisieren, die CVE-2025-57819 behebt.
- Administrative PBX-Oberflächen nicht direkt aus dem VPN-/Internet-Scope exponieren.
- Root-Automationen sollten keine Shell-Kommandos aus zusammengesetzten Strings ausführen.
- Parameter sollten allowlist-basiert validiert und nicht über Shell-Metazeichen-Blocklists abgesichert werden.
- File-Watcher, die privilegierte Aktionen auslösen, sollten nicht auf world-writable Pfade reagieren.
