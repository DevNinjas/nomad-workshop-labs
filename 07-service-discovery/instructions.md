# Lab 07 – Service Discovery mit Consul

## Ziel

Sie registrieren Services in Consul über Nomad und beobachten Health Checks, Service-Registrierung und DNS-Auflösung in der Consul UI und über die Consul HTTP-API.

## Voraussetzungen

- Zugang zum Remote-Cluster (Nomad + Consul laufen).
- Consul UI erreichbar unter `https://consul.nomad.devninjas.io`
- Eigener Namespace gesetzt (siehe Lab 04, Schritt 0): `export NOMAD_NAMESPACE=<ihr-name>`
- Consul CLI auf Ihrem Rechner (optional, aber empfohlen):

  ```bash
  export CONSUL_HTTP_ADDR=https://consul.nomad.devninjas.io
  ```

## Hintergrund

Wenn Nomad mit Consul integriert ist, registriert jede `service`-Stanza in einem Nomad-Job automatisch einen Service in Consul. Consul bietet dann:

- **DNS-Interface**: Services sind unter `<service-name>.service.consul` erreichbar (clusterintern).
- **HTTP-API**: Programmatischer Zugriff auf Service-Informationen – auch von außen erreichbar.
- **Health Checks**: Automatische Überwachung der Service-Gesundheit.

## Schritt 1 – Service mit Health Check deployen

1. Erstellen Sie eine Datei `web-with-check.hcl`:

   ```hcl
   job "web-with-check" {
     datacenters = ["dc1"]
     type        = "service"

     group "web" {
       count = 2

       network {
         mode = "bridge"
         port "http" {
           to = 5678
         }
       }

       service {
         name = "web-app"
         port = "http"

         check {
           name     = "web-app-health"
           type     = "http"
           path     = "/health"
           interval = "10s"
           timeout  = "2s"
         }
       }

       task "echo" {
         driver = "docker"

         config {
           image = "hashicorp/http-echo"
           args  = ["-text=Healthy!", "-listen=:5678"]
         }

         resources {
           cpu    = 100
           memory = 64
         }
       }
     }
   }
   ```

2. Reichen Sie den Job ein:

   ```bash
   nomad job run web-with-check.hcl
   ```

## Schritt 2 – Service in der Consul UI beobachten

1. Öffnen Sie die Consul UI: `https://consul.nomad.devninjas.io`
2. Navigieren Sie zu **Services**.
3. Suchen Sie den Service `web-app`.
4. Klicken Sie auf den Service und beobachten Sie:
   - Wie viele Instanzen sind registriert? (sollten 2 sein)
   - Welchen Health-Status haben sie? (grün = passing)
   - Welche IPs und Ports sind eingetragen?

5. Alternativ per Consul CLI:

   ```bash
   # Alle registrierten Services anzeigen
   consul catalog services

   # Nodes anzeigen, die web-app bereitstellen
   consul catalog nodes -service=web-app
   ```

> **Hinweis**: Da mehrere Teilnehmer denselben Service-Namen `web-app` verwenden, sehen Sie in der Consul UI und per CLI alle Instanzen aller Teilnehmer. Die eigenen Instanzen erkennen Sie an der Node-ID.

## Schritt 3 – Service-Informationen abfragen

Die Consul HTTP-API und CLI sind von außen erreichbar (kein SSH nötig).

1. Nodes anzeigen, die den Service bereitstellen:

   ```bash
   consul catalog nodes -service=web-app
   ```

2. Alle Instanzen mit Health-Status (HTTP-API):

   ```bash
   curl -s "https://consul.nomad.devninjas.io/v1/health/service/web-app" | jq '.[].Service | {ID, Address, Port}'
   ```

3. Nur healthy Instanzen:

   ```bash
   curl -s "https://consul.nomad.devninjas.io/v1/health/service/web-app?passing=true" | jq length
   ```

3. Fragen:
   - Wie viele Instanzen sind registered? Wie viele passing?
   - Was sehen Sie bei `Address` und `Port`?

## Schritt 4 – Consul DNS: Service-zu-Service-Kommunikation

Die DNS-Auflösung (`web-app.service.consul`) funktioniert nur innerhalb des Clusters. Der beste Weg, das zu demonstrieren, ist ein Job, der einen anderen Job per Consul-DNS aufruft:

1. Erstellen Sie eine Datei `consumer.hcl`:

   ```hcl
   job "consumer" {
     datacenters = ["dc1"]
     type        = "batch"

     group "consumer" {
       network {
         mode = "host"
       }

       task "call-web" {
         driver = "docker"

         config {
           image        = "curlimages/curl:latest"
           network_mode = "host"
           args  = [
             "-sv",
             "http://web-app.service.consul:5678",
           ]
         }

         resources {
           cpu    = 100
           memory = 64
         }
       }
     }
   }
   ```

2. Reichen Sie den Job ein:

   ```bash
   nomad job run consumer.hcl
   ```

3. Prüfen Sie die Logs:

   ```bash
   nomad alloc logs <alloc-id>
   ```

4. Erwarteter Output: `Healthy!` – der Consumer hat den Service per Consul-DNS gefunden und aufgerufen.

5. Beobachten Sie die Consul UI: Welche Instanz von `web-app` hat den Request beantwortet?

## Schritt 5 – Health Check Failure beobachten

1. Stoppen Sie eine der `web-with-check`-Allocations:

   ```bash
   nomad alloc stop <alloc-id>
   ```

2. Beobachten Sie sofort in der **Consul UI** (`Services → web-app`):
   - Wechselt die gestoppte Instanz auf `critical`?
   - Wie schnell wird sie als unhealthy markiert?

3. Prüfen Sie, ob die gestoppte Instanz aus den healthy Services verschwindet:

   ```bash
   curl -s "https://consul.nomad.devninjas.io/v1/health/service/web-app?passing=true" | jq length
   ```

   Vorher waren es 2, jetzt sollte es 1 sein.

4. Nomad startet automatisch eine neue Allocation. Beobachten Sie in der Consul UI, wie der Service wieder auf 2 healthy Instanzen hochläuft.

## Schritt 6 – Aufräumen

```bash
nomad job stop web-with-check
nomad job stop consumer
```

> **Tipp:** Mit `-purge` (z.B. `nomad job stop -purge web-with-check`) wird der Job vollständig aus Nomad entfernt. Ohne `-purge` bleibt er als „dead" sichtbar, verbraucht aber keine Cluster-Ressourcen.

## Erweiterungen (optional)

- Fügen Sie dem `web-with-check`-Job Tags hinzu (`tags = ["v1", "production"]`) und filtern Sie in Consul danach:
  ```bash
  curl -s "https://consul.nomad.devninjas.io/v1/health/service/web-app?tag=production&passing=true"
  ```
- Ändern Sie `count = 3` und beobachten Sie, wie Consul automatisch eine neue Instanz registriert.

## Erfolgskriterien

- Service `web-app` ist in Consul registriert und in der UI sichtbar.
- Consul HTTP-API liefert Service-Informationen von außen.
- Consumer-Job erreicht `web-app` per Consul-DNS (`web-app.service.consul`).
- Bei Ausfall einer Instanz aktualisiert Consul den Health-Status automatisch.
