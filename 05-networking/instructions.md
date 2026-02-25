# Lab 06 – Networking: Bridge vs Host Mode

## Ziel

Sie verstehen die Unterschiede zwischen den Netzwerkmodi `host` und `bridge` in Nomad und können gezielt entscheiden, welcher Modus für einen Job geeignet ist.

## Voraussetzungen

- Lab 01 erfolgreich abgeschlossen (Nomad-Dev-Agent läuft) oder Zugang zum Remote-Cluster.
- Docker ist installiert und läuft.
- Eigener Namespace gesetzt (siehe Lab 04, Schritt 0): `export NOMAD_NAMESPACE=<ihr-name>`

## Hintergrund

Nomad unterstützt verschiedene Netzwerkmodi für Container-Tasks:

- **`host`**: Der Container teilt sich den Netzwerk-Namespace mit dem Host. Er sieht alle Host-Interfaces und Ports direkt.
- **`bridge`**: Der Container erhält einen eigenen Netzwerk-Namespace (via CNI). Ports müssen explizit gemappt werden.

## Schritt 1 – Job im Host-Modus deployen

1. Erstellen Sie eine Datei `echo-host.hcl`:

   ```hcl
   job "echo-host" {
     datacenters = ["dc1"]
     type        = "service"

     group "echo" {
       count = 1

       network {
         mode = "host"
         port "http" {
           static = 8080
         }
       }

       task "echo" {
         driver = "docker"

         config {
           image        = "hashicorp/http-echo"
           args         = ["-text=Hello from HOST mode", "-listen=:8080"]
           network_mode = "host"
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
   nomad job run echo-host.hcl
   ```

3. Ermitteln Sie die Allocation-ID und den dynamisch zugewiesenen Port:

   ```bash
   nomad job status echo-bridge
   ```

4. Ermitteln Sie die öffentliche IP des Nodes, auf dem der Job läuft:

   ```bash
   nomad alloc status <alloc-id>
   # → Node ID aus der Ausgabe notieren

   nomad node status -verbose <node-id> | grep public_ip
   # → public_ip = <öffentliche IP>
   ```

5. Rufen Sie den Service auf:

   ```bash
   curl http://<public-ip>:8080
   ```

6. Erwarteter Output: `Hello from HOST mode`

## Schritt 2 – Job im Bridge-Modus deployen

1. Erstellen Sie eine Datei `echo-bridge.hcl`:

   ```hcl
   job "echo-bridge" {
     datacenters = ["dc1"]
     type        = "service"

     group "echo" {
       count = 1

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
           args  = ["-text=Hello from BRIDGE mode", "-listen=:5678"]
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
   nomad job run echo-bridge.hcl
   ```

3. Ermitteln Sie die Allocation-ID und den dynamisch zugewiesenen Port:

   ```bash
   nomad job status echo-bridge
   ```

   Notieren Sie die Allocation-ID aus der letzten Zeile (z.B. `e89ff35e`), dann:

   ```bash
   nomad alloc status <alloc-id>
   ```

   Unter **Allocation Addresses** sehen Sie den zugewiesenen Host-Port:

   ```
   Allocation Addresses (mode = "bridge"):
   Label  Dynamic  Address
   http   yes      10.0.1.13:XXXXX -> 5678
   ```

   Die öffentliche IP des Nodes ermitteln Sie über die Node-ID:

   ```bash
   nomad node status -verbose <node-id>
   ```

   Unter **Meta** steht: `public_ip = <öffentliche IP>`

4. Rufen Sie den Service auf:

   ```bash
   curl http://<public-ip>:<host-port>
   ```

5. Erwarteter Output: `Hello from BRIDGE mode`

## Schritt 3 – Unterschiede vergleichen

1. Prüfen Sie die Allocation-Details beider Jobs:

   ```bash
   nomad alloc status <alloc-id-host>
   nomad alloc status <alloc-id-bridge>
   ```

2. Beachten Sie die Unterschiede bei:
   - **Port-Zuweisung**: Host-Mode verwendet den statischen Port direkt, Bridge-Mode mappt einen dynamischen Host-Port auf den Container-Port.
   - **Netzwerk-Isolation**: Im Bridge-Mode hat der Container ein eigenes Netzwerk-Interface.

3. Fragen zur Diskussion:
   - Was passiert, wenn Sie zwei Host-Mode-Jobs mit demselben statischen Port deployen?
   - Warum ist Bridge-Mode für die meisten Workloads zu bevorzugen?

## Schritt 4 – Static Port im Bridge-Modus

1. Erstellen Sie eine Datei `echo-bridge-static.hcl`:

   ```hcl
   job "echo-bridge-static" {
     datacenters = ["dc1"]
     type        = "service"

     group "echo" {
       count = 1

       network {
         mode = "bridge"
         port "http" {
           static = 9090
           to     = 5678
         }
       }

       task "echo" {
         driver = "docker"

         config {
           image = "hashicorp/http-echo"
           args  = ["-text=Hello from BRIDGE with static port", "-listen=:5678"]
         }

         resources {
           cpu    = 100
           memory = 64
         }
       }
     }
   }
   ```

2. Reichen Sie den Job ein und rufen Sie ihn auf:

   ```bash
   nomad job run echo-bridge-static.hcl
   ```

3. Ermitteln Sie die Allocation-ID und den dynamisch zugewiesenen Port:

   ```bash
   nomad job status echo-bridge
   ```

   Notieren Sie die Allocation-ID aus der letzten Zeile (z.B. `e89ff35e`), dann:

   ```bash
   nomad alloc status <alloc-id>
   ```

   Unter **Allocation Addresses** sehen Sie den zugewiesenen Host-Port:

   ```
   Label  Dynamic  Address
   http   yes      10.0.1.13:XXXXX -> 5678
   ```

   Die öffentliche IP des Nodes ermitteln Sie über die Node-ID:

   ```bash
   nomad node status -verbose <node-id>
   ```

   Unter **Meta** steht: `public_ip = <öffentliche IP>`

4. Rufen Sie den Service auf:

   ```bash
   curl http://<public-ip>:<host-port>
   ``` 

4. Erwarteter Output: `Hello from BRIDGE with static port`

## Schritt 5 – Aufräumen

1. Stoppen Sie alle drei Jobs:

   ```bash
   nomad job stop echo-host
   nomad job stop echo-bridge
   nomad job stop echo-bridge-static
   ```

   > **Tipp:** Mit `-purge` (z.B. `nomad job stop -purge echo-host`) wird der Job vollständig aus Nomad entfernt. Ohne `-purge` bleibt er als „dead" sichtbar, verbraucht aber keine Cluster-Ressourcen.

## Erweiterungen (optional)

- Deployen Sie den Bridge-Mode-Job mit `count = 3` und beobachten Sie, welche Ports zugewiesen werden.
- Versuchen Sie, zwei Host-Mode-Jobs mit demselben statischen Port zu starten. Was meldet Nomad?

## Erfolgskriterien

- Sie können Jobs in beiden Netzwerkmodi deployen und aufrufen.
- Sie verstehen den Unterschied zwischen dynamischen und statischen Ports.
- Sie können erklären, wann Host-Mode und wann Bridge-Mode sinnvoll ist.
