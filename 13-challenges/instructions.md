# Lab 13 – Aufgaben & Challenges

## Ziel

Diese Aufgaben sind offen gestaltet – es gibt keine Schritt-für-Schritt-Anleitung. Sie erhalten ein Ziel und Hinweise. Nutzen Sie die Nomad-Dokumentation, die vorangegangenen Labs und Ihre Erfahrung, um die Aufgaben zu lösen.

## Voraussetzungen

- Labs 01–12 durchgearbeitet oder vergleichbare Kenntnisse.
- Zugang zum lokalen Dev-Cluster und/oder Remote-Cluster.
- Namespace gesetzt: `export NOMAD_NAMESPACE=<ihr-name>`

## Traefik-Zugriff

Services sind über `https://<job-name>-<namespace>.nomad.devninjas.io` erreichbar, wenn der Job folgende Tags im `service`-Block hat:

```hcl
service {
  name = "${NOMAD_JOB_NAME}"
  port = "http"

  tags = [
    "traefik.enable=true",
    "traefik.http.routers.${NOMAD_JOB_NAME}-${NOMAD_NAMESPACE}.rule=Host(`${NOMAD_JOB_NAME}-${NOMAD_NAMESPACE}.nomad.devninjas.io`)",
    "traefik.http.routers.${NOMAD_JOB_NAME}-${NOMAD_NAMESPACE}.entrypoints=https",
    "traefik.http.routers.${NOMAD_JOB_NAME}-${NOMAD_NAMESPACE}.tls=true",
    "traefik.http.routers.${NOMAD_JOB_NAME}-${NOMAD_NAMESPACE}.tls.certresolver=letsencrypt",
  ]
}
```

---

## Aufgabe 1 – Webserver mit Custom-Content

**Schwierigkeit**: Einsteiger

**Ziel**: Deployen Sie einen Nginx-Container, der eine selbst gestaltete HTML-Seite ausliefert. Die Seite soll mindestens den Namen des Jobs und den aktuellen Zeitstempel enthalten.

**Hinweise**:
- Nutzen Sie die `template`-Stanza, um eine `index.html` zu generieren.
- Nomad stellt Variablen wie `NOMAD_JOB_NAME`, `NOMAD_ALLOC_ID` und `NOMAD_NODE_NAME` bereit.
- Nginx kann Dateien aus `/local` oder `/alloc/data` ausliefern.

**Erfolgskriterium**: Der HTTP-Aufruf liefert eine HTML-Seite mit dynamisch generierten Inhalten.

---

## Aufgabe 2 – Periodischer Job

**Schwierigkeit**: Einsteiger–Mittel

**Ziel**: Erstellen Sie einen Batch-Job, der alle 5 Minuten ausgeführt wird und eine Log-Nachricht mit Zeitstempel schreibt.

**Hinweise**:
- Nutzen Sie den Job-Typ `batch` zusammen mit der `periodic`-Stanza.
- Die Cron-Syntax in Nomad verwendet 5 Felder (ohne Sekunden): `cron = "*/5 * * * *"`.
- Setzen Sie `prohibit_overlap = true`, um parallele Ausführungen zu verhindern.
- Prüfen Sie die Child-Jobs in der UI oder per `nomad job status`.

**Erfolgskriterium**: Der Job wird automatisch alle 5 Minuten gestartet. Jede Ausführung ist in der Job-Historie sichtbar.

---

## Aufgabe 3 – Service Mesh Microservices

**Schwierigkeit**: Fortgeschritten

**Ziel**: Deployen Sie ein Frontend und ein Backend als separate Nomad-Jobs. Das Frontend soll das Backend über Consul Connect erreichen. Konfigurieren Sie Intentions, sodass nur das Frontend Zugriff auf das Backend hat.

**Hinweise**:
- Beide Services benötigen `network { mode = "bridge" }` und `connect { sidecar_service {} }`.
- Das Frontend definiert einen `upstream` zum Backend.
- Intentions werden per `consul intention create` gesetzt.
- Testen Sie zuerst ohne Intentions, dann mit einer Deny-Default-Regel.

**Erfolgskriterium**: Frontend erreicht Backend über Connect. Ein dritter, unautorisierter Service wird blockiert.

---

## Aufgabe 4 – Zero-Downtime Deployment

**Schwierigkeit**: Fortgeschritten

**Ziel**: Deployen Sie einen Service mit `count = 3` und konfigurieren Sie die `update`-Stanza so, dass bei einem Rolling Update zu keinem Zeitpunkt weniger als 2 Instanzen verfügbar sind.

