# Nomad Workshop – Labs

Praktische Labs für einen HashiCorp-Nomad-Workshop: Lernen durch Übungen – vom Dev-Cluster über Service Discovery bis zum Service Mesh.

## Zielgruppe

Workshop-Teilnehmer und Trainer, die Nomad in einer geführten Umgebung (Coder) nutzen.

## Voraussetzungen

- Zugang zur Coder-Umgebung (Zugangsdaten vom Trainer)
- Aktueller Browser, VS Code (Web-IDE im Browser oder lokales VS Code mit Remote-SSH)
- Ab Lab 02: Docker installiert und lauffähig
- Ab Lab 04: Zugriff auf den gemeinsamen Lab-Cluster, eigener Namespace

## Labs im Überblick

Die Labs sind in numerischer Reihenfolge (01–13) zu bearbeiten. Jedes Lab enthält eine detaillierte Anleitung in `instructions.md`.

| Lab | Thema | Kurzbeschreibung |
| --- | ----- | ---------------- |
| [Lab 01 – Dev-Cluster](01-dev-cluster/instructions.md) | Dev-Cluster in der Cloud starten | Nomad-Dev-Cluster in Coder starten, Web-UI per Port-Forwarding öffnen und Clusterstatus prüfen. |
| [Lab 02 – Einfacher Service-Job mit Docker](02-http-echo-service/instructions.md) | Service-Job mit Docker | Einen einfachen HTTP-Service als Nomad-Job mit Docker deployen und im Browser oder per `curl` aufrufen. |
| [Lab 03 – Batch-Job & Scheduling](03-batch-job/instructions.md) | Batch-Job & Scheduling | Einen Batch-Job erstellen, der einmalig ein Kommando ausführt; Job-Lifecycle und Ressourcenangaben beobachten. |
| [Lab 04 – Multi-Node-Cluster & Failover](04-multinode-failover/instructions.md) | Multi-Node & Failover | Service mit mehreren Instanzen deployen und Failover-Verhalten bei Ausfall einer Allocation beobachten. |
| [Lab 05 – Integrationen](06-integrations/instructions.md) | Vault / Canary | Vertiefung: Vault-Integration (Secrets) oder Canary-Deployment – je nach Umgebung. |
| [Lab 06 – Networking](05-networking/instructions.md) | Bridge vs Host | Unterschiede zwischen Netzwerkmodi `host` und `bridge` verstehen und gezielt einsetzen. |
| [Lab 07 – Service Discovery mit Consul](07-service-discovery/instructions.md) | Service Discovery | Services in Consul registrieren; Health Checks, Registrierung und DNS in Consul UI und API beobachten. |
| [Lab 08 – Templates & dynamische Konfiguration](08-templates/instructions.md) | Templates | `template`-Stanza nutzen: Konfiguration dynamisch rendern, Umgebungsvariablen setzen, Consul KV auslesen. |
| [Lab 09 – Update-Strategien & Deployments](09-update-strategies/instructions.md) | Update-Strategien | Rolling Updates, Canary Deployments und automatische Rollbacks konfigurieren und beobachten. |
| [Lab 10 – Consul Connect (Service Mesh)](10-consul-connect/instructions.md) | Service Mesh | Zwei Services über Consul Connect sicher verbinden; Sidecar-Proxies, Upstreams und Intentions konfigurieren. |
| [Lab 11 – Multi-Task Groups & Lifecycle Hooks](11-multi-task-lifecycle/instructions.md) | Multi-Task & Lifecycle | Task Group mit mehreren Tasks, Lifecycle Hooks für Init/Cleanup, Datenaustausch über das Allocation-Verzeichnis. |
| [Lab 12 – Constraints, Affinities & Scaling](12-constraints-scaling/instructions.md) | Constraints & Scaling | Platzierung mit Constraints/Affinities steuern, Spread-Stanza nutzen und Jobs dynamisch skalieren. |
| [Lab 13 – Aufgaben & Challenges](13-challenges/instructions.md) | Challenges | Offene Aufgaben ohne Schritt-für-Schritt-Anleitung; Ziel und Hinweise, Lösung mit Doku und Erfahrung aus den Labs. |

## So starten Sie

1. **Mit Lab 01 beginnen**: Dev-Cluster in der Coder-Umgebung starten (`nomad agent -dev`), Port-Forwarding für Port `4646` einrichten und die Nomad-Web-UI im Browser öffnen.
2. **Labs der Reihe nach durcharbeiten**: Jedes Lab baut auf den vorherigen auf; die Anleitungen stehen in den verlinkten `instructions.md`.
3. **Ab Lab 04**: Eigenen Namespace anlegen und setzen, z. B. `export NOMAD_NAMESPACE=<ihr-name>`.
4. **Services von außen aufrufen**: Ab Lab 13 wird beschrieben, wie Services über Traefik erreichbar sind (`https://<job-name>-<namespace>.nomad.devninjas.io` bei entsprechender Tag-Konfiguration).

## Weitere Informationen

- **Nomad-Dokumentation**: [developer.hashicorp.com/nomad](https://developer.hashicorp.com/nomad)
- **Sprache**: Alle Lab-Anleitungen sind auf Deutsch.
