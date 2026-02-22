# Lab 11 – Multi-Task Groups & Lifecycle Hooks

## Ziel

Sie deployen eine Task Group mit mehreren Tasks, nutzen Lifecycle Hooks für Initialisierung und Cleanup und teilen Daten zwischen Tasks über das gemeinsame Allocation-Verzeichnis.

## Voraussetzungen

- Nomad-Dev-Agent läuft (lokal) oder Zugang zum Remote-Cluster.
- Docker ist installiert.
- Eigener Namespace gesetzt (siehe Lab 04, Schritt 0): `export NOMAD_NAMESPACE=<ihr-name>`

## Hintergrund

Eine Nomad Task Group kann mehrere Tasks enthalten. Alle Tasks einer Gruppe:

- Werden auf demselben Node geplant.
- Teilen sich denselben Netzwerk-Namespace (im Bridge-Mode).
- Haben Zugriff auf das gemeinsame `alloc`-Verzeichnis.

Lifecycle Hooks steuern die Reihenfolge der Task-Ausführung:

- **`prestart`**: Läuft vor dem Haupttask (z.B. für Initialisierung).
- **`poststart`**: Läuft nach dem Start des Haupttasks.
- **`poststop`**: Läuft nach dem Stoppen des Haupttasks (z.B. für Cleanup).

## Schritt 1 – Multi-Task Group: App + Sidecar

1. Erstellen Sie eine Datei `multi-task.hcl`:

   ```hcl
   job "multi-task" {
     datacenters = ["dc1"]
     type        = "service"

     group "app" {
       count = 1

       network {
         mode = "bridge"
         port "http" {
           to = 80
         }
       }

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

      task "web" {
        driver = "docker"

        config {
          image = "nginx:alpine"
          ports = ["http"]
        }

        resources {
          cpu    = 100
          memory = 64
        }
      }

       task "log-sidecar" {
         driver = "docker"

         config {
           image = "alpine:3"
           args  = [
             "sh", "-c",
             "while true; do echo \"[$(date)] Sidecar: nginx laeuft\"; sleep 10; done"
           ]
         }

         resources {
           cpu    = 50
           memory = 32
         }
       }
     }
   }
   ```

2. Reichen Sie den Job ein:

   ```bash
   nomad job run multi-task.hcl
   ```

3. Prüfen Sie den Status:

   ```bash
   nomad job status multi-task
   ```

4. Beide Tasks sollten in derselben Allocation laufen. Prüfen Sie die Logs beider Tasks:

   ```bash
   nomad alloc logs <alloc-id> web
   nomad alloc logs <alloc-id> log-sidecar
   ```

## Schritt 2 – Prestart-Task für Initialisierung

1. Erstellen Sie eine Datei `lifecycle-demo.hcl`:

   ```hcl
   job "lifecycle-demo" {
     datacenters = ["dc1"]
     type        = "service"

     group "app" {
       count = 1

       network {
         mode = "bridge"
         port "http" {
           to = 80
         }
       }

       task "init" {
         driver = "docker"

         lifecycle {
           hook    = "prestart"
           sidecar = false
         }

         config {
           image = "alpine:3"
           args  = [
             "sh", "-c",
             "echo '<!DOCTYPE html><html><body><h1>Generiert vom Init-Task</h1><p>Zeitstempel: '$(date)'</p></body></html>' > /alloc/data/index.html && echo 'Init abgeschlossen'"
           ]
         }

         resources {
           cpu    = 50
           memory = 32
         }
       }

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

      task "web" {
        driver = "docker"

        config {
          image = "nginx:alpine"
          ports = ["http"]
        }

        template {
          data = <<-EOF
            server {
              listen 80;
              location / {
                root /alloc/data;
                index index.html;
              }
            }
          EOF

          destination = "local/default.conf"
        }

         resources {
           cpu    = 100
           memory = 64
         }
       }
     }
   }
   ```

   Beachten Sie:
   - Der `init`-Task hat `lifecycle { hook = "prestart" }` und schreibt eine HTML-Datei in `/alloc/data/`.
   - Der `web`-Task startet erst, nachdem `init` erfolgreich abgeschlossen ist.
   - Beide Tasks teilen sich das `/alloc/data`-Verzeichnis.

2. Reichen Sie den Job ein:

   ```bash
   nomad job run lifecycle-demo.hcl
   ```

3. Prüfen Sie die Logs des Init-Tasks:

   ```bash
   nomad alloc logs <alloc-id> init
   ```

   Erwarteter Output: `Init abgeschlossen`

4. Rufen Sie den Web-Service über Traefik auf und prüfen Sie, ob die generierte HTML-Seite angezeigt wird:

   ```bash
   curl https://lifecycle-demo-<namespace>.nomad.devninjas.io
   ```

   Oder öffnen Sie die URL im Browser.

## Schritt 3 – Poststop-Task für Cleanup

1. Erstellen Sie eine Datei `poststop-demo.hcl`:

   ```hcl
   job "poststop-demo" {
     datacenters = ["dc1"]
     type        = "batch"

     group "app" {
       task "main" {
         driver = "docker"

         config {
           image = "alpine:3"
           args  = ["sh", "-c", "echo 'Haupttask laeuft...'; sleep 5; echo 'Haupttask fertig'"]
         }

         resources {
           cpu    = 50
           memory = 32
         }
       }

       task "cleanup" {
         driver = "docker"

         lifecycle {
           hook    = "poststop"
           sidecar = false
         }

         config {
           image = "alpine:3"
           args  = ["sh", "-c", "echo 'Poststop: Cleanup wird ausgefuehrt...'; sleep 2; echo 'Cleanup abgeschlossen'"]
         }

         resources {
           cpu    = 50
           memory = 32
         }
       }
     }
   }
   ```

2. Reichen Sie den Job ein:

   ```bash
   nomad job run poststop-demo.hcl
   ```

3. Warten Sie, bis der Batch-Job abgeschlossen ist:

   ```bash
   nomad job status poststop-demo
   ```

4. Prüfen Sie die Logs beider Tasks:

   ```bash
   nomad alloc logs <alloc-id> main
   nomad alloc logs <alloc-id> cleanup
   ```

5. Die Reihenfolge sollte sein:
   1. `main` startet und läuft 5 Sekunden.
   2. `main` endet.
   3. `cleanup` startet und führt den Cleanup durch.

## Schritt 4 – Aufräumen

```bash
nomad job stop multi-task
nomad job stop lifecycle-demo
```

> **Tipp:** Mit `-purge` (z.B. `nomad job stop -purge multi-task`) wird der Job vollständig aus Nomad entfernt. Ohne `-purge` bleibt er als „dead" sichtbar, verbraucht aber keine Cluster-Ressourcen.

## Erweiterungen (optional)

- Fügen Sie einen `poststart`-Task hinzu, der nach dem Start des Haupttasks eine Konfiguration validiert.
- Kombinieren Sie `prestart` und `poststop` in derselben Task Group.
- Nutzen Sie `sidecar = true` bei einem Prestart-Task, damit er neben dem Haupttask weiterläuft (z.B. als Log-Shipper).

## Erfolgskriterien

- Mehrere Tasks laufen in derselben Allocation.
- Prestart-Task wird vor dem Haupttask ausgeführt.
- Poststop-Task wird nach Beendigung des Haupttasks ausgeführt.
- Tasks teilen Daten über das gemeinsame `alloc`-Verzeichnis.
