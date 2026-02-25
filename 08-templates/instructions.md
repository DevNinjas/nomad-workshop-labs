# Lab 08 – Templates & Dynamische Konfiguration

## Ziel

Sie nutzen die `template`-Stanza in Nomad, um Konfigurationsdateien dynamisch zu rendern, Environment-Variablen zu setzen und Werte aus Consul KV auszulesen.

## Voraussetzungen

- Nomad-Dev-Agent läuft (lokal) oder Zugang zum Remote-Cluster.
- Docker ist installiert.
- Für Consul-KV-Schritte: Consul muss laufen (Remote-Cluster oder lokaler Consul-Agent).
- Eigener Namespace gesetzt (siehe Lab 04, Schritt 0): `export NOMAD_NAMESPACE=<ihr-name>`

## Hintergrund

Die `template`-Stanza in Nomad nutzt die Go Template-Syntax (ähnlich zu Consul Template). Damit können Sie:

- Konfigurationsdateien für Tasks dynamisch erzeugen.
- Environment-Variablen aus Templates setzen.
- Werte aus Consul KV, Vault oder Nomad-Variablen einbinden.
- Auf Änderungen reagieren (Restart, Signal oder keine Aktion).

## Schritt 1 – Einfaches Template als Datei

1. Erstellen Sie eine Datei `template-file.hcl`:

   ```hcl
job "template-file" {
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

    task "nginx" {
        driver = "docker"

        config {
            image = "nginx:alpine"
            ports = ["http"]
            volumes    = [
                "local/default.conf:/etc/nginx/conf.d/default.conf",
            ]
        }

        template {
        data = <<-EOF
            <!DOCTYPE html>
            <html>
            <body>
            <h1>Nomad Template Demo</h1>
            <p>Job: {{ env "NOMAD_JOB_NAME" }}</p>
            <p>Alloc-ID: {{ env "NOMAD_ALLOC_ID" }}</p>
            <p>Node: {{ env "NOMAD_NODE_NAME" }}</p>
            </body>
            </html>
        EOF

        destination = "local/index.html"
        }

        template {
        data = <<-EOF
            server {
            listen 80;
            location / {
                root /local;
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

   **Warum so?** Die Template-Dateien landen im Allocation-Verzeichnis (`local/index.html`, `local/nginx.conf`). Nginx nutzt standardmäßig `/usr/share/nginx/html` und liest Config nur aus `/etc/nginx/conf.d/`. Damit Nginx Ihre generierte HTML-Datei ausliefert, wird eine eigene `nginx.conf` per Template erzeugt – mit `root {{ env "NOMAD_ALLOC_DIR" }}/local` – und Nginx mit `nginx -c $NOMAD_ALLOC_DIR/local/nginx.conf` gestartet. So lädt Nginx die Config aus dem Allocation-Verzeichnis und liefert die Dateien aus `local/` (u. a. Ihr `index.html`) aus.

2. Reichen Sie den Job ein:

   ```bash
   nomad job run template-file.hcl
   ```

3. Rufen Sie den Service über Traefik auf (ersetzen Sie `<namespace>` durch Ihren Namespace aus Schritt 0):

   ```bash
   curl https://template-file-<namespace>.nomad.devninjas.io
   ```

   Oder öffnen Sie die URL im Browser.

4. Prüfen Sie: Werden die Nomad-Variablen (Job-Name, Alloc-ID, Node-Name) korrekt angezeigt?

## Schritt 2 – Environment-Variablen aus Templates

1. Erstellen Sie eine Datei `template-env.hcl`:

   ```hcl
   job "template-env" {
     datacenters = ["dc1"]
     type        = "batch"

     group "app" {
       task "show-env" {
         driver = "docker"

         config {
           image = "alpine:3"
           args  = ["sh", "-c", "echo APP=$APP_NAME ENV=$APP_ENV VERSION=$APP_VERSION; sleep 10"]
         }

         template {
           data = <<-EOF
             APP_NAME=meine-app
             APP_ENV=production
             APP_VERSION=1.2.3
           EOF

           destination = "secrets/app.env"
           env         = true
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
   nomad job run template-env.hcl
   ```

3. Prüfen Sie die Logs:

   ```bash
   nomad alloc logs <alloc-id>
   ```

4. Erwarteter Output: `APP=meine-app ENV=production VERSION=1.2.3`

## Schritt 3 – Werte aus Consul KV lesen

> Dieser Schritt erfordert einen laufenden Consul-Agent.

1. Schreiben Sie einen Wert in den Consul KV-Store:

   ```bash
   consul kv put app/config/greeting "Hallo aus Consul KV!"
   consul kv put app/config/color "#3498db"
   ```

2. Erstellen Sie eine Datei `template-consul-kv.hcl`:

   ```hcl
   job "template-consul-kv" {
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

      task "nginx" {
        driver = "docker"

        config {
          image   = "nginx:alpine"
          ports   = ["http"]
          command = "sh"
          args    = ["-c", "nginx -c $NOMAD_ALLOC_DIR/local/nginx.conf -g 'daemon off;'"]
        }

         template {
           data = <<-EOF
             <!DOCTYPE html>
             <html>
             <body style="background-color: {{ key "app/config/color" }}; color: white; font-family: sans-serif; text-align: center; padding-top: 100px;">
               <h1>{{ key "app/config/greeting" }}</h1>
               <p>Diese Seite wird dynamisch aus Consul KV generiert.</p>
             </body>
             </html>
           EOF

           destination   = "local/index.html"
           change_mode   = "restart"
           change_signal = ""
         }

         template {
           data = <<-EOF
             events { worker_connections 1024; }
             http {
               server {
                 listen 80;
                 location / {
                   root {{ env "NOMAD_ALLOC_DIR" }}/local;
                   index index.html;
                 }
               }
             }
           EOF

           destination = "local/nginx.conf"
         }

         resources {
           cpu    = 100
           memory = 64
         }
       }
     }
   }
   ```

3. Reichen Sie den Job ein:

   ```bash
   nomad job run template-consul-kv.hcl
   ```

4. Rufen Sie den Service über Traefik auf und prüfen Sie, ob der Greeting-Text und die Hintergrundfarbe aus Consul KV kommen:

   ```bash
   curl https://template-consul-kv-<namespace>.nomad.devninjas.io
   ```

   Oder öffnen Sie die URL im Browser.

## Schritt 4 – Dynamische Aktualisierung beobachten

1. Ändern Sie den Wert in Consul KV:

   ```bash
   consul kv put app/config/greeting "Willkommen zum Nomad Workshop!"
   consul kv put app/config/color "#e74c3c"
   ```

2. Beobachten Sie die Allocation:

   ```bash
   nomad job status template-consul-kv
   ```

3. Da `change_mode = "restart"` konfiguriert ist, sollte die Task automatisch neu gestartet werden.

4. Rufen Sie den Service erneut auf. Der neue Text und die neue Farbe sollten sichtbar sein:

   ```bash
   curl https://template-consul-kv-<namespace>.nomad.devninjas.io
   ```

5. Fragen zur Diskussion:
   - Was passiert bei `change_mode = "noop"`?
   - Wann ist `change_mode = "signal"` sinnvoll?

## Schritt 5 – Aufräumen

```bash
nomad job stop template-file
nomad job stop template-env
nomad job stop template-consul-kv
consul kv delete -recurse app/config/
```

> **Tipp:** Mit `-purge` (z.B. `nomad job stop -purge template-file`) wird der Job vollständig aus Nomad entfernt. Ohne `-purge` bleibt er als „dead" sichtbar, verbraucht aber keine Cluster-Ressourcen.

## Erweiterungen (optional)

- Nutzen Sie `{{ range service "web-app" }}` in einem Template, um eine Liste aller Instanzen eines Consul-Service zu rendern.
- Kombinieren Sie Consul KV und Environment-Variablen: Lesen Sie einen KV-Wert und setzen Sie ihn als `env = true`.

## Erfolgskriterien

- Templates rendern Nomad-Variablen (Job-Name, Alloc-ID) korrekt in Dateien.
- Environment-Variablen werden aus Templates gesetzt und sind im Container verfügbar.
- Werte aus Consul KV werden in Templates eingebunden.
- Bei Änderung eines KV-Wertes wird die Task automatisch neu gestartet.
