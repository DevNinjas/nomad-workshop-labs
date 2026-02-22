# Lab 09 – Update-Strategien & Deployments

## Ziel

Sie konfigurieren verschiedene Update-Strategien in Nomad und beobachten das Verhalten bei Rolling Updates, Canary Deployments und automatischen Rollbacks.

## Voraussetzungen

- Zugang zum Remote-Cluster (mindestens 2 Client-Nodes).
- Docker ist installiert.
- Eigener Namespace gesetzt (siehe Lab 04, Schritt 0): `export NOMAD_NAMESPACE=<ihr-name>`

## Hintergrund

Die `update`-Stanza in Nomad steuert, wie neue Versionen eines Jobs ausgerollt werden:

- **Rolling Update**: Instanzen werden schrittweise ersetzt.
- **Canary Deployment**: Eine neue Version wird parallel zur alten gestartet und muss manuell oder automatisch promoted werden.
- **Auto-Revert**: Bei fehlgeschlagenem Health Check wird automatisch zur vorherigen Version zurückgekehrt.

## Schritt 1 – Service mit Rolling Update deployen

1. Erstellen Sie eine Datei `rolling-update.hcl`:

   ```hcl
   job "rolling-update" {
     datacenters = ["dc1"]
     type        = "service"

     update {
       max_parallel     = 1
       min_healthy_time = "10s"
       healthy_deadline = "1m"
       auto_revert      = false
     }

     group "web" {
       count = 3

       network {
         mode = "bridge"
         port "http" {
           to = 5678
         }
       }

      service {
        name = "rolling-web"
        port = "http"

        tags = [
          "traefik.enable=true",
          "traefik.http.routers.${NOMAD_JOB_NAME}-${NOMAD_NAMESPACE}.rule=Host(`${NOMAD_JOB_NAME}-${NOMAD_NAMESPACE}.nomad.devninjas.io`)",
          "traefik.http.routers.${NOMAD_JOB_NAME}-${NOMAD_NAMESPACE}.entrypoints=https",
          "traefik.http.routers.${NOMAD_JOB_NAME}-${NOMAD_NAMESPACE}.tls=true",
          "traefik.http.routers.${NOMAD_JOB_NAME}-${NOMAD_NAMESPACE}.tls.certresolver=letsencrypt",
        ]

        check {
          type     = "http"
          path     = "/"
          interval = "5s"
          timeout  = "2s"
        }
      }

       task "echo" {
         driver = "docker"

         config {
           image = "hashicorp/http-echo"
           args  = ["-text=Version 2", "-listen=:5678"]
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
   nomad job run rolling-update.hcl
   ```

3. Warten Sie, bis alle 3 Instanzen `healthy` sind:

   ```bash
   nomad job status rolling-update
   ```

4. Testen Sie den Service:

   ```bash
   curl https://rolling-update-<namespace>.nomad.devninjas.io
   ```

## Schritt 2 – Rolling Update durchführen

1. Ändern Sie den Text in der Job-Datei von `Version 1` auf `Version 2`.

2. Reichen Sie den Job erneut ein:

   ```bash
   nomad job run rolling-update.hcl
   ```

3. Beobachten Sie den Deployment-Fortschritt:

   ```bash
   nomad job status rolling-update
   ```

   Wiederholen Sie den Befehl mehrmals. Sie sollten sehen, wie Instanzen nacheinander (`max_parallel = 1`) ersetzt werden.

4. Prüfen Sie auch die Deployment-Details:

   ```bash
   nomad deployment list
   nomad deployment status <deployment-id>
   ```

5. Frage: Warum dauert das Update mindestens 30 Sekunden (3 Instanzen × `min_healthy_time = 10s`)?

## Schritt 3 – Canary Deployment

