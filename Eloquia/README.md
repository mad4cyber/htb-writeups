# Hack The Box: Eloquia Writeup

> Public-safe Writeup. Flags sind absichtlich nicht enthalten.

## Überblick

Eloquia war eine Windows-HTB-Maschine mit zwei gekoppelten Django-Webanwendungen:

- `eloquia.htb`
- `qooqle.htb`

Die Hauptkette bestand aus einem OAuth-Linking-Fehler ohne ausreichenden `state`-/CSRF-Schutz, Admin-Bot-Delivery, Django SQL Explorer Missbrauch, SQLite `load_extension()` zur Codeausführung, Browser-Credential-Recovery und einer Windows-Service-Binary-Race bis `NT AUTHORITY\SYSTEM`.

## Recon

Relevante Dienste:

```text
80/tcp    HTTP
5985/tcp  WinRM
```

VHosts:

```text
eloquia.htb
qooqle.htb
```

Auffällig war früh, dass Eloquia OAuth gegen Qooqle nutzt. Daraus entstand die wichtigste Angriffsfläche: Account-Linking über OAuth-Codes.

## Foothold

### OAuth-State-Fehler

Der OAuth-Flow konnte ohne robusten `state`-/CSRF-Schutz missbraucht werden:

1. In einer eigenen Qooqle-Session wurde ein frischer OAuth-Code erzeugt.
2. Dieser Callback-Code wurde in einer anderen Eloquia-Session konsumiert.
3. Dadurch ließ sich die fremde Eloquia-Identität an die kontrollierte Qooqle-Identität binden.

Wichtig war die Trennung zwischen:

- OAuth-Primitive funktioniert manuell
- Admin-Bot-Delivery muss nur noch zuverlässig ausgelöst werden

### Admin-Bot-Delivery

Statt eine komplexe Browser-Exploit-Kette zu bauen, reichte ein einfacher HTML-Delivery-Ansatz über erhaltene Tags im Artikelinhalt. Ein frischer OAuth-Callback wurde per Bot in dessen authentifizierter Session aufgerufen, wodurch Admin-Zugriff auf Eloquia erreicht wurde.

## Post-Auth Enumeration

Mit Admin-Zugriff war Django SQL Explorer erreichbar:

```text
/dev/sql-explorer/play/
```

Die Standard-Django-Verbindung war für manche Primitives ungeeignet, daher wurde eine direkte SQLite-Verbindung genutzt. Damit waren mehrere Dinge möglich:

- Tabellen und Benutzer enumerieren
- SQLite-Datenbanken über `VACUUM INTO` in weblesbare statische Pfade exportieren
- Sibling-Datenbanken über relative Pfade untersuchen, z.B. Qooqle neben Eloquia
- `load_extension()` als RCE-Kandidat testen

## SQLite `load_extension()`

Entscheidend war die Fehlerklasse:

- `not authorized` hätte bedeutet: Extension Loading blockiert
- OS-/Pfad-/Library-Fehler bedeuteten: der Loader wird erreicht

UNC-Pfade zu einer Angreifer-SMB-Freigabe triggerten Outbound-Authentifizierung des Web-/App-Kontexts. Das war ein guter Capability-Proof, aber noch keine RCE. Da unauthenticated SMB/Guest-Zugriff geblockt war, wurde die DLL nicht weiter über SMB geladen, sondern lokal auf dem Ziel platziert und anschließend per `load_extension()` geladen.

Die DLL führte minimalen Code aus und schrieb Befehlsausgaben in einen weblesbaren Pfad. Damit war Codeausführung als Web-/Service-Kontext bestätigt.

## Windows Enumeration

Nach RCE auf Windows wurde nicht sofort breit in AD eskaliert. Der bessere nächste Schritt war lokale Credential-Enumeration.

Besonders wertvoll waren Edge/Chromium-Artefakte im Benutzerprofil:

```text
User Data
Login Data
Local State
```

Die Browser-Credentials wurden ausgelesen/decrypted und lieferten brauchbare Zugangsdaten für einen interaktiveren Windows-Kontext.

## WinRM

Die wiederhergestellten Credentials wurden gegen WinRM validiert:

```text
5985/tcp WinRM
```

