# Scrutiny — surveillance SMART des disques

Surveillance S.M.A.R.T. du pool `tank` : tendances historiques et score de panne (seuils Backblaze). Image **omnibus** unique (web + InfluxDB + collector) sur la **VM 200 TrueNAS** — seul hôte qui voit les disques, le HBA LSI y étant passé en direct.

## Déploiement — Custom App (Apps → Install via YAML)

```yaml
services:
  scrutiny:
    image: ghcr.io/analogj/scrutiny:latest-omnibus
    container_name: scrutiny
    restart: unless-stopped
    cap_add:
      - SYS_RAWIO
    ports:
      - "8080:8080" # web ; l'InfluxDB embarqué reste interne
    volumes:
      - /mnt/tank/apps/scrutiny/config:/opt/scrutiny/config
      - /mnt/tank/apps/scrutiny/influxdb:/opt/scrutiny/influxdb
      - /run/udev:/run/udev:ro
    devices:
      - /dev/disk/by-id/ata-WDC_WD40EFRX-68WT0N0_WD-WCC4EFAKRJS8
      - /dev/disk/by-id/ata-WDC_WD40EFRX-68WT0N0_WD-WCC4EHZ9R6XY
      - /dev/disk/by-id/ata-WDC_WD40EFRX-68N32N0_WD-WCC7K3NR3J7X
    environment:
      - COLLECTOR_CRON_SCHEDULE=0 3 * * *
```

Config et base sur `tank/apps/scrutiny` → snapshots + réplication vers `backup`.

## Collector — déclaration explicite des disques

`smartctl --scan` ne détecte **rien** depuis le conteneur (topologie `/sys` absente). Déclarer les disques à la main, **type `sat`** (SATA derrière HBA LSI).

`/mnt/tank/apps/scrutiny/config/collector.yaml` :

```yaml
devices:
  - device: /dev/disk/by-id/ata-WDC_WD40EFRX-68WT0N0_WD-WCC4EFAKRJS8
    type: sat
  - device: /dev/disk/by-id/ata-WDC_WD40EFRX-68WT0N0_WD-WCC4EHZ9R6XY
    type: sat
  - device: /dev/disk/by-id/ata-WDC_WD40EFRX-68N32N0_WD-WCC7K3NR3J7X
    type: sat
```

## Notifications Discord

Fichier **web** distinct (`scrutiny.yaml`, pas `collector.yaml`). URL au format **Shoutrrr** `discord://TOKEN@ID` — **pas** l'URL brute `https://discord.com/...`. Depuis `.../webhooks/<ID>/<TOKEN>` : le TOKEN passe **avant** le `@`, l'ID numérique après.

`/mnt/tank/apps/scrutiny/config/scrutiny.yaml` :

```yaml
version: 1
notify:
  urls:
    - "discord://TOKEN@ID"
```

TOKEN/ID du webhook → gestionnaire de mots de passe. Notifie uniquement au **passage en échec** d'un disque, pas à chaque collecte.

## Commandes

```bash
# Collecte immédiate (sinon cron quotidien 03h00)
docker exec scrutiny scrutiny-collector-metrics run     # attendu : « Sending 3/3 detected devices »

# Test des notifications (message de test → Discord)
curl -X POST http://localhost:8080/api/health/notify    # attendre ~20 s après un restart

docker restart scrutiny                                 # après toute modif de config
docker logs --tail 40 scrutiny                          # diagnostic
```
