# Stack \*arr — CT 208

Arr stack allows to downloading legal digital media from already owned medias

`192.168.1.28` · pve-nas · LXC Debian 13 non privilégié 4 vCPU / 6 Go / 12 Go disque · Docker Compose dans `/opt/arr`

Adapté de [TechHutTV/homelab](https://github.com/TechHutTV/homelab/tree/main/media).

## Services

| Service      | Port | Réseau         | Rôle                                  |
| ------------ | ---- | -------------- | ------------------------------------- |
| gluetun      | —    | —              | Tunnel VPN AirVPN + kill-switch       |
| qbittorrent  | 8080 | via gluetun    | Client torrent                        |
| prowlarr     | 9696 | via gluetun    | Gestion des indexeurs                 |
| flaresolverr | 8191 | via gluetun    | Contourne les protections Cloudflare  |
| sonarr       | 8989 | servarrnetwork | Séries                                |
| radarr       | 7878 | servarrnetwork | Films                                 |
| bazarr       | 6767 | servarrnetwork | Sous-titres                           |
| seerr        | 5055 | servarrnetwork | Interface de demandes                 |
| deunhealth   | —    | aucun          | Redémarre qBittorrent si le VPN coupe |

Jellyfin tourne séparément en CT 204 (`192.168.1.24`) pour bénéficier du passthrough GPU.

## Adressage interne — la règle qui compte

Deux conteneurs partageant `network_mode: service:gluetun` se voient en **localhost**. Tout conteneur extérieur passe par l'**IP de gluetun**.

| Depuis                  | Vers qBittorrent  | Vers Prowlarr     | Vers FlareSolverr |
| ----------------------- | ----------------- | ----------------- | ----------------- |
| Prowlarr                | `localhost:8080`  | —                 | `localhost:8191`  |
| Sonarr / Radarr / Seerr | `172.39.0.2:8080` | `172.39.0.2:9696` | —                 |

Adresses sur `servarrnetwork` :

```
gluetun (+ qbittorrent, prowlarr, flaresolverr) → 172.39.0.2
sonarr → 172.39.0.3
radarr → 172.39.0.4
bazarr → 172.39.0.6
seerr  → 172.39.0.7
```

C'est la source d'erreur numéro un sur cette pile.

---

## 1. Stockage

Principe : **un seul point de montage** `**/data**`, sinon les liens durs ne fonctionnent pas et chaque import devient une copie complète.

Bind mount depuis pve-nas, qui monte lui-même le NFS :

```
pct set 208 -mp0 /mnt/tank-media,mp=/data
```

Structure dans `tank/media` :

```
tank/media
├── downloads
│   └── qbittorrent
│       ├── completed
│       ├── incomplete
│       └── torrents
├── movies
└── shows
```

**Vérifier avant toute écriture** que le NFS est bien monté, sinon tout atterrit sur le disque système de l'hôte :

```
df -h /mnt/tank-media          # sur pve-nas
pct exec 208 -- df -h /data    # dans le conteneur
```

Doit afficher `192.168.1.20:/mnt/tank/media`, jamais `/dev/mapper/pve-root`.

### Permissions

Le dataset `tank/media` porte une ACL NFSv4 (héritée du partage SMB). Un `chmod` ne suffit pas. L'UID 1000 du conteneur non privilégié devient 101000 côté TrueNAS, qui ne le connaît pas d'où l'entrée `everyone@` :

```
midclt call filesystem.setacl '{
  "path": "/mnt/tank/media",
  "uid": 0,
  "gid": 0,
  "dacl": [
    {"tag":"owner@","type":"ALLOW","perms":{"BASIC":"FULL_CONTROL"},"flags":{"BASIC":"INHERIT"}},
    {"tag":"group@","type":"ALLOW","perms":{"BASIC":"MODIFY"},"flags":{"BASIC":"INHERIT"}},
    {"tag":"everyone@","type":"ALLOW","perms":{"BASIC":"MODIFY"},"flags":{"BASIC":"INHERIT"}}
  ],
  "options": {"recursive": true, "traverse": true}
}'
```

---

## 2. Conteneur

```
pct create 208 local:vztmpl/debian-13-standard_13.6-1_amd64.tar.zst \
  --hostname arr \
  --cores 4 --memory 6144 --rootfs local-lvm:12 \
  --net0 name=eth0,bridge=vmbr0,ip=192.168.1.28/24,gw=192.168.1.254 \
  --nameserver "192.168.1.11 1.1.1.1" \
  --features nesting=1,keyctl=1 \
  --mp0 /mnt/tank-media,mp=/data \
  --onboot 1 --unprivileged 1 \
  --startup order=4,up=30
```

### Périphérique TUN

gluetun échoue en LXC sans cela (`cannot create TUN device file node: operation not permitted`) :

```
cat >> /etc/pve/lxc/208.conf << 'EOF'
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.mount.entry: /dev/net dev/net none bind,create=dir
lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
EOF
```

### Docker

```
apt update && apt install -y curl ca-certificates
curl -fsSL https://get.docker.com | sh

mkdir -p /etc/systemd/system/docker.service.d
printf '[Unit]\nRequiresMountsFor=/data\n' > /etc/systemd/system/docker.service.d/mount-dep.conf
systemctl daemon-reload
```

Créer les dossiers de configuration **avec le bon propriétaire avant le premier lancement**, sinon plusieurs conteneurs bouclent sur des erreurs `EACCES` :

```
mkdir -p /opt/arr/{gluetun,qbittorrent,prowlarr,sonarr,radarr,bazarr,seerr}
chown -R 1000:1000 /opt/arr/{qbittorrent,prowlarr,sonarr,radarr,bazarr,seerr}
```

---

## 3. VPN — AirVPN

### Génération de la configuration

**Client Area → Config Generator** : Linux, WireGuard, New device, By single server. Choisir un serveur peu chargé.

**Client Area → Ports** : ajouter un port, vérifier avec **Test open** qu'il n'affiche pas `Connection refused` sur le serveur retenu.

Port en service : **4314**, TCP+UDP, tous appareils.

**Ne jamais rediriger ce port sur la Box.**

### Correspondance `.conf` → `.env`

| Fichier `.conf`          | Variable                  |
| ------------------------ | ------------------------- |
| `[Interface] Address`    | `WIREGUARD_ADDRESSES`     |
| `[Interface] PrivateKey` | `WIREGUARD_PRIVATE_KEY`   |
| `[Peer] PublicKey`       | `WIREGUARD_PUBLIC_KEY`    |
| `[Peer] PresharedKey`    | `WIREGUARD_PRESHARED_KEY` |

`WIREGUARD_PUBLIC_KEY` prend la clé **du serveur**, sous `[Peer]` — pas la tienne. Erreur classique.

`MTU`, `DNS`, `Endpoint`, `AllowedIPs` et `PersistentKeepalive` ne se reportent pas : gluetun les gère.

**Retirer la partie IPv6 de** `**Address**`**.** Le LXC n'a pas d'IPv6 et gluetun refuse de démarrer :

```
ERROR VPN settings: interface address is IPv6 but IPv6 is not supported
```

### `.env`

```
TZ=Europe/Paris
PUID=1000
PGID=1000

VPN_SERVICE_PROVIDER=airvpn
VPN_TYPE=wireguard

FIREWALL_VPN_INPUT_PORTS=4314

WIREGUARD_PUBLIC_KEY=<clé du serveur>
WIREGUARD_PRIVATE_KEY=<clé privée>
WIREGUARD_PRESHARED_KEY=<clé partagée>
WIREGUARD_ADDRESSES=10.x.x.x/32

SERVER_COUNTRIES=Netherlands
SERVER_CITIES=Alblasserdam

HEALTH_VPN_DURATION_INITIAL=120s

SET_IP_GLUETUN=172.39.0.2
SET_IP_SONARR=172.39.0.3
SET_IP_RADARR=172.39.0.4
SET_IP_BAZARR=172.39.0.6
SET_IP_SEERR=172.39.0.7
```

`chmod 600 .env`

`**SERVER_COUNTRIES**` **et** `**SERVER_CITIES**` **sont nécessaires ici.** Sans elles, gluetun choisit un serveur AirVPN au hasard en ignorant l'endpoint du `.conf` — la connexion réussit (les clés valent sur toute l'infrastructure) mais le port redirigé ne correspond plus au serveur utilisé. Constaté : configuration néerlandaise

