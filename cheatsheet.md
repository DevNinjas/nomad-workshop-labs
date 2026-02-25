# CLI Cheat-Sheet – Nomad, Consul, Vault

> Befehle mit **(Workshop)** werden in den Labs dieses Workshops aktiv verwendet.

---

# Nomad

## Umgebungsvariablen

```bash
export NOMAD_ADDR="http://<ip>:4646"        # Nomad-Server-Adresse
export NOMAD_NAMESPACE="<name>"              # Ziel-Namespace (Workshop)
export NOMAD_REGION="<region>"               # Ziel-Region
export NOMAD_TOKEN="<token>"                 # ACL-Token
unset  NOMAD_ADDR                            # Zurücksetzen auf localhost (Workshop)
```

## Agent / Cluster

```bash
nomad version                                # Installierte Version anzeigen (Workshop)
nomad agent -dev                             # Dev-Agent starten (Workshop)
nomad status                                 # Cluster-Status anzeigen (Workshop)
nomad server members                         # Server-Mitglieder auflisten (Workshop)
nomad agent-info                             # Agent-Diagnose-Informationen
```

## Jobs

```bash
# Einreichen und Verwalten
nomad job run <file.hcl>                     # Job einreichen (Workshop)
nomad job run -check-index <idx> <file.hcl>  # Job nur bei unverändertem Index einreichen
nomad job stop <job>                         # Job stoppen (Workshop)
nomad job stop -purge <job>                  # Job stoppen und vollständig entfernen (optional Workshop)

# Status und Inspektion
nomad job status                             # Alle Jobs auflisten (Workshop)
nomad job status <job>                       # Status eines Jobs anzeigen (Workshop)
nomad job inspect <job>                      # Vollständige Job-Definition als JSON
nomad job history <job>                      # Versionshistorie anzeigen (Workshop)

# Planung und Validierung
nomad job plan <file.hcl>                    # Dry-Run / Scheduler-Vorschau
nomad job validate <file.hcl>               # Syntax prüfen (Workshop)
nomad job init                               # Beispiel-Jobspezifikation erzeugen

# Deployment-Steuerung
nomad job promote <job>                      # Canaries befördern (Workshop)
nomad job revert <job> <version>             # Auf frühere Version zurückrollen
nomad job eval <job>                         # Neue Evaluation erzwingen

# Skalierung
nomad job scale <job> <group> <count>        # Task-Group skalieren (Workshop)
nomad job scaling-events <job>               # Skalierungs-Events auflisten

# Parameterisierte und periodische Jobs
nomad job dispatch <job>                     # Parameterisierten Job auslösen (Workshop)
nomad job dispatch -meta key=val <job>       # Mit Metadaten auslösen (Workshop)
nomad job periodic force <job>               # Periodischen Job sofort auslösen
```

## Allocations

```bash
nomad alloc status <alloc-id>                # Allocation-Details anzeigen (Workshop)
nomad alloc logs <alloc-id>                  # Logs der Allocation anzeigen (Workshop)
nomad alloc logs <alloc-id> <task>           # Logs einer bestimmten Task (Workshop)
nomad alloc logs -f <alloc-id>               # Logs live verfolgen (follow)
nomad alloc logs -stderr <alloc-id>          # Stderr-Logs anzeigen
nomad alloc exec <alloc-id> <cmd>            # Befehl in Allocation ausführen
nomad alloc exec -task <task> <alloc> <cmd>  # Befehl in bestimmter Task ausführen
nomad alloc fs <alloc-id> <path>             # Dateisystem der Allocation durchsuchen
nomad alloc stop <alloc-id>                  # Allocation stoppen (Workshop)
nomad alloc restart <alloc-id>               # Allocation neu starten
nomad alloc signal -s SIGTERM <alloc-id>     # Signal an Allocation senden
nomad alloc checks <alloc-id>                # Health-Check-Status anzeigen
```

## Nodes

