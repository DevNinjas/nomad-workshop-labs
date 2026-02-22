# Lab 02 – Einfacher Service-Job mit Docker

## Ziel

Sie deployen einen einfachen HTTP-Service als Nomad-Job mit Docker und rufen ihn im Browser oder per `curl` auf.

## Voraussetzungen

- Lab 01 erfolgreich abgeschlossen (Nomad-Dev-Agent läuft).
- Docker ist installiert und läuft auf demselben Host.

## Schritt 1 – Job-Definition anlegen

1. Erstellen Sie eine Datei `http-echo.hcl`.
2. Fügen Sie den folgenden Inhalt ein und passen Sie ggf. das Datacenter an:

   ```hcl
   job "http-echo" {
     datacenters = ["dc1"]
     type        = "service"

     group "echo-group" {
       count = 1

      network {
        mode = "bridge"
        port "http" {
          to = 5678
        }
      }

      service {
        name        = "http-echo"
        port        = "http"
        address_mode = "alloc"
        provider = "nomad"
      }

      task "echo" {
        driver = "docker"

        config {
          image = "hashicorp/http-echo"
          args  = [
            "-text=Hello from Nomad",
            "-listen=:5678"
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

## Schritt 2 – Job einreichen

1. Reichen Sie den Job ein:

   ```bash
   nomad job run http-echo.hcl
   ```

2. Prüfen Sie, ob es Fehler gibt.

## Schritt 3 – Job-Status prüfen

1. Anzeigen des Job-Status:

   ```bash
   nomad job status http-echo
   ```

2. Fragen:
   - Wie ist der aktuelle Status des Jobs?
   - Wie viele Allocations laufen?

3. Öffnen Sie die Web-UI und suchen Sie den Job „http-echo“.

## Schritt 4 – Service aufrufen

1. Ermitteln Sie Host-Port und Node-ID der Allocation:

   ```bash
   nomad alloc status <alloc-id>
   ```

   Unter **Allocation Addresses** sehen Sie den Host-Port und die interne Node-IP:

   ```
   Label  Dynamic  Address
   http   yes      10.0.1.2X:XXXXX -> 5678
   ```

   Die öffentliche IP des Nodes ermitteln Sie über die Node-ID aus der gleichen Ausgabe:

   ```bash
   nomad node status -verbose <node-id>
   ```

   Unter **Meta** steht: `public_ip = <öffentliche IP>`

2. Rufen Sie den Service auf:

   ```bash
   curl http://<public-ip>:<host-port>
   ```

3. Erwarteter Output: `Hello from Nomad`

## Schritt 5 – Logs prüfen

1. Sehen Sie sich die Logs der Allocation an:

   ```bash
   nomad alloc logs <alloc-id>
   ```

2. Beobachten Sie:
   - Startmeldungen
   - HTTP-Requests (falls sichtbar)

## Erweiterungen (optional)

- Ändern Sie den Text von `Hello from Nomad` zu einem eigenen String.
- Erhöhen Sie `count` auf `2` und beobachten Sie, was im Cluster passiert.

## Erfolgskriterien

- Job „http-echo“ hat den Status `running`.
- HTTP-Aufruf liefert die erwartete Ausgabe.
- Sie können Logs der Task anzeigen.