---

## 4. `compose.yaml`

```
networks:
  servarrnetwork:
    name: servarrnetwork
    ipam:
      config:
        - subnet: 172.39.0.0/24

services:
  portainer-agent:
    image: portainer/agent:latest
    container_name: portainer-agent
    restart: unless-stopped
    ports:
      - 9001:9001
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /var/lib/docker/volumes:/var/lib/docker/volumes
    networks:
      servarrnetwork:
        ipv4_address: 172.39.0.8
  gluetun:
    image: qmcgaw/gluetun
    container_name: gluetun
    cap_add:
      - NET_ADMIN
    devices:
      - /dev/net/tun:/dev/net/tun
    networks:
      servarrnetwork:
        ipv4_address: ${SET_IP_GLUETUN}
    ports:
      - 8080:8080   # qbittorrent
      - 4314:4314   # port torrent AirVPN
      - 9696:9696   # prowlarr
      - 8191:8191   # flaresolverr
    volumes:
      - ./gluetun:/gluetun
    env_file:
      - .env
    environment:
      - BLOCK_MALICIOUS=off
    healthcheck:
      test: ping -c 1 www.google.com || exit 1
      interval: 20s
      timeout: 10s
      retries: 5
    restart: unless-stopped

  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    restart: unless-stopped
    labels:
      - deunhealth.restart.on.unhealthy=true
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
      - WEBUI_PORT=8080
      - TORRENTING_PORT=${FIREWALL_VPN_INPUT_PORTS}
    volumes:
      - ./qbittorrent:/config
      - /data:/data
    depends_on:
      gluetun:
        condition: service_healthy
        restart: true
    network_mode: service:gluetun
    healthcheck:
      test: ping -c 1 www.google.com || exit 1
      interval: 60s
      retries: 3
      start_period: 20s
      timeout: 10s

  deunhealth:
    image: qmcgaw/deunhealth
    container_name: deunhealth
    network_mode: "none"
    environment:
      - LOG_LEVEL=info
      - HEALTH_SERVER_ADDRESS=127.0.0.1:9999
      - TZ=${TZ}
    restart: always
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock

  prowlarr:
    image: lscr.io/linuxserver/prowlarr:latest
    container_name: prowlarr
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
    volumes:
      - ./prowlarr:/config
    restart: unless-stopped
    depends_on:
      gluetun:
        condition: service_healthy
        restart: true
    network_mode: service:gluetun

  flaresolverr:
    image: ghcr.io/flaresolverr/flaresolverr:latest
    container_name: flaresolverr
    environment:
      - LOG_LEVEL=info
      - LOG_HTML=false
      - CAPTCHA_SOLVER=none
      - TZ=${TZ}
    depends_on:
      gluetun:
        condition: service_healthy
        restart: true
    network_mode: service:gluetun
    restart: unless-stopped

  sonarr:
    image: lscr.io/linuxserver/sonarr:latest
    container_name: sonarr
    restart: unless-stopped
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
    volumes:
      - ./sonarr:/config
      - /data:/data
    ports:
      - 8989:8989
    networks:
      servarrnetwork:
        ipv4_address: ${SET_IP_SONARR}

  radarr:
    image: lscr.io/linuxserver/radarr:latest
    container_name: radarr
    restart: unless-stopped
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
    volumes:
      - ./radarr:/config
      - /data:/data
    ports:
      - 7878:7878
    networks:
      servarrnetwork:
        ipv4_address: ${SET_IP_RADARR}

  bazarr:
    image: lscr.io/linuxserver/bazarr:latest
    container_name: bazarr
    restart: unless-stopped
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
    volumes:
      - ./bazarr:/config
      - /data:/data
    ports:
      - 6767:6767
    networks:
      servarrnetwork:
        ipv4_address: ${SET_IP_BAZARR}

  seerr:
    image: ghcr.io/seerr-team/seerr:latest
    container_name: seerr
    restart: unless-stopped
    environment:
      - LOG_LEVEL=debug
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
    volumes:
      - ./seerr:/app/config
    ports:
      - 5055:5055
    healthcheck:
      test: wget --no-verbose --tries=1 --spider http://localhost:5055/api/v1/status || exit 1
      start_period: 20s
      timeout: 3s
      interval: 15s
      retries: 3
    networks:
      servarrnetwork:
        ipv4_address: ${SET_IP_SEERR}
```

