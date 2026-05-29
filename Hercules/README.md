# Hack The Box: Hercules Writeup

Flags sind absichtlich nicht enthalten. Sensitive Werte wie konkrete NT-Hashes, Passwörter, machineKeys und Token sind redacted.

## Überblick

Hercules ist eine **Insane** Windows-Maschine (Server 2019, Active-Directory-DC) mit einer entscheidenden Rahmenbedingung: **NTLM ist domänenweit deaktiviert** — jede Authentifizierung läuft ausschließlich über Kerberos. Pass-the-Hash im klassischen Sinn ist damit unmöglich; ein NT-Hash muss erst über `getTGT` in ein Kerberos-TGT umgewandelt werden (RC4-über-Kerberos ist erlaubt).

Die offizielle Kette ist lang und kombiniert: Blind-LDAP-Injection (WAF-Bypass), `machineKey`-Diebstahl per Path-Traversal, FormsAuth-Cookie-Forgery, NTLM-Coercion über einen DOCX-Upload, mehrere Shadow-Credential-Pivots, einen AD-Object-Move (LDAP ModifyDN), eine SDProp/adminCount-Umgehung über einen SYSTEM-Task, ADCS **ESC3** und schließlich **U2U-RBCD** mit Session-Key-Hash-Sync zur DCSync.

## Recon

```text
53/tcp    DNS         Simple DNS Plus
80/443    IIS 10.0    ASP.NET MVC SSO-Portal (hercules.htb)
88        Kerberos    Primär-Auth (NTLM disabled)
389/636   LDAP/S      AD-Directory; SAN: dc.hercules.htb, hercules.htb, HERCULES
445       SMB         signing required
3268/3269 GC          Global Catalog
464       kpasswd
593       RPC over HTTP
5986      WinRM (SSL)  Shell-Eintritt (HTTPS-only)
9389      ADWS
```

Zertifikate und LDAP rootDSE bestätigen Domain `hercules.htb`, DC `dc.hercules.htb`, defaultNamingContext `DC=hercules,DC=htb`. Anonymes LDAP rootDSE funktioniert, eine echte Suche erfordert aber einen Bind. SMB-null und NTLM-Auth scheitern (NTLM aus).

## Foothold (Intended Path)

**LDAP-Injection mit WAF-Bypass.** Das SSO-Login `/Login` gibt den Usernamen ungefiltert in eine LDAP-Query. Ein WAF blockt LDAP-Metazeichen, scheitert aber an **Double-URL-Encoding** (`*` → `%252A`, `(` → `%2528`, `)` → `%2529`). Über einen blinden, zeichenweisen Oracle (Response-Größe) lässt sich das `description`-Attribut von `johnathan.j` auslesen — dort liegt ein Klartext-Passwort, das für einen weiteren Account in der Forest-Migration-OU gilt.

**machineKey via Path-Traversal.** `/Home/Download` nimmt einen `fileName`-Parameter und gibt Dateien zurück. Forward-Slashes sind geblockt, **Backslashes** funktionieren als Windows-Pfadtrenner, und `.config` ist erlaubt → `web.config` lesbar. Darin der ASP.NET `machineKey` (AES decryptionKey + HMAC-SHA256 validationKey, hier redacted).

**Cookie-Forgery + NTLM-Coercion.** Mit dem `machineKey` lässt sich ein `.ASPXAUTH`-Cookie für jeden User/jede Rolle fälschen. Die Rolle „web administrators" schaltet einen Datei-Upload auf der Report-Seite frei. Ein präpariertes DOCX (mit eingebettetem UNC-Pfad, z. B. via `ntlm_theft`) zwingt den serverseitigen Dokumentprozessor zu einer ausgehenden SMB-Verbindung gegen `Responder` → NetNTLMv2 von `natalie.a` → mit hashcat (`-m 5600`) gecrackt.

## Privilege Escalation (Intended Path)

1. **Shadow Credentials → bob.w.** `natalie.a` (Web Support) hat implizite Schreibrechte auf User der Web-Department-OU. Über `msDS-KeyCredentialLink` (PKINIT) → NT-Hash von `bob.w` (Recruitment Managers).
2. **AD Object Move (ModifyDN) → auditor.** `bob.w` hat CreateChild/DeleteChild auf Security- und Web-Department-OU. Per LDAP-ModifyDN wird `auditor` von Security → Web Department verschoben — das ändert die greifenden geerbten ACEs.
3. **Shadow Credentials → auditor → User-Flag.** Jetzt greifen natalie.a's Schreibrechte auf `auditor`; Shadow Creds → PKINIT → NT-Hash. `auditor` ist in Remote Management Users → WinRM über SSL (5986) → **User-Flag**.
4. **GenericAll auf Forest Migration OU + SDProp-Bypass.** `auditor` setzt GenericAll auf die Forest-Migration-OU für sich und für `IT Support`. Ein als SYSTEM laufender Task „Password Cleanup" cleart daraufhin `adminCount` auf `IIS_Administrator` (Mitglied der geschützten Gruppe Service Operators) und reaktiviert Vererbung.
5. **ESC3.** `fernando.r` (Smartcard Operators) wird aktiviert und mit Passwort versehen, fordert ein **Enrollment-Agent-Zertifikat** an und stellt damit ein User-Cert **on-behalf-of** `ashley.b` aus (CA `CA-HERCULES`). PKINIT → Zugriff als `ashley.b`, der den SYSTEM-Task triggert.
6. **U2U-RBCD → DCSync.** `IIS_Administrator` (Service Operators) setzt das Passwort von `iis_webserver$`. Da dieses Konto ein User-Objekt **ohne SPN** ist, scheitert klassisches S4U2Self. Trick: **Session-Key-Hash-Sync** — der NT-Hash wird gleich dem TGT-Session-Key gesetzt, wodurch `getST -u2u -force-forwardable` ein forwardable Ticket für `cifs/dc.hercules.htb` erzeugt (impersonate Administrator). Anschließend **DCSync** → Administrator.