Damit war eine Evil-WinRM-Session als normaler Benutzer möglich. Ab hier wurde lokal nach klassischen Windows-Privesc-Primitives gesucht:

- Benutzerrechte
- beschreibbare Dateien/Verzeichnisse
- Services
- Scheduled Tasks
- Custom Software unter `Program Files`

## Privilege Escalation

Der entscheidende Fund war eine custom Service-Binary unter:

```text
C:\Program Files\Qooqle IPS Software\Failure2Ban - Prototype\Failure2Ban\bin\Debug\Failure2Ban.exe
```

Die Binary war für den Benutzer schreibbar, aber im laufenden Betrieb meistens gelockt. Wichtig: Ein gelocktes, aber schreibbares Service-Executable ist nicht automatisch nutzlos.

Da der Service periodisch neu startete, wurde eine bounded Copy-Race-Schleife genutzt:

```powershell
$src='C:\Path\To\payload.exe'
$dst='C:\Program Files\Qooqle IPS Software\Failure2Ban - Prototype\Failure2Ban\bin\Debug\Failure2Ban.exe'
$end=(Get-Date).AddMinutes(7)
$n=0
while((Get-Date) -lt $end){
  $n++
  try {
    Copy-Item -LiteralPath $src -Destination $dst -Force -ErrorAction Stop
    Write-Output "SUCCESS $n $(Get-Date)"
    break
  } catch {
    Start-Sleep -Milliseconds 20
  }
}
Write-Output "DONE $n $(Get-Date)"
```

Beim Restart-Zeitfenster konnte die Binary ersetzt werden. Die Payload lief anschließend als:

```text
nt authority\system
```

Damit war die Maschine vollständig kompromittiert.

## Kill Chain

```text
VHost Enumeration
→ paired Django apps: Eloquia + Qooqle
→ OAuth missing state / CSRF weakness
→ Admin-Bot consumes attacker OAuth callback
→ Eloquia admin access
→ Django SQL Explorer
→ direct SQLite connection
→ DB export / sibling DB pivot
→ SQLite load_extension reaches OS loader
→ local DLL placement and command execution
→ Edge/Chromium credential extraction
→ WinRM as recovered user
→ writable but locked Failure2Ban.exe service binary
→ copy-race during service restart
→ SYSTEM
```

## Takeaways

- Bei OAuth-Flows immer zuerst prüfen, ob `state` wirklich sessiongebunden validiert wird.
- Wenn der manuelle OAuth-Missbrauch funktioniert, ist Bot-Ausnutzung oft nur noch ein Delivery-Problem.
- SQL Explorer/Admin-Debug-Tools sind auf CTF-Boxen oft direkte Privesc-/RCE-Hilfen.
- Bei SQLite `load_extension()` ist die Fehlerklasse entscheidend: OS-Fehler sind ein gutes Zeichen, `not authorized` nicht.
- UNC-Loads sind nützlich für NetNTLM/Capability-Proofs, aber nicht automatisch RCE.
- Nach Windows-Web-RCE sofort lokale Browser-Credential-Stores prüfen.
- Eine gelockte Service-Binary mit Schreibrechten kann über Restart-/Copy-Race trotzdem exploitable sein.
- Minimalpayloads mit Output nach `C:\Windows\Temp` oder weblesbaren Pfaden sind oft stabiler als Reverse Shells.

## Empfehlungen

- OAuth-Callbacks müssen `state` serverseitig pro Session validieren und Codes nur im korrekten Kontext akzeptieren.
- Admin-Bots sollten HTML restriktiv sanitisieren und keine privilegierten Sessions für untrusted Content verwenden.
- SQL Explorer/Admin-Tools dürfen nicht produktiv oder auf privilegierten Datenbankverbindungen erreichbar sein.
- SQLite Extension Loading sollte deaktiviert bleiben.
- Browser-Credentials sollten nicht in Service-/Admin-Kontexten gespeichert werden.
- Service-Binaries und Installationsverzeichnisse unter `Program Files` dürfen nicht für normale Benutzer schreibbar sein.
- Services sollten signierte Binaries prüfen oder mit sicheren ACLs und Update-Mechanismen betrieben werden.

## Public Safety

Flags, private Schlüssel, vollständige Hashes, Session-Cookies und unnötige Secrets wurden aus diesem Writeup entfernt.