### Vérifier le tunnel

**À faire avant tout téléchargement :**

```
docker run --rm --network=container:gluetun alpine:3.18 sh -c "apk add wget && wget -qO- https://ipinfo.io"
```

L'IP doit être celle d'AirVPN et la ville correspondre au serveur choisi.

---

## 5. Configuration des applications

### qBittorrent

Mot de passe temporaire : `docker logs qbittorrent | grep -i password`

_Settings → Downloads_ :

- Default Save Path : `/data/downloads/qbittorrent/completed`
- Incomplete : `/data/downloads/qbittorrent/incomplete`
- Torrent files : `/data/downloads/qbittorrent/torrents`

_Settings → WebUI_ : identifiants personnels + **Bypass authentication for clients on localhost**

_Settings → Advanced_ : **Network Interface →** `**tun0**`. Empêche toute émission si le tunnel tombe avant le kill-switch.

### Prowlarr

_Settings → Apps_ — le champ « Prowlarr Server » doit contenir `http://172.39.0.2:9696` (l'IP de gluetun, car Prowlarr se présente ainsi aux autres) :

| Application | Adresse                  | Clé API                       |
| ----------- | ------------------------ | ----------------------------- |
| Sonarr      | `http://172.39.0.3:8989` | Settings → General → Security |
| Radarr      | `http://172.39.0.4:7878` | idem                          |

_Settings → Download Clients_ → qBittorrent, hôte `**localhost**`, port 8080.

### FlareSolverr

_Settings → Indexers → Indexer Proxies → **+** → FlareSolverr_

- Host : `http://localhost:8191`
- **Tags** : `flaresolverr`

Le proxy affiche « Disabled » tant qu'aucun tag ne lui est associé — ce n'est pas un état d'erreur.

Appliquer ensuite le tag `flaresolverr` **uniquement sur les indexeurs qui en ont besoin** (1337x dans notre cas). FlareSolverr lance un vrai navigateur : le taguer partout ralentit toutes les recherches.

Symptôme typique dans les logs Prowlarr :

```
Unable to access 1337x.to, blocked by CloudFlare Protection.
```

### Sonarr et Radarr

Root Folders : `/data/shows` et `/data/movies`

_Settings → Download Clients_ → qBittorrent, hôte `**172.39.0.2**`, port 8080.

_Media Management → Importing_ → **Use Hardlinks instead of Copy** ✅

Vérification qu'un lien dur a bien été créé :

```
stat /data/movies/*/*.mkv | grep Links
```

`Links: 2` signifie que le fichier existe dans `downloads` et `movies` sans occuper le double.

### Bazarr

Sonarr `172.39.0.3:8989`, Radarr `172.39.0.4:7878`.

### Seerr

Jellyfin sur `http://192.168.1.24:8096` (LAN, pas `servarrnetwork`), puis Sonarr et Radarr par leurs IP internes.

_Settings → Users → Import Jellyfin Users_ évite de gérer un second jeu d'identifiants.

---

## 6. Reverse proxy

Records locaux Pi-hole vers `192.168.1.12`, hôtes NPM avec certificat wildcard, **sans entrée Cloudflare** :

```
sonarr.ndd.xyz   → 192.168.1.28:8989
radarr.ndd.xyz   → 192.168.1.28:7878
bazarr.ndd.xyz   → 192.168.1.28:6767
prowlarr.ndd.xyz → 192.168.1.28:9696
qbit.ndd.xyz     → 192.168.1.28:8080
```

**Seerr est exposé publiquement** (`seerr.ndd.xyz`, enregistrement Cloudflare) pour permettre les demandes depuis l'extérieur.

Prévoir aussi un record local Pi-hole pour `seerr.ndd.xyz` → `192.168.1.12`, afin d'éviter l'aller-retour par Cloudflare depuis le LAN.

---

## 7. Optimisations

### Ce qui compte vraiment

**Les indexeurs sont le plafond.** Qualité, disponibilité, faux fichiers — tout remonte à la source. Aucun réglage en aval ne compense un mauvais indexeur.

**Adapter la qualité aux appareils.** Chaque fichier qui ne peut pas être lu en _direct play_ est transcodé par le i5 — acceptable en local, coûteux pour un accès distant. Concrètement :

- Plafonner en **1080p** sauf besoin réel de 4K
- Éviter **x265/HEVC** par défaut si des téléviseurs anciens lisent la bibliothèque
- Ignorer les paliers **remux** et **BluRay** : le WEB-DL est indiscernable à distance normale pour une fraction de la taille
- Définir une **taille maximale par qualité**, pour écarter les releases anormalement lourdes

### TRaSH Guides

[trash-guides.info](https://trash-guides.info) est la référence de configuration des \*arr : schémas de nommage, définitions de qualité, custom formats.

Deux apports immédiats :

- Les **schémas de nommage** intègrent qualité, codec et groupe dans un format que Jellyfin exploite pour décider du direct play
- Les **custom formats** couvrent les groupes connus pour de mauvais encodages, les upscales et les releases mal étiquetées

### Filtrer les exécutables

Un `.exe` de 900 Mo a déjà été téléchargé depuis un tracker public. Sonarr et Radarr ne l'importeront jamais dans la bibliothèque, mais il reste dans `downloads` — et ce dossier est exposé en SMB.

Custom Format dans Sonarr et Radarr, condition **Release Title**, ni Negate ni Required :

```
\.(exe|scr|bat|cmd|com|pif|msi|vbs|jar|ps1|lnk|apk)
```

Score **-10000** dans le Quality Profile.

Complément côté qBittorrent, _Settings → Downloads → Run external program on torrent finished_ :

```
#!/bin/bash
find "$1" -type f \( \
  -iname "*.exe" -o -iname "*.scr" -o -iname "*.bat" -o -iname "*.cmd" \
  -o -iname "*.com" -o -iname "*.pif" -o -iname "*.msi" -o -iname "*.vbs" \
  -o -iname "*.jar" -o -iname "*.ps1" -o -iname "*.lnk" -o -iname "*.apk" \
\) -delete
```

Paramètre `%F`. Script à placer dans `/opt/arr/qbittorrent/scripts/`, soit `/config/scripts/` vu du conteneur.

### Bazarr — fournisseurs

Inutilisable sans comptes. Pour le français :

- [**OpenSubtitles.com**](http://OpenSubtitles.com) (compte gratuit requis, et non le provider `.org` obsolète)
- **Podnapisi**, **Subf2m** — sans compte

_Settings → Subtitles_ :

- **Use embedded subtitles** ✅
- **Adaptive searching** ✅
- **Automatic subtitle synchronization** ✅ — des sous-titres désynchronisés sont pires que pas de sous-titres

Language Profile : français en principal, anglais en secours.

### Divers

- **Prowlarr → priorité des indexeurs** : nombre plus bas = priorité plus haute
- **Sonarr → Series Type** : anime et séries quotidiennes se parsent différemment
- **qBittorrent → Advanced** : monter les connexions globales à \~500 et par torrent à \~100 aide sur les trackers publics
- **Jellyfin → Trickplay** : miniatures de prévisualisation lors du défilement, généré une fois par fichier