```bash
nomad node status                            # Alle Nodes auflisten (Workshop)
nomad node status <node-id>                  # Node-Details anzeigen
nomad node status -verbose <node-id>         # Ausführliche Node-Details (Workshop)
nomad node drain -enable <node-id>           # Drain aktivieren (Workshop)
nomad node drain -enable -deadline 5m <id>   # Drain mit Zeitlimit (Workshop)
nomad node drain -disable <node-id>          # Drain deaktivieren (Workshop)
nomad node eligibility -enable <node-id>     # Scheduling-Berechtigung aktivieren
nomad node eligibility -disable <node-id>    # Scheduling-Berechtigung deaktivieren
nomad node meta apply key=val <node-id>      # Node-Metadaten setzen
```

### Öffentliche IP eines Nodes ermitteln (Workshop)

Im Workshop-Cluster sind die Nodes über interne IPs verbunden. Die öffentliche IP jedes Nodes ist als Metadatum `public_ip` hinterlegt. Um einen Service im Browser aufzurufen, benötigen Sie diese IP zusammen mit dem Host-Port der Allocation:

```bash
# 1. Allocation-ID und Node-ID aus dem Job-Status ablesen
nomad job status <job>

# 2. Host-Port aus den Allocation-Details ablesen
nomad alloc status <alloc-id>
#    → Unter "Allocation Addresses":  10.0.1.2X:XXXXX -> <container-port>

# 3. Öffentliche IP des Nodes ermitteln
nomad node status -verbose <node-id> | grep public_ip
#    → public_ip = <öffentliche IP>

# 4. Service aufrufen
curl http://<öffentliche-ip>:<host-port>
```

## Deployments

```bash
nomad deployment list                        # Alle Deployments auflisten (Workshop)
nomad deployment status <deployment-id>      # Deployment-Status anzeigen (Workshop)
nomad deployment promote <deployment-id>     # Canary-Deployment befördern (Workshop)
nomad deployment fail <deployment-id>        # Deployment als fehlgeschlagen markieren (Workshop)
nomad deployment pause <deployment-id>       # Deployment pausieren
nomad deployment resume <deployment-id>      # Deployment fortsetzen
```

## Namespaces

```bash
nomad namespace list                         # Alle Namespaces auflisten (Workshop)
nomad namespace apply -description "..." <n> # Namespace erstellen/aktualisieren (Workshop)
nomad namespace delete <name>                # Namespace löschen
nomad namespace status <name>                # Namespace-Details anzeigen
```

## Service Discovery

```bash
nomad service list                           # Alle registrierten Services auflisten
nomad service info <service>                 # Service-Details anzeigen
```

> **Doku:** https://developer.hashicorp.com/nomad/docs/commands

---

# Consul

## Umgebungsvariablen

```bash
export CONSUL_HTTP_ADDR="http://<ip>:8500"   # Consul-Server-Adresse
export CONSUL_HTTP_TOKEN="<token>"           # ACL-Token
export CONSUL_HTTP_SSL=true                  # HTTPS erzwingen
export CONSUL_CACERT="/path/to/ca.pem"       # CA-Zertifikat für TLS
```

## Agent / Cluster

```bash
consul version                               # Installierte Version anzeigen
consul agent -dev                            # Dev-Agent starten
consul members                               # Cluster-Mitglieder auflisten
consul members -detailed                     # Mitglieder mit Details
consul join <ip>                             # Einem Cluster beitreten
consul leave                                 # Cluster ordnungsgemäß verlassen
consul force-leave <node>                    # Node erzwungen entfernen
consul reload                                # Konfiguration neu laden
consul info                                  # Agent-Diagnose-Informationen
consul monitor                               # Logs live streamen
consul monitor -log-level=debug              # Debug-Logs streamen
```

## Service Catalog

```bash
consul catalog services                      # Alle Services auflisten (Workshop)
consul catalog nodes                         # Alle Nodes auflisten
consul catalog nodes -service=<svc>          # Nodes für einen Service (Workshop)
consul services register <file.json>         # Service registrieren
consul services deregister -id=<id>          # Service deregistrieren
```

