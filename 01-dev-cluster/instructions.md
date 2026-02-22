# Lab 01 – Dev-Cluster in der Cloud-Umgebung starten

## Ziel

Am Ende dieses Labs läuft ein Nomad-Dev-Cluster in Ihrer Cloud-Umgebung (Coder). Sie können die Web-UI über Port-Forwarding im Browser öffnen und den Clusterstatus prüfen.

## Voraussetzungen

- Zugang zur Coder-Umgebung (Zugangsdaten vom Trainer erhalten)
- Aktueller Browser (Chrome, Firefox, Edge)
- VS Code – entweder die Web-IDE im Browser oder lokales VS Code mit Remote-SSH-Erweiterung

## Schritt 1 – Mit der Cloud-Umgebung verbinden

1. Öffnen Sie die Coder-Oberfläche im Browser und loggen Sie sich mit Ihren Zugangsdaten ein.
2. Starten Sie Ihren Workspace und öffnen Sie die VS Code IDE (Web-IDE im Browser oder über die Remote-SSH-Erweiterung in Ihrem lokalen VS Code).
3. Öffnen Sie ein Terminal in VS Code (`Terminal → New Terminal` oder `` Ctrl+` ``).
4. Prüfen Sie, dass Nomad vorinstalliert ist:

   ```bash
   nomad version
   ```

   Sie sollten die installierte Nomad-Version sehen (z.B. `Nomad v1.x.x`).

## Schritt 2 – Dev-Agent starten

1. In der Umgebung ist `NOMAD_ADDR` standardmäßig auf den gemeinsamen Lab-Cluster gesetzt. Damit der Dev-Agent lokal angesprochen wird, müssen Sie die Variable zuerst zurücksetzen:

   ```bash
   unset NOMAD_ADDR
   ```

2. Starten Sie Nomad im Dev-Mode:

   ```bash
   nomad agent -dev
   ```

3. Lassen Sie dieses Terminal geöffnet (Logs sollten sichtbar sein).

## Schritt 3 – Port-Forwarding einrichten

Da der Dev-Cluster in der Cloud-Umgebung läuft und nicht auf Ihrem lokalen Rechner, können Sie Adressen wie `localhost:4646` nicht direkt im Browser aufrufen. Über **Port-Forwarding** in VS Code leiten Sie den Port aus der Cloud-Umgebung auf Ihren lokalen Rechner um.

So richten Sie Port-Forwarding ein:

1. Öffnen Sie im VS Code Terminal-Bereich den Tab **Ports** (direkt neben den Tabs „Terminal" und „Problems").
2. Klicken Sie auf **Forward a Port** (oder auf das `+`-Symbol).
3. Geben Sie den Port `4646` ein und bestätigen Sie mit Enter.
4. VS Code zeigt nun eine lokale Adresse an (z.B. `localhost:4646`). Klicken Sie auf das **Globus-Symbol** (Open in Browser) neben dem Eintrag, oder kopieren Sie die Adresse und öffnen Sie sie manuell im Browser.

> **Hinweis:** Port-Forwarding über den Ports-Tab ist nur im Dev-Modus nötig, da der Nomad-Agent hier im selben Container wie Ihre Coder-Umgebung läuft. Der gemeinsame Lab-Cluster in den späteren Labs ist über die Node-IPs bzw. Traefik erreichbar – dort ist kein Port-Forwarding erforderlich.

## Schritt 4 – Web-UI öffnen

1. Stellen Sie sicher, dass Port-Forwarding für Port `4646` aktiv ist (siehe Schritt 3).
2. Öffnen Sie im Browser die weitergeleitete Adresse (z.B. `http://localhost:4646`).
3. Prüfen Sie:
   - Wird der Clusterstatus angezeigt?
   - Sehen Sie mindestens einen aktiven Node?

## Schritt 5 – Status mit CLI prüfen

1. Öffnen Sie ein zweites Terminal in VS Code (`Terminal → New Terminal`).
2. Auch hier müssen Sie zuerst `NOMAD_ADDR` zurücksetzen, damit die CLI den lokalen Dev-Cluster anspricht:

   ```bash
   unset NOMAD_ADDR
   ```

3. Führen Sie aus:

   ```bash
   nomad status
   ```

3. Notieren Sie:
   - Wie heißt Ihr Datacenter?
   - Ist ein Leader vorhanden?

## Erfolgskriterien

- Nomad-Dev-Agent läuft ohne Fehlermeldungen.
- Port-Forwarding für Port `4646` ist eingerichtet.
- UI ist über den weitergeleiteten Port im Browser erreichbar.
- `nomad status` zeigt einen Leader und mindestens einen Node.

## Bonus

- Suchen Sie in den Logs nach Hinweisen, dass der Dev-Mode aktiv ist.
- Stoppen Sie den Agent mit `Ctrl+C` und starten Sie ihn erneut.
- Leiten Sie einen weiteren Port weiter (z.B. wenn Sie später einen Service auf einem anderen Port starten).
