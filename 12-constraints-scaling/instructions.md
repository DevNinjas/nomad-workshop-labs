# Lab 12 – Constraints, Affinities & Job-Scaling

## Ziel

Sie steuern die Platzierung von Nomad-Jobs auf bestimmten Nodes mit Constraints und Affinities, verteilen Workloads gleichmäßig mit der Spread-Stanza und skalieren Jobs dynamisch.

## Voraussetzungen

- Zugang zum Remote-Cluster (mindestens 2 Client-Nodes).
- Docker ist installiert.
- Eigener Namespace gesetzt (siehe Lab 04, Schritt 0): `export NOMAD_NAMESPACE=<ihr-name>`

## Hintergrund

Nomad bietet drei Mechanismen zur Steuerung der Job-Platzierung:

- **`constraint`**: Harte Regel – der Job läuft **nur** auf Nodes, die die Bedingung erfüllen. Wenn kein Node passt, bleibt der Job unplaced.
- **`affinity`**: Weiche Regel – Nomad **bevorzugt** passende Nodes, weicht aber bei Bedarf auf andere aus.
- **`spread`**: Verteilt Allocations gleichmäßig über einen Attributwert (z.B. Datacenter, Node-ID).

## Schritt 1 – Node-Attribute erkunden

1. Listen Sie die verfügbaren Nodes auf:

   ```bash
   nomad node status
   ```

2. Wählen Sie einen Node und zeigen Sie seine Details an:

   ```bash
   nomad node status -verbose <node-id>
   ```

3. Notieren Sie sich einige Attribute, z.B.:
   - `${attr.kernel.name}` (z.B. `linux`)
   - `${attr.cpu.arch}` (z.B. `arm64`)
   - `${attr.os.name}` (z.B. `ubuntu`)
   - `${node.unique.id}`
   - `${node.unique.name}`

## Schritt 2 – Constraint: Job auf bestimmte Nodes beschränken

1. Erstellen Sie eine Datei `constraint-demo.hcl`:

   ```hcl
   job "constraint-demo" {
     datacenters = ["dc1"]
     type        = "service"

     group "web" {
       count = 2

       constraint {
         attribute = "${attr.kernel.name}"
         value     = "linux"
       }

       network {
         mode = "bridge"
         port "http" {
           to = 5678
         }
       }

       task "echo" {
         driver = "docker"

         config {
           image = "hashicorp/http-echo"
           args  = ["-text=Constraint: nur Linux", "-listen=:5678"]
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
   nomad job run constraint-demo.hcl
   ```

3. Prüfen Sie, auf welchen Nodes die Allocations laufen:

   ```bash
   nomad job status constraint-demo
   ```

4. Experimentieren Sie: Ändern Sie den Constraint auf einen Wert, der keinem Node entspricht:

   ```hcl
   constraint {
     attribute = "${attr.kernel.name}"
     value     = "windows"
   }
   ```

5. Reichen Sie den Job ein und beobachten Sie die Placement-Failure-Meldung:

   ```bash
   nomad job status constraint-demo
   ```

## Schritt 3 – Affinity: Weiche Platzierungspräferenz

1. Erstellen Sie eine Datei `affinity-demo.hcl`:

   ```hcl
   job "affinity-demo" {
     datacenters = ["dc1"]
     type        = "service"

     group "web" {
       count = 2

       affinity {
         attribute = "${node.unique.name}"
         value     = "nomad-client-0"
         weight    = 80
       }

       network {
         mode = "bridge"
         port "http" {
           to = 5678
         }
       }

       task "echo" {
         driver = "docker"

         config {
           image = "hashicorp/http-echo"
           args  = ["-text=Affinity Demo", "-listen=:5678"]
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
   nomad job run affinity-demo.hcl
   ```

3. Prüfen Sie die Platzierung:

   ```bash
   nomad job status affinity-demo
   ```

4. Fragen:
   - Laufen beide Allocations auf `nomad-client-0`?
   - Was passiert bei `weight = 100` vs `weight = 50`?
   - Was ist der Unterschied zu einem Constraint?

## Schritt 4 – Spread: Gleichmäßige Verteilung

1. Erstellen Sie eine Datei `spread-demo.hcl`:

   ```hcl
   job "spread-demo" {
     datacenters = ["dc1"]
     type        = "service"

     group "web" {
       count = 4

       spread {
         attribute = "${node.unique.id}"
       }

       network {
         mode = "bridge"
         port "http" {
           to = 5678
         }
       }

       task "echo" {
         driver = "docker"

         config {
           image = "hashicorp/http-echo"
           args  = ["-text=Spread Demo", "-listen=:5678"]
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
   nomad job run spread-demo.hcl
   ```

3. Prüfen Sie die Verteilung der Allocations:

   ```bash
   nomad job status spread-demo
   ```

4. Bei 4 Allocations und 2 Clients sollten jeweils 2 Allocations pro Node laufen.

5. Vergleichen Sie: Entfernen Sie die `spread`-Stanza und reichen Sie den Job erneut ein. Ändert sich die Verteilung?

## Schritt 5 – Job-Scaling

1. Ändern Sie die Anzahl der Instanzen per CLI:

   ```bash
   nomad job scale spread-demo web 6
   ```

2. Prüfen Sie den Status:

   ```bash
   nomad job status spread-demo
   ```

3. Es sollten jetzt 6 Allocations laufen, gleichmäßig verteilt.

4. Skalieren Sie herunter:

   ```bash
   nomad job scale spread-demo web 2
   ```

5. Beobachten Sie, wie die überzähligen Allocations gestoppt werden.

## Schritt 6 – Aufräumen

```bash
nomad job stop constraint-demo
nomad job stop affinity-demo
nomad job stop spread-demo
```

> **Tipp:** Mit `-purge` (z.B. `nomad job stop -purge constraint-demo`) wird der Job vollständig aus Nomad entfernt. Ohne `-purge` bleibt er als „dead" sichtbar, verbraucht aber keine Cluster-Ressourcen.

## Erweiterungen (optional)

- Kombinieren Sie Constraint und Affinity in derselben Group.
- Nutzen Sie `distinct_hosts` als Constraint, um sicherzustellen, dass jede Allocation auf einem anderen Node läuft:

  ```hcl
  constraint {
    operator = "distinct_hosts"
    value    = "true"
  }
  ```

- Verwenden Sie Meta-Attribute: Setzen Sie auf einem Node `meta { role = "gpu" }` und schreiben Sie einen Constraint dagegen.

## Erfolgskriterien

- Constraints verhindern die Platzierung auf unpassenden Nodes.
- Affinities bevorzugen bestimmte Nodes, sind aber flexibel.
- Spread verteilt Allocations gleichmäßig über Nodes.
- `nomad job scale` ändert die Instanzanzahl dynamisch.
