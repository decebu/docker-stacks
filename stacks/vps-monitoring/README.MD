# VPS Monitoring Stack

Dieses Verzeichnis enthält den Monitoring-Stack für den VPS. Er dient zur Überwachung der Infrastruktur und zum Versenden von Push-Benachrichtigungen.

## Architektur
- **Caddy**: Reverse Proxy mit TLS (Cloudflare DNS Challenge).
- **ntfy**: Push-Benachrichtigungsdienst mit Authentifizierung.
- **Uptime Kuma**: Überwachung der Erreichbarkeit von Diensten.
- **Dozzle**: Echtzeit-Log-Viewer und Container-Monitoring mit Alarm-Funktion.

## Installation & Start
1. Sicherstellen, dass die Datei ```ntfy_user.db``` existiert:
   ```bash
   touch ntfy_user.db
   ```
2. Stack starten:
   ```bash
   docker compose up -d
   ```

## Benutzerverwaltung (ntfy)
Da ```NTFY_AUTH_DEFAULT_ACCESS=deny-all``` aktiv ist, müssen User manuell angelegt werden.

### Admin anlegen
```bash
docker exec -it ntfy sh -c "NTFY_AUTH_FILE=/etc/ntfy/user.db ntfy user add --role=admin DEIN_USER"
```

### Bot-User für Dozzle/Kuma anlegen
1. User erstellen:
   ```bash
   docker exec -it ntfy sh -c "NTFY_AUTH_FILE=/etc/ntfy/user.db ntfy user add monitoring-bot"
   ```
2. Schreibrechte für Topics vergeben:
   ```bash
   docker exec -it ntfy sh -c "NTFY_AUTH_FILE=/etc/ntfy/user.db ntfy access monitoring-bot "fleet_*" write-only"
   ```

## Alarm-Konfiguration in Dozzle
Dozzle kann über Umgebungsvariablen in der ```docker-compose.yml``` Alarme an ntfy senden:
- ```DOZZLE_REMOTE_AGENT_URL```: ```https://notify.server.decebu.com/fleet_logs```
- Authentifizierung erfolgt über User:Passwort in der URL oder via Header.

## Wichtige URLs
- **ntfy**: ```https://notify.server.decebu.com```
- **Uptime Kuma**: ```http://10.10.10.1:3001``` (VPN)
- **Dozzle**: ```http://10.10.10.1:8080``` (VPN)

## Wartung
- **Logs**: ```docker compose logs -f```
- **Cache**: ntfy bereinigt Nachrichten nach 24h (```NTFY_CACHE_DURATION```).