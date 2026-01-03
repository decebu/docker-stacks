# Unbound DNS‑Server Docker‑Stack

Dieses Verzeichnis enthält die Docker‑Stack‑Konfiguration für
**Unbound**, einen rekursiven, validierenden und zwischenspeichernden
DNS‑Server.\
Das Setup wurde nach intensiver Fehlersuche optimiert und folgt nun
einer finalen, robusten und portablen Architektur für sichere
DNS‑Auflösung im Heimnetz.

------------------------------------------------------------------------

## ✨ Funktionen

-   **Rekursiver DNS‑Resolver** -- löst DNS‑Abfragen direkt bei den
    autoritativen Root‑Servern auf.
-   **DNSSEC‑Validierung** -- schützt vor DNS‑Manipulation und stellt
    die Authentizität der Antworten sicher.
-   **DNS‑over‑TLS (DoT) Forwarding** -- optionale Weiterleitung an
    externe Resolver wie *NextDNS* für verschlüsselte Auflösung.
-   **Lokales DNS‑Caching** -- beschleunigt wiederholte DNS‑Abfragen
    drastisch.
-   **Security‑Hardening** durch `chroot` und bewährte
    Unbound‑Konfigurationsoptionen.
-   **Portable One‑Folder Architektur** -- einfach per Git versionierbar
    & ohne versteckte Pfad‑Komplexität.

------------------------------------------------------------------------

## 🧠 Architektur -- Die „Ein‑Ordner‑Philosophie"

Dieses Setup basiert auf folgenden Erkenntnissen:

1.  **Ein zentraler Konfigurations‑Ordner (`./config`)**\
    → enthält alle manuell gepflegten Dateien flach (z. B.
    `unbound.conf`, `a‑records.conf`, `root.hints`,
    `forward‑records.conf`).

2.  **Nur ein Volume‑Mount im Docker‑Stack**\
    → `./config` wird direkt nach `/opt/unbound/etc/unbound` im
    Container gemappt.

3.  **Intelligentes Start‑Script (im Image enthalten)**\
    → übernimmt automatisch:

    -   Erstellung des `root.key` (DNSSEC Trust Anchor)
    -   Erkennung des aktivierten `chroot` und Kopieren des
        `ca‑certificates.crt` in das chroot‑Gefängnis
    -   Sicherstellung, dass **DoT trotz chroot funktioniert**

Vorteile: - Maximal **portabel** - Sehr **robust** -
**Debug‑freundlich** - **Keine verschachtelten Pfad‑Mappings** - Ideal
für **GitOps in deinem Home‑Lab** (z. B. später auch im K3s‑Cluster
einsetzbar)

------------------------------------------------------------------------

## 📦 Voraussetzungen

-   Installierter **Docker Engine**
-   Installiertes **Docker Compose Plugin**
-   Manuell bereitgestellte `root.hints` (einmaliger Download)
-   Eigene Anpassung der Dateien:
    -   `unbound.conf`
    -   `a‑records.conf`
    -   `forward‑records.conf`

------------------------------------------------------------------------

## 🚀 Inbetriebnahme

``` bash
# Repository klonen
git clone <repo_url>
cd stacks/unbound

# Konfig‑Ordner erstellen (falls noch nicht vorhanden)
mkdir ‑p config

# root.hints einmalig herunterladen
wget ‑O ./config/root.hints https://www.internic.net/domain/named.root

# Eigene Konfigurationen in den config‑Ordner legen
# z. B.:
# cp unbound.conf ./config/
# cp a‑records.conf ./config/
# cp forward‑records.conf ./config/

# Stack starten
docker compose up ‑d
```

------------------------------------------------------------------------

## ⚙️ Konfiguration

### `docker‑compose.yml`

``` yaml
version: '3.8'

services:
  unbound:
    image: mvance/unbound:latest
    container_name: unbound
    restart: unless‑stopped
    ports:
      ‑ "53:53/tcp"
      ‑ "53:53/udp"
    volumes:
      ‑ ./config:/opt/unbound/etc/unbound
    environment:
      ‑ TZ=Europe/Berlin
```

### `unbound.conf` (Beispiel)

``` ini
server:
    directory: "/opt/unbound/etc/unbound"
    chroot: "/opt/unbound/etc/unbound"

    logfile: ""
    use‑syslog: no
    interface: 0.0.0.0

    tls‑cert‑bundle: "ca‑certificates.crt"
    root‑hints: "root.hints"
    auto‑trust‑anchor‑file: "root.key"

    include: "a‑records.conf"
    include: "forward‑records.conf"
```

------------------------------------------------------------------------

## 🧪 Validierung & Tests

``` bash
# DNSSEC‑Validierung – muss NOERROR + ad‑Flag zeigen
dig @<host_ip> sigok.verteiltesysteme.net

# DNSSEC‑Test – muss SERVFAIL liefern oder timeout
dig @<host_ip> sigfail.verteiltesysteme.net

# Prüfen der Root‑Hints‑Nutzung – liefert Root‑NS‑Liste
dig @<host_ip> . NS

# Test lokaler A‑Records
dig @<host_ip> ha.home.decebu.com
```

------------------------------------------------------------------------

## 🌿 Git Versionskontrolle

Damit der Stack Git‑freundlich bleibt, sollten **Laufzeit‑Artefakte
ignoriert werden**:

### `.gitignore`

    # Unbound Laufzeit‑Artefakte
    /stacks/unbound/config/root.key
    /stacks/unbound/config/unbound.pid
    /stacks/unbound/config/ca‑certificates.crt
    /stacks/unbound/config/dev/
    /stacks/unbound/config/var/

> **Wichtig:** `root.hints` wird **nicht ignoriert**, da sie bewusst
> manuell verwaltet wird.

------------------------------------------------------------------------

## 🔒 Sicherheitshinweise

-   Der Container läuft im `chroot`‑Modus → **keine absoluten Pfade
    außerhalb von `/opt/unbound/etc/unbound`**

-   Logging erfolgt über `stdout` → ideal für `docker logs ‑f unbound`

-   Keine Shell im Container nötig -- **alles liegt im config‑Ordner**

-   Update‑fähig per:

    ``` bash
    docker compose pull && docker compose restart
    ```
