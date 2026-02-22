# Lab 10 – Consul Connect (Service Mesh)

## Ziel

Sie deployen zwei Services, die über Consul Connect (Service Mesh) sicher miteinander kommunizieren. Sie konfigurieren Sidecar-Proxies, Upstreams und Intentions.

## Voraussetzungen

- Zugang zum Remote-Cluster (Nomad + Consul mit Connect-Unterstützung).
- Consul Connect ist aktiviert (`connect { enabled = true }` in der Consul-Konfiguration).
- CNI-Plugins sind installiert.
- Eigener Namespace gesetzt (siehe Lab 04, Schritt 0): `export NOMAD_NAMESPACE=<ihr-name>`

## Hintergrund

Consul Connect bietet:

- **Sidecar-Proxies**: Envoy-Proxies, die den Traffic zwischen Services verschlüsseln (mTLS).
- **Upstreams**: Definieren, welche anderen Services ein Service erreichen darf.
- **Intentions**: Zugriffskontrolle (allow/deny) zwischen Services.

In Nomad wird Connect über die `connect`-Stanza innerhalb der `service`-Definition konfiguriert.

## Schritt 1 – Backend-Service deployen

1. Erstellen Sie eine Datei `backend.hcl`:

   ```hcl
   job "backend" {
     datacenters = ["dc1"]
     type        = "service"

     group "backend" {
       count = 1

       network {
         mode = "bridge"
         port "http" {
           to = 5678
         }
       }

       service {
         name = "backend-api"
         port = "http"

        connect {
          sidecar_service {
            proxy {
              local_service_port = 5678
            }
          }
        }
       }

       task "api" {
         driver = "docker"

         config {
           image = "hashicorp/http-echo"
           args  = ["-text=Antwort vom Backend (via Connect)", "-listen=:5678"]
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
   nomad job run backend.hcl
   ```

3. Prüfen Sie in der Consul UI, dass `backend-api` und `backend-api-sidecar-proxy` registriert sind.

## Schritt 2 – Frontend-Service mit Upstream deployen

1. Erstellen Sie eine Datei `frontend.hcl`:

   ```hcl
      job "frontend" {
     datacenters = ["dc1"]
     type        = "service"

     group "frontend" {
       count = 1

       network {
         mode = "bridge"
         port "http" {
           to = 9090
         }
       }

       service {
         name = "frontend-web"
         port = "http"

         connect {
           sidecar_service {
             proxy {
               upstreams {
                 destination_name = "backend-api"
                 local_bind_port  = 5678
               }
             }
           }
         }
       }

       task "web" {
         driver = "docker"

         config {
          image = "alpine:3"
          args  = [
            "sh", "-c",
            "while true; do echo \"Backend sagt: $(wget -q -O - http://127.0.0.1:5678 2>/dev/null)\"; sleep 5; done"
          ]
         }

         resources {
           cpu    = 100
           memory = 128
         }
       }
     }
   }
   
   ```

   Beachten Sie:
   - Der Upstream `backend-api` ist auf `local_bind_port = 5678` konfiguriert.
   - Der Frontend-Task erreicht das Backend über `http://127.0.0.1:5678` – der Sidecar-Proxy leitet den Traffic verschlüsselt weiter.

2. Reichen Sie den Job ein:

   ```bash
   nomad job run frontend.hcl
   ```

3. Prüfen Sie die Logs des Frontend-Tasks:

   ```bash
   nomad alloc logs <frontend-alloc-id>
   ```

4. Erwarteter Output (wiederholt): `Backend sagt: Antwort vom Backend (via Connect)`

## Schritt 3 – Connect-Verbindung in Consul prüfen

1. Öffnen Sie die Consul UI.
2. Navigieren Sie zu „Services" und prüfen Sie:
   - `backend-api` mit zugehörigem `backend-api-sidecar-proxy`
   - `frontend-web` mit zugehörigem `frontend-web-sidecar-proxy`

3. Unter „Topology" (falls verfügbar) können Sie die Verbindung zwischen Frontend und Backend visualisieren.

## Schritt 4 – Intentions konfigurieren

1. Standardmäßig ist der Traffic zwischen allen Services erlaubt. Erstellen Sie eine Deny-Intention:

   ```bash
   consul intention create -deny frontend-web backend-api
   ```

2. Prüfen Sie die Frontend-Logs erneut:

   ```bash
   nomad alloc logs -f <frontend-alloc-id>
   ```

   Der `curl`-Aufruf sollte jetzt fehlschlagen.

3. Erlauben Sie den Traffic wieder:

   ```bash
   consul intention delete frontend-web backend-api
   consul intention create -allow frontend-web backend-api
   ```

4. Die Frontend-Logs sollten wieder die Backend-Antwort zeigen.

## Schritt 5 – Dritten Service hinzufügen und einschränken

1. Erstellen Sie eine Datei `unauthorized-client.hcl`:

   ```hcl
   job "unauthorized-client" {
     datacenters = ["dc1"]
     type        = "service"

     group "client" {
       count = 1

       network {
         mode = "bridge"
       }

       service {
         name = "unauthorized-client"

         connect {
           sidecar_service {
             proxy {
               upstreams {
                 destination_name = "backend-api"
                 local_bind_port  = 5678
               }
             }
           }
         }
       }

       task "client" {
         driver = "docker"

         config {
           image = "alpine:3"
           args  = [
             "sh", "-c",
             "while true; do RESP=$(wget -q -O - -T 3 http://127.0.0.1:5678 2>/dev/null) && echo \"Versuch: $RESP\" || echo \"Versuch: BLOCKIERT\"; sleep 5; done"
           ]
         }

         resources {
           cpu    = 100
           memory = 128
         }
       }
     }
   }
   ```

2. Erstellen Sie Intentions, die nur dem Frontend den Zugriff erlauben:

   ```bash
   consul intention create -deny '*' backend-api
   consul intention create -allow frontend-web backend-api
   ```

3. Reichen Sie den unautorisierten Client ein:

   ```bash
   nomad job run unauthorized-client.hcl
   ```

4. Prüfen Sie die Logs:
   - **Frontend**: Sollte weiterhin Antworten vom Backend erhalten.
   - **Unauthorized Client**: Sollte `BLOCKIERT` ausgeben.

## Schritt 6 – Aufräumen

```bash
nomad job stop backend
nomad job stop frontend
nomad job stop unauthorized-client
consul intention delete frontend-web backend-api
consul intention delete '*' backend-api
```

> **Tipp:** Mit `-purge` (z.B. `nomad job stop -purge backend`) wird der Job vollständig aus Nomad entfernt. Ohne `-purge` bleibt er als „dead" sichtbar, verbraucht aber keine Cluster-Ressourcen.

## Erweiterungen (optional)

- Prüfen Sie die Envoy-Admin-Oberfläche des Sidecar-Proxies (Port 19001 im Allocation-Netzwerk).
- Konfigurieren Sie `expose`-Pfade, um Health Checks durch den Sidecar-Proxy zu leiten.

## Erfolgskriterien

- Frontend erreicht Backend über den Consul-Connect-Sidecar-Proxy.
- Die Kommunikation ist per mTLS verschlüsselt.
- Intentions blockieren unautorisierten Traffic.
- Autorisierter Traffic (Frontend → Backend) funktioniert trotz Deny-Default-Intention.