## Kill Chain

```text
LDAP-Injection (Double-URL-Encode WAF-Bypass) -> Passwort aus description-Feld
  -> Path-Traversal /Home/Download -> web.config machineKey
  -> ASPXAUTH-Cookie "web administrators" -> DOCX-Upload -> NTLM-Coercion -> natalie.a
  -> Shadow Creds -> bob.w -> LDAP ModifyDN (auditor: Security -> Web Dept)
  -> Shadow Creds -> auditor -> USER FLAG (WinRM SSL)
  -> GenericAll Forest Migration OU -> SDProp-Bypass (SYSTEM "Password Cleanup")
  -> ESC3 (fernando.r EnrollmentAgent -> ashley.b on-behalf-of)
  -> IIS_Administrator -> reset iis_webserver$
  -> Session-Key-Hash-Sync + U2U S4U2Self/Proxy -> cifs/dc.hercules.htb
  -> DCSync -> Administrator -> ROOT FLAG
```

## Hinweis zum praktischen Vorgehen

HTB-Account-Secrets sind pro Release statisch — nur die Flags rotieren. In der Praxis lohnt es sich, die höchstprivilegierten *hash-/zertifikatbasierten* Credentials zuerst live zu testen: Während passwort-basierte Konten (z. B. die Einstiegs-Accounts) nach einem Reset oft ungültig sind, bleiben hash-derivierte Konten häufig gültig. Da NTLM zwar aus, **RC4-über-Kerberos aber erlaubt** ist, lässt sich ein gültiger NT-Hash via `getTGT` direkt in ein TGT umwandeln und für SMB/WinRM nutzen — was die komplette ESC3/SDProp/RBCD/DCSync-Kette abkürzen kann.

## Takeaways

- **NTLM-disabled ≠ Hash wertlos.** Solange RC4-Kerberos zugelassen ist, wird ein NT-Hash über `getTGT` zum TGT — Pass-the-Hash wird zu „Pass-the-Key".
- **machineKey = vollständige Cookie-Forgery.** Jede Datei-Lese-Lücke, die `web.config` erreicht, hebelt die ASP.NET FormsAuthentication komplett aus (beliebiger User, beliebige Rolle).
- **Double-URL-Encoding** umgeht WAFs, die Eingaben nur einmal dekodieren, bevor das Backend ein zweites Mal dekodiert.
- **LDAP ModifyDN ändert den ACL-Scope.** Ein Objekt zwischen OUs zu verschieben, ändert die greifenden geerbten ACEs — OU-ACLs gehören regelmäßig auditiert, nicht nur Objekt-ACLs.
- **SDProp/adminCount-Bypass** über einen SYSTEM-Task, der Sicherheitsbeschreibungen anhand von Gruppenmitgliedschaft verändert.
- **ESC3** erlaubt mit Enroll-Rechten auf einem Enrollment-Agent-Template das Ausstellen von Zertifikaten on-behalf-of beliebiger User.
- **U2U-RBCD** ermöglicht RBCD-Missbrauch auch für Accounts ohne SPN, wenn NT-Hash = TGT-Session-Key gesetzt wird (`-u2u -force-forwardable`).

## Empfehlungen

- NTLM-Deaktivierung beibehalten, aber zusätzlich RC4-Etypes für Kerberos abschalten (AES-only), damit gestohlene NT-Hashes nicht zu TGTs werden.
- `web.config`/`machineKey` dateisystemseitig schützen und rotieren; Path-Traversal-Filter müssen Backslashes und alternative Encodings normalisieren.
- WAF-Eingaben vor der Inspektion vollständig normalisieren (rekursives Decoding).
- OU-Level-ACEs auditieren (CreateChild/DeleteChild, WriteOwner/WriteDACL), Shadow-Credential-Schreibrechte minimieren.
- Enrollment-Agent-Templates auf dedizierte, auditierte Konten beschränken (ESC3).
- SYSTEM-Automatisierung, die Security-Descriptors anhand von Gruppen ändert, kritisch prüfen (SDProp-Bypass).