1. Erstellen Sie eine Datei `canary-deploy.hcl`:

   ```hcl
   job "canary-deploy" {
     datacenters = ["dc1"]
     type        = "service"

     update {
       max_parallel     = 1
       canary           = 1
       min_healthy_time = "10s"
       healthy_deadline = "1m"
       auto_promote     = false
       auto_revert      = true
     }

     group "web" {
       count = 3

       network {
         mode = "bridge"
         port "http" {
           to = 5678
         }
       }

      service {
        name = "canary-web"
        port = "http"

        tags = [
          "traefik.enable=true",
          "traefik.http.routers.${NOMAD_JOB_NAME}-${NOMAD_NAMESPACE}.rule=Host(`${NOMAD_JOB_NAME}-${NOMAD_NAMESPACE}.nomad.devninjas.io`)",
          "traefik.http.routers.${NOMAD_JOB_NAME}-${NOMAD_NAMESPACE}.entrypoints=https",
          "traefik.http.routers.${NOMAD_JOB_NAME}-${NOMAD_NAMESPACE}.tls=true",
          "traefik.http.routers.${NOMAD_JOB_NAME}-${NOMAD_NAMESPACE}.tls.certresolver=letsencrypt",
        ]

        check {
          type     = "http"
          path     = "/"
          interval = "5s"
          timeout  = "2s"
        }
      }

       task "echo" {
         driver = "docker"

         config {
           image = "hashicorp/http-echo"
           args  = ["-text=Stable v1", "-listen=:5678"]
         }

         resources {
           cpu    = 100
           memory = 64
         }
       }
     }
   }
   ```

2. Reichen Sie den Job ein und warten Sie, bis alle Instanzen healthy sind:

   ```bash
   nomad job run canary-deploy.hcl
   ```

3. Ändern Sie den Text auf `Canary v2` und reichen Sie den Job erneut ein:

   ```bash
   nomad job run canary-deploy.hcl
   ```

4. Prüfen Sie den Status:

   ```bash
   nomad job status canary-deploy
   ```

   Sie sollten sehen:
   - 3 Allocations mit `Stable v1` (laufend)
   - 1 Allocation mit `Canary v2` (Canary, laufend)

5. Testen Sie den Service über Traefik – Traefik verteilt die Anfragen auf alle Instanzen (stabile + Canary).
   Führen Sie den Befehl mehrmals aus, um beide Versionen zu sehen:

   ```bash
   for i in $(seq 1 10); do curl -s https://canary-deploy-<namespace>.nomad.devninjas.io; echo; done
   ```

## Schritt 4 – Canary promoten oder reverten

**Option A – Promote (Canary wird zur neuen Version):**

```bash
nomad deployment promote <deployment-id>
```

Beobachten Sie, wie die verbleibenden Instanzen auf `Canary v2` aktualisiert werden.

**Option B – Rollback (Canary wird verworfen):**

```bash
nomad deployment fail <deployment-id>
```

Die Canary-Instanz wird gestoppt und alle Instanzen bleiben auf `Stable v1`.

## Schritt 5 – Auto-Revert bei fehlerhaftem Update

1. Reichen Sie den Job mit einer funktionierenden Version ein (falls nötig).

2. Ändern Sie das Image auf ein nicht existierendes Image:

   ```hcl
   config {
     image = "hashicorp/http-echo:nonexistent-tag"
     args  = ["-text=Broken v3", "-listen=:5678"]
   }
   ```

3. Reichen Sie den Job ein:

   ```bash
   nomad job run canary-deploy.hcl
   ```

4. Beobachten Sie den Deployment-Status:

   ```bash
   nomad deployment list
   nomad deployment status <deployment-id>
   ```

5. Da die Canary-Allocation nicht healthy wird und `auto_revert = true` gesetzt ist, sollte Nomad automatisch zur vorherigen Version zurückkehren.

6. Prüfen Sie die Job-Events in der UI oder per CLI:

   ```bash
   nomad job history canary-deploy
   ```

## Schritt 6 – Aufräumen

```bash
nomad job stop rolling-update
nomad job stop canary-deploy
```

> **Tipp:** Mit `-purge` (z.B. `nomad job stop -purge rolling-update`) wird der Job vollständig aus Nomad entfernt. Ohne `-purge` bleibt er als „dead" sichtbar, verbraucht aber keine Cluster-Ressourcen.

## Erweiterungen (optional)

- Konfigurieren Sie `auto_promote = true` mit einer `min_healthy_time = "30s"`. Beobachten Sie, wie der Canary automatisch promoted wird, wenn er 30 Sekunden lang healthy ist.
- Erhöhen Sie `max_parallel = 2` beim Rolling Update und beobachten Sie den Unterschied.
- Setzen Sie `progress_deadline` und beobachten Sie, was passiert, wenn ein Deployment zu lange dauert.

## Erfolgskriterien

- Rolling Update ersetzt Instanzen schrittweise.
- Canary Deployment startet eine Testinstanz parallel zur stabilen Version.
- Promote und Rollback funktionieren über CLI.
- Auto-Revert greift automatisch bei fehlerhaftem Update.