## Key-Value Store

```bash
consul kv put <key> <value>                  # Schlüssel-Wert setzen (Workshop)
consul kv get <key>                          # Wert lesen
consul kv get -recurse <prefix>              # Alle Werte unter Prefix lesen
consul kv get -detailed <key>                # Wert mit Metadaten lesen
consul kv delete <key>                       # Schlüssel löschen
consul kv delete -recurse <prefix>           # Alle Schlüssel unter Prefix löschen (Workshop)
consul kv export <prefix>                    # KV-Daten als JSON exportieren
consul kv import @backup.json                # KV-Daten aus JSON importieren
```

## Service Mesh / Intentions

```bash
consul intention list                        # Alle Intentions auflisten
consul intention create -allow <src> <dst>   # Allow-Intention erstellen (Workshop)
consul intention create -deny <src> <dst>    # Deny-Intention erstellen (Workshop)
consul intention delete <src> <dst>          # Intention löschen (Workshop)
consul intention check <src> <dst>           # Verbindung prüfen
consul intention match <svc>                 # Passende Intentions für Service anzeigen
```

## Connect / Envoy

```bash
consul connect proxy -sidecar-for <svc>      # Sidecar-Proxy starten
consul connect envoy -sidecar-for <svc>      # Envoy-Sidecar starten
```

## Snapshots

```bash
consul snapshot save backup.snap             # Snapshot erstellen
consul snapshot restore backup.snap          # Snapshot wiederherstellen
consul snapshot inspect backup.snap          # Snapshot-Metadaten anzeigen
```

## ACL

```bash
consul acl bootstrap                         # Erstes Management-Token erstellen
consul acl token create -policy-name=<p>     # Token erstellen
consul acl token list                        # Alle Tokens auflisten
consul acl token read -id=<id>               # Token-Details anzeigen
consul acl token delete -id=<id>             # Token löschen
consul acl policy create -name=<n> -rules=@f # Policy erstellen
consul acl policy list                       # Alle Policies auflisten
```

## TLS-Zertifikate

```bash
consul tls ca create                         # CA-Zertifikat erzeugen
consul tls cert create -server               # Server-Zertifikat erzeugen
consul tls cert create -client               # Client-Zertifikat erzeugen
```

## Debugging

```bash
consul validate <config-dir>                 # Konfigurationsdateien validieren
consul debug -duration=30s                   # Debug-Archiv aufzeichnen
consul troubleshoot proxy -upstream <svc>    # Mesh-Probleme untersuchen
consul rtt <node-a> <node-b>                 # Netzwerk-Latenz zwischen Nodes messen
```

> **Doku:** https://developer.hashicorp.com/consul/commands

---

# Vault

## Umgebungsvariablen

```bash
export VAULT_ADDR="http://<ip>:8200"         # Vault-Server-Adresse
export VAULT_TOKEN="<token>"                 # Authentifizierungs-Token
export VAULT_NAMESPACE="<ns>"                # Ziel-Namespace (Enterprise)
export VAULT_FORMAT="json"                   # Ausgabeformat (table|json|yaml)
export VAULT_SKIP_VERIFY=1                   # TLS-Prüfung deaktivieren (nur Dev!)
```

## Server / Operator

```bash
vault version                                # Installierte Version anzeigen
vault status                                 # Server-Status und Seal-Status
vault operator init                          # Vault initialisieren
vault operator init -key-shares=3 -key-threshold=2  # Mit Shamir-Parametern
vault operator unseal <key>                  # Vault entsiegeln (je Unseal-Key)
vault operator seal                          # Vault siegeln
vault operator key-status                    # Verschlüsselungskey-Info
vault operator step-down                     # Leader-Rolle abgeben (HA)
vault operator raft list-peers               # Raft-Cluster-Mitglieder anzeigen
vault operator raft snapshot save bk.snap    # Raft-Snapshot erstellen
vault operator raft snapshot restore bk.snap # Raft-Snapshot wiederherstellen
```

