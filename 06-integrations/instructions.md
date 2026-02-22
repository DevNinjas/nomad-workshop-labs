# Lab 05 – Integrationen / Vertiefungsszenario (Vault oder Canary-Deployment)

## Ziel

Je nach verfügbarer Umgebung bearbeiten Sie eines von zwei Vertiefungsszenarien:

- **Variante A (Vault-Integration)**: Secrets aus Vault in einen Nomad-Job einbinden.
- **Variante B (Canary-Deployment)**: Eine neue Version eines Services schrittweise ausrollen.

Der Trainer entscheidet, welche Variante im Kurs durchgeführt wird (oder ob beide als Demo gezeigt werden).

---

## Variante A – Vault-Integration (konzeptionell + Hands-on, falls Umgebung vorhanden)

### Voraussetzungen

- Laufender Nomad-Cluster.
- Vault-Instanz, die für Nomad-Jobs erreichbar ist.
- Basis-Konfiguration der Nomad–Vault-Integration (vom Trainer vorbereitet).
- Eigener Namespace gesetzt (siehe Lab 04, Schritt 0): `export NOMAD_NAMESPACE=<ihr-name>`

> Hinweis: Wenn keine Vault-Umgebung verfügbar ist, kann dieser Teil als reine Demo stattfinden.

### Schritt 1 – Vault-UI öffnen

Vault ist unter **https://vault.nomad.devninjas.io** erreichbar. Der Trainer stellt das Root-Token bereit.

Das Secret `secret/myapp` mit dem Wert `value = "super-secret-value"` ist bereits angelegt.

> **Hinweis:** Wenn keine Vault-Umgebung verfügbar ist, kann dieser Teil als reine Demo stattfinden.

### Schritt 2 – Nomad-Job mit Template vorbereiten

1. Erstellen Sie eine Datei `vault-demo.hcl`:

   ```hcl
   job "vault-demo" {
     datacenters = ["dc1"]
     type        = "service"

     group "vault-group" {
       task "app" {
         driver = "docker"

         vault {
           role = "nomad-workload"
         }

         config {
           image = "alpine:3"
           args  = ["sh", "-c", "cat /secrets/secret.txt && sleep 600"]
         }

         template {
           destination = "secrets/secret.txt"
           change_mode = "noop"

           data = <<EOH
{{ with secret "secret/data/myapp" }}
{{ .Data.data.value }}
{{ end }}
EOH
         }

         resources {
           cpu    = 100
           memory = 64
         }
       }
     }
   }
   ```

2. Passen Sie ggf. den Secret-Pfad und das Datacenter an Ihre Umgebung an.

### Schritt 3 – Job einreichen und prüfen

1. Reichen Sie den Job ein:

   ```bash
   nomad job run vault-demo.hcl
   ```

2. Prüfen Sie den Job-Status:

   ```bash
   nomad job status vault-demo
   ```

3. Ermitteln Sie die Allocation-ID und zeigen Sie die Logs an:

   ```bash
   nomad alloc logs <alloc-id>
   ```

4. Erwartet:
   - In den Logs wird der Secret-Wert aus Vault angezeigt.

### Erfolgskriterien

- Job `vault-demo` läuft erfolgreich.
- Secret aus Vault wird in die Task injiziert und im Log sichtbar.
- Teilnehmer verstehen den grundlegenden Flow von Vault-Integration.

---

## Variante B – Canary-Deployment eines Services

### Voraussetzungen

- Laufender Nomad-Cluster (Dev oder Multi-Node).
- Lab 02 (`http-echo`-Service) erfolgreich durchgeführt oder vergleichbarer Service vorhanden.

### Schritt 1 – Ausgangsservice definieren (Version v1)

1. Verwenden Sie den bestehenden `http-echo`-Job oder erstellen Sie `http-echo-v1.hcl`:

   ```hcl
   job "http-echo" {
     datacenters = ["dc1"]
     type        = "service"

     group "echo-group" {
       count = 3

       network {
         port "http" {
           to = 5678
         }
       }

       task "echo" {
         driver = "docker"

         config {
           image = "hashicorp/http-echo"
           args  = [
             "-text=Hello from v1",
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

2. Reichen Sie den Job ein (falls noch nicht geschehen):

   ```bash
   nomad job run http-echo-v1.hcl
   ```

### Schritt 2 – Canary-Version vorbereiten (Version v2)

1. Kopieren Sie die Job-Datei nach `http-echo-v2.hcl` und passen Sie den Text an:

   ```hcl
   job "http-echo-canary" {
     datacenters = ["dc1"]
     type        = "service"

     group "echo-group" {
       count = 1

       network {
         port "http" {
           to = 5678
         }
       }

       task "echo" {
         driver = "docker"

         config {
           image = "hashicorp/http-echo"
           args  = [
             "-text=Hello from v2 (canary)",
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

2. Reichen Sie die Canary-Version ein:

   ```bash
   nomad job run http-echo-v2.hcl
   ```

### Schritt 3 – Verhalten beobachten und testen

1. Überprüfen Sie beide Jobs in der UI:
   - `http-echo` (v1)
   - `http-echo-canary` (v2)
2. Rufen Sie beide Services mehrfach auf (je nach Konfiguration evtl. unterschiedliche IP/Ports).
3. Beobachten Sie die Antworten:
   - Wird „Hello from v1“ oder „Hello from v2 (canary)“ ausgegeben?

### Schritt 4 – Rollout-Strategie diskutieren

1. Überlegen Sie gemeinsam:
   - Wie könnte man die Canary-Instanz sukzessive hochskalieren?
   - Wie könnte man den v1-Job schrittweise herunterfahren?
2. Diskutieren Sie, wie externe Load Balancer / Service Meshes mit Nomad-Jobs zusammenspielen könnten.

### Erfolgskriterien

- Beide Job-Versionen (v1 und v2) laufen im Cluster.
- Teilnehmer verstehen das Prinzip eines Canary-Deployments.
- Klarheit über mögliche Rollout-Strategien mit Nomad-Jobs.