**Hinweise**:
- `max_parallel` steuert, wie viele Instanzen gleichzeitig ersetzt werden.
- `min_healthy_time` bestimmt, wie lange eine neue Instanz healthy sein muss.
- Fügen Sie dem Job Traefik-Tags hinzu (siehe Lab 09-Muster), damit der Service über `https://<job>-<namespace>.nomad.devninjas.io` erreichbar ist.
- Verifizieren Sie mit einer Schleife:

  ```bash
  while true; do curl -s https://<job>-<namespace>.nomad.devninjas.io; sleep 0.5; done
  ```

- Nutzen Sie einen Health Check, damit Nomad die neue Instanz erst als healthy markiert, wenn sie Requests beantworten kann.

**Erfolgskriterium**: Während des gesamten Update-Vorgangs sind immer mindestens 2 Instanzen erreichbar.

---

## Aufgabe 5 – Node Drain

**Schwierigkeit**: Fortgeschritten

**Ziel**: Setzen Sie einen Client-Node in den Drain-Modus und beobachten Sie, wie Nomad die Allocations auf den verbleibenden Node migriert.

**Hinweise**:
- `nomad node drain -enable <node-id>` aktiviert den Drain.
- `nomad node drain -disable <node-id>` deaktiviert ihn.
- Deployen Sie vorher einen Job mit `count = 4` und `spread`, damit Allocations auf beiden Nodes laufen.
- Beobachten Sie den Vorgang mit `nomad node status <node-id>` und `nomad job status <job>`.
- Frage: Was passiert, wenn der verbleibende Node nicht genug Ressourcen hat?

**Erfolgskriterium**: Alle Allocations werden vom drainable Node migriert. Nach Deaktivierung des Drains kann der Node wieder Allocations aufnehmen.

---

## Aufgabe 6 – Custom Health Check

**Schwierigkeit**: Mittel

**Ziel**: Deployen Sie einen Service mit einem HTTP-Health-Check, der nicht nur den Status-Code, sondern auch den Response-Body prüft.

**Hinweise**:
- Nutzen Sie einen `script`-Health-Check mit `curl` und `grep`:

  ```hcl
  check {
    type     = "script"
    command  = "/bin/sh"
    args     = ["-c", "curl -s http://localhost:5678 | grep -q 'healthy'"]
    interval = "10s"
    timeout  = "5s"
  }
  ```

- Alternativ: `check_restart`-Stanza, um den Task automatisch neu zu starten, wenn der Check fehlschlägt.

**Erfolgskriterium**: Der Health Check prüft den Response-Body und markiert den Service als healthy oder unhealthy basierend auf dem Inhalt.

---

## Aufgabe 7 – Parameterized Job (Dispatch)

**Schwierigkeit**: Mittel

**Ziel**: Erstellen Sie einen parametrisierten Job, der per `nomad job dispatch` mit verschiedenen Parametern gestartet werden kann.

**Hinweise**:
- Nutzen Sie die `parameterized`-Stanza:

  ```hcl
  parameterized {
    meta_required = ["target_url"]
  }
  ```

- Die Meta-Variablen sind im Task als `${NOMAD_META_target_url}` verfügbar.
- Dispatch mit:

  ```bash
  nomad job dispatch -meta target_url="https://example.com" <job-name>
  ```

- Anwendungsfall: Ein Job, der eine URL per `curl` abruft und das Ergebnis loggt.

**Erfolgskriterium**: Der Job kann mehrmals mit unterschiedlichen Parametern dispatched werden. Jede Dispatch-Instanz erscheint als eigene Allocation.

---

## Aufgabe 8 – Resource Limits testen

**Schwierigkeit**: Mittel

**Ziel**: Deployen Sie einen Job mit absichtlich zu wenig Memory und beobachten Sie das OOM-Verhalten (Out of Memory).

**Hinweise**:
- Setzen Sie `memory = 16` (16 MB) bei einem Container, der mehr verbraucht.
- Nutzen Sie ein Image wie `progrium/stress` oder einen einfachen Befehl:

  ```bash
  dd if=/dev/zero of=/dev/null bs=1M count=100
  ```

  oder ein Python-Skript, das Speicher allokiert.
- Beobachten Sie die Allocation-Events:

  ```bash
  nomad alloc status <alloc-id>
  ```

- Achten Sie auf Events wie `OOM Killed` oder `Memory limit exceeded`.
- Prüfen Sie, ob Nomad die Task automatisch neu startet (`restart`-Policy).

**Erfolgskriterium**: Sie können erklären, was bei OOM passiert und wie die `restart`-Stanza das Verhalten steuert.

---

## Allgemeine Tipps

- **Nomad-Dokumentation**: [developer.hashicorp.com/nomad/docs](https://developer.hashicorp.com/nomad/docs)
- **Job-Spezifikation**: `nomad job validate <datei>` prüft die Syntax vor dem Einreichen.
- **Debugging**: `nomad alloc status` und `nomad alloc logs` sind Ihre besten Werkzeuge.
- **UI**: Die Nomad Web-UI zeigt Events, Logs und Deployment-Status übersichtlich an.