## Authentifizierung

```bash
vault login                                  # Interaktiv anmelden
vault login <token>                          # Mit Token anmelden
vault login -method=userpass username=<u>    # Mit Userpass anmelden
vault login -method=ldap username=<u>        # Mit LDAP anmelden
vault token create                           # Neues Token erstellen
vault token create -policy=<p> -ttl=1h       # Token mit Policy und TTL
vault token lookup                           # Eigenes Token inspizieren
vault token lookup <token>                   # Fremdes Token inspizieren
vault token renew <token>                    # Token verlängern
vault token revoke <token>                   # Token widerrufen
vault token revoke -mode=orphan <token>      # Token ohne Kind-Tokens widerrufen
```

## Auth-Methoden verwalten

```bash
vault auth list                              # Aktive Auth-Methoden auflisten
vault auth enable <method>                   # Auth-Methode aktivieren
vault auth enable -path=<p> userpass         # Auth-Methode an Pfad aktivieren
vault auth disable <path>                    # Auth-Methode deaktivieren
vault auth tune -default-lease-ttl=1h <path> # Auth-Methode tunen
```

## Secrets Engines

```bash
vault secrets list                           # Aktive Engines auflisten
vault secrets enable <engine>                # Engine aktivieren
vault secrets enable -path=<p> kv-v2         # Engine an Pfad aktivieren
vault secrets disable <path>                 # Engine deaktivieren
vault secrets tune -max-lease-ttl=24h <path> # Engine tunen
vault secrets move <old-path> <new-path>     # Engine verschieben
```

## KV Secrets (Version 2)

```bash
vault kv put -mount=secret <path> key=val    # Secret schreiben
vault kv put -mount=secret <path> @data.json # Secret aus Datei schreiben
vault kv get -mount=secret <path>            # Secret lesen
vault kv get -mount=secret -version=2 <path> # Bestimmte Version lesen
vault kv get -mount=secret -field=key <path> # Einzelnes Feld lesen
vault kv list -mount=secret <path>           # Secrets unter Pfad auflisten
vault kv delete -mount=secret <path>         # Aktuelle Version löschen
vault kv destroy -mount=secret -versions=1,2 <path>  # Versionen unwiderruflich löschen
vault kv undelete -mount=secret -versions=1 <path>   # Soft-Delete rückgängig
vault kv metadata get -mount=secret <path>   # Metadaten anzeigen
vault kv metadata delete -mount=secret <path> # Secret vollständig entfernen
```

## Policies

```bash
vault policy list                            # Alle Policies auflisten
vault policy read <name>                     # Policy anzeigen
vault policy write <name> <file.hcl>         # Policy erstellen/aktualisieren
vault policy delete <name>                   # Policy löschen
vault policy fmt <file.hcl>                  # Policy-Datei formatieren
```

## Generische API-Befehle

```bash
vault read <path>                            # GET-Request an API-Pfad
vault write <path> key=val                   # PUT/POST-Request an API-Pfad
vault delete <path>                          # DELETE-Request an API-Pfad
vault list <path>                            # LIST-Request an API-Pfad
```

## PKI / Zertifikate

```bash
vault write pki/root/generate/internal \
  common_name="Root CA" ttl=87600h           # Root-CA erzeugen
vault write pki/issue/<role> \
  common_name="example.com"                  # Zertifikat ausstellen
vault write pki/roles/<role> \
  allowed_domains="example.com"              # PKI-Rolle erstellen
```

## Debugging

```bash
vault audit enable file file_path=/var/log/vault.log  # Audit-Log aktivieren
vault audit list                             # Audit-Backends auflisten
vault path-help <path>                       # Hilfe zu einem API-Pfad
vault debug -duration=30s                    # Debug-Bundle erstellen
```

> **Doku:** https://developer.hashicorp.com/vault/docs/commands
