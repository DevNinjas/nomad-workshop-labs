# Lab 03 – Batch-Job & Scheduling-Konzepte

## Ziel

Sie erstellen einen Batch-Job, der einmalig ein Kommando ausführt, und beobachten den Job-Lifecycle sowie Ressourcenangaben.

## Voraussetzungen

- Lab 01 erfolgreich (Nomad-Dev-Agent läuft).
- Docker ist installiert.

## Schritt 1 – Job-Definition anlegen

1. Erstellen Sie eine Datei `batch-example.hcl`.
2. Fügen Sie den folgenden Inhalt ein (Datacenter ggf. anpassen):

   ```hcl
   job "batch-example" {
     datacenters = ["dc1"]
     type        = "batch"

     group "batch-group" {
       count = 1

       task "run-script" {
         driver = "docker"

         config {
           image = "alpine:3"
           args  = ["sh", "-c", "echo 'Batch job running on $(hostname)'; sleep 5"]
         }

         resources {
           cpu    = 10
           memory = 64
         }
       }
     }
   }
   ```

## Schritt 2 – Job einreichen

1. Reichen Sie den Job ein:

   ```bash
   nomad job run batch-example.hcl
   ```

2. Beobachten Sie die Ausgabe.

## Schritt 3 – Job-Lifecycle beobachten

1. Zeigen Sie den Job-Status an:

   ```bash
   nomad job status batch-example
   ```

2. Wiederholen Sie den Befehl nach einigen Sekunden.
3. Fragen:
   - Welchen Status hat der Job direkt nach dem Start?
   - Welchen Status hat er nach Abschluss?

## Schritt 4 – Logs und Allocations prüfen

1. Ermitteln Sie die Allocation-ID, z.B. aus:

   ```bash
   nomad job status batch-example
   ```

2. Anzeigen der Logs:

   ```bash
   nomad alloc logs <alloc-id>
   ```

3. Erwarteter Log-Auszug:
   - `Batch job running on ...`

## Schritt 5 – Constraint hinzufügen (optional)

1. Ergänzen Sie in der `group`-Definition einen Constraint, z.B.:

   ```hcl
   constraint {
     attribute = "${attr.kernel.name}"
     value     = "linux"
   }
   ```

2. Reichen Sie den Job erneut ein (ggf. vorher `nomad job stop`).
3. Beobachten Sie, ob der Job geplant werden kann.

## Erfolgskriterien

- Job „batch-example“ erreicht den Status `complete`.
- Logs zeigen die Ausgabe des Scripts.
- Teilnehmer verstehen den Unterschied zwischen `service`- und `batch`-Jobs.

