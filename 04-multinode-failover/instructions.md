# Lab 04 – Multi-Node-Cluster & Failover

## Ziel

Sie deployen einen Service mit mehreren Instanzen auf dem bestehenden Cluster und beobachten das Failover-Verhalten, wenn eine Allocation ausfällt. Der Cluster ist bereits vollständig konfiguriert und läuft.

## Voraussetzungen

- Zugriff auf den Nomad-Cluster (`NOMAD_ADDR` gesetzt)
- Abgeschlossenes Lab 02 (grundlegender Job-Workflow)

## Schritt 0 – Eigenen Namespace erstellen

Da alle Teilnehmer denselben Cluster verwenden, arbeitet jeder in einem **eigenen Namespace** – so entstehen keine Konflikte bei Job-Namen.

1. Erstellen Sie einen Namespace mit Ihrem Namen (z.B. `anna`):

   ```bash
   nomad namespace apply -description "Namespace von anna" anna
   ```

2. Setzen Sie den Namespace als Standard für Ihre Session:

   ```bash
   export NOMAD_NAMESPACE=anna
   ```

3. Prüfen Sie, dass der Namespace existiert:

   ```bash
   nomad namespace list
   ```

> **Wichtig:** Der `export`-Befehl gilt nur für das aktuelle Terminal. Nach einem Neustart muss er wiederholt werden.

## Schritt 1 – Cluster erkunden

1. Zeigen Sie alle verfügbaren Client-Nodes an:

   ```bash
   nomad node status
   ```

2. Sehen Sie sich einen Node genauer an – Ressourcen, Attribute und öffentliche IP:

   ```bash
   nomad node status -verbose <node-id>
   ```

   Unter **Meta** sehen Sie u.a.:
   ```
   public_ip  = <öffentliche IP>
   node_role  = client
   ```

3. Wie viele Nodes sind verfügbar? Auf wie vielen soll der Service verteilt laufen?

## Schritt 2 – HA-Service mit Verteilung deployen

1. Erstellen Sie eine Datei `http-echo-multi.hcl`:

   ```hcl
   job "http-echo-multi" {
     datacenters = ["dc1"]
     type        = "service"

     group "echo-group" {
       count = 3

       # Spread verteilt Allocations anhand von attributes möglichst gleichmäßig auf verschiedene Nodes
       spread {
         attribute = "${node.unique.name}"
       }

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
             "-text=Hello from Nomad multi-node",
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

2. Reichen Sie den Job ein:

   ```bash
   nomad job run http-echo-multi.hcl
   ```

3. Prüfen Sie den Status und die Verteilung:

   ```bash
   nomad job status http-echo-multi
   ```

   Öffnen Sie die Web-UI und beobachten Sie, auf welchen Nodes die 3 Allocations gelandet sind.

4. Rufen Sie eine Instanz auf. Ermitteln Sie dazu Host-Port und öffentliche IP einer Allocation:

   ```bash
   nomad alloc status <alloc-id>
   # → Allocation Addresses: zeigt Host-Port

   nomad node status -verbose <node-id> | grep public_ip
   # → public_ip = <öffentliche IP>
   ```

   ```bash
   curl http://<public-ip>:<host-port>
   ```

## Schritt 3 – Failover simulieren

Sie können eine Allocation gezielt stoppen – Nomad erkennt den Ausfall und plant sie neu ein.

1. Wählen Sie eine laufende Allocation aus:

   ```bash
   nomad job status http-echo-multi
   # → Allocation-ID notieren
   ```

2. Stoppen Sie die Allocation (simuliert einen Task-Absturz):

   ```bash
   nomad alloc stop <alloc-id>
   ```

3. Beobachten Sie sofort den Job-Status:

   ```bash
   nomad job status http-echo-multi
   ```

   Fragen:
   - Wie schnell erkennt Nomad den Ausfall?
   - Wird eine neue Allocation gestartet?
   - Auf welchem Node landet die neue Allocation?

4. Verfolgen Sie die Events der gestoppten Allocation:

   ```bash
   nomad alloc status <alte-alloc-id>
   ```

## Schritt 4 – Node Drain simulieren (optional)

Node Drain simuliert einen geplanten Wartungsfall: Nomad verschiebt alle Allocations auf andere Nodes, bevor der Node offline geht.

1. Wählen Sie einen Node aus, auf dem eine Ihrer Allocations läuft:

   ```bash
   nomad node drain -enable -deadline 30s <node-id>
   ```

2. Beobachten Sie:

   ```bash
   nomad node status
   nomad job status http-echo-multi
   ```

3. Heben Sie den Drain-Modus wieder auf:

   ```bash
   nomad node drain -disable <node-id>
   ```

> **Hinweis:** Node Drain kann den Workshop für alle beeinflussen. Sprechen Sie sich mit dem Trainer ab, bevor Sie diesen Schritt ausführen.

## Schritt 5 – Aufräumen

```bash
nomad job stop http-echo-multi
```

> **Tipp:** Mit `nomad job stop -purge <job>` wird der Job nicht nur gestoppt, sondern auch vollständig aus Nomad entfernt. Ohne `-purge` bleibt der Job als „dead" sichtbar (z.B. in der UI und bei `nomad job status`), verbraucht aber keine Cluster-Ressourcen. `-purge` ist nützlich, um die Job-Liste sauber zu halten.

## Erfolgskriterien

- Service mit `count = 3` läuft verteilt auf verschiedene Nodes (sichtbar in der UI)
- Nach `nomad alloc stop` wird automatisch eine neue Allocation gestartet
- Die Failover-Zeit ist beobachtbar (typisch: unter 30 Sekunden)
