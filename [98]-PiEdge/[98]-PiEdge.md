# pi-edge : Raspberry Pi 5 (QDevice, Tailscale, Komodo, Pi-hole secondaire)

Hôte physique hors cluster Proxmox, dédié aux services qui doivent rester disponibles pendant la maintenance des nœuds.

## Matériel et système

| Élément        | Valeur                                                                       |
| -------------- | ---------------------------------------------------------------------------- |
| Machine        | SunFounder Pironman 5, Raspberry Pi 5 16 Go                                  |
| Stockage       | NVMe 256 Go                                                                  |
| OS             | Raspberry Pi OS Lite 64 bits (Debian 13 « trixie »), noyau 6.18 rpt-rpi-2712 |
| Taille de page | 16 Ko (noyau par défaut, conservé)                                           |
| Nom d'hôte     | `pi-edge`                                                                    |
| Adresse IP     | `192.168.1.98` (réservation DHCP, Ethernet)                                  |
| Alimentation   | Prise non connectée, hors prise « smart »                                    |

L'adresse .98 se place à côté des deux nœuds (`pve-home` .99, `pve-nas` .100).

## Configuration de base

`apt update && apt full-upgrade -y hostnamectl set-hostname pi-edge sed -i 's/\b\(qdevice\|raspberrypi\)\b/pi-edge/g' /etc/hosts curl -fsSL https://get.docker.com | sh        # Docker 29.8.1 usermod -aG docker $USER `

### SSH root

Connexion root par clé.`~/.ssh/config` :

`Host pi-edge     HostName 192.168.1.98     User root     IdentityFile ~/.ssh/id_ed25519     AddKeysToAgent yes     UseKeychain yes `

Connexion : `ssh root@pi-edge`. Le drop-in `/etc/ssh/sshd_config.d/00-root.conf`

## QDevice (troisième voix du cluster)

Le cluster `HomeCluster` compte deux nœuds ; `pi-edge` fournit le vote de départage.

`# sur pi-edge apt install -y corosync-qnetd  # sur pve-home et pve-nas apt install -y corosync-qdevice  # depuis pve-home (accès SSH root vers pi-edge requis pendant l'opération) pvecm qdevice setup 192.168.1.98 `

Résultat vérifié avec `pvecm status` :

- `Expected votes: 3`, `Total votes: 3`, `Quorum: 2`
- `Flags: Quorate Qdevice`
- Membres : `192.168.1.99` (local), `192.168.1.100`, `Qdevice`, tous avec les indicateurs `A,V,NMW`

Vérification côté Pi : `corosync-qnetd-tool -l`. Le service écoute sur le port 5403. L'appairage utilise l'adresse IP, jamais un nom, pour ne pas dépendre du DNS.

## Tailscale (nœud de sortie et routeur de sous-réseau)

`curl -fsSL https://tailscale.com/install.sh | sh cat > /etc/sysctl.d/99-tailscale.conf <<'EOF' net.ipv4.ip_forward=1 net.ipv6.conf.all.forwarding=1 EOF sysctl -p /etc/sysctl.d/99-tailscale.conf tailscale up --advertise-exit-node --advertise-routes=192.168.1.0/24 `

Dans la console d'administration : exit node et route `192.168.1.0/24` approuvés pour `pi-edge`.

| Rôle                                    | pi-edge    | CT 110 (pve-home)                        |
| --------------------------------------- | ---------- | ---------------------------------------- |
| Routeur du sous-réseau `192.168.1.0/24` | principal  | secours (bascule automatique des routes) |
| Exit node                               | par défaut | secours (choix manuel)                   |

Sélectionner le nœud de sortie sur un client : `tailscale set --exit-node=pi-edge`. La bascule d'un exit node vers l'autre est manuelle, seules les routes de sous-réseau basculent automatiquement.

### DNS Tailscale

Nameservers globaux (« Override DNS servers » activé, « Use with exit node » activé) :

1. `192.168.1.11` (Pi-hole principal, CT 111)
2. `192.168.1.98` (Pi-hole secondaire, pi-edge)

## Komodo (gestion des conteneurs du Pi)

Portainer reste en place sur le ThinkStation, sans changement. Komodo gère uniquement `pi-edge`.

`mkdir -p /opt/komodo && cd /opt/komodo wget https://raw.githubusercontent.com/moghtech/komodo/main/compose/mongo.compose.yaml wget https://raw.githubusercontent.com/moghtech/komodo/main/compose/compose.env chmod 600 compose.env docker compose -p komodo -f mongo.compose.yaml --env-file compose.env up -d `

- Interface : `http://192.168.1.98:9120`
- Base : MongoDB (image `mongo`), testé au préalable sur le noyau 16 Ko (`docker run mongo:8` puis `db.runCommand({ping:1})` renvoie `{ ok: 1 }`).
- Fichiers dans `/opt/komodo` : `mongo.compose.yaml`, `compose.env` (mots de passe et secrets, permissions 600).

## Pi-hole secondaire

Pi-hole principal : v6.4.1, CT 111, `192.168.1.11`. Le secondaire tourne sur `pi-edge` dans une stack Komodo (serveur `pi-edge`), avec `nebula-sync` v0.11.2 pour recopier la configuration du principal vers le secondaire (sens unique).

`services:   pihole:     image: pihole/pihole:latest     container_name: pihole     restart: unless-stopped     ports:       - "53:53/tcp"       - "53:53/udp"       - "8080:80/tcp"     environment:       TZ: Europe/Paris       FTLCONF_webserver_api_password: ${PIHOLE_PASSWORD}       FTLCONF_dns_listeningMode: ALL     volumes:       - ./etc-pihole:/etc/pihole     cap_add:       - SYS_NICE    nebula-sync:     image: ghcr.io/lovelaze/nebula-sync:latest     container_name: nebula-sync     restart: unless-stopped     depends_on:       - pihole     environment:       - PRIMARY=http://192.168.1.11|${PIHOLE_PASSWORD}       - REPLICAS=http://pihole|${PIHOLE_PASSWORD}       - FULL_SYNC=false       - SYNC_CONFIG_DNS=true       - SYNC_CONFIG_DNS_EXCLUDE=listeningMode,interface       - SYNC_GRAVITY_GROUP=true       - SYNC_GRAVITY_AD_LIST=true       - SYNC_GRAVITY_AD_LIST_BY_GROUP=true       - SYNC_GRAVITY_DOMAIN_LIST=true       - SYNC_GRAVITY_DOMAIN_LIST_BY_GROUP=true       - SYNC_GRAVITY_CLIENT=true       - SYNC_GRAVITY_CLIENT_BY_GROUP=true       - RUN_GRAVITY=true       - CRON=0 * * * *       - TZ=Europe/Paris `

- Interface web du secondaire : `http://192.168.1.98:8080`.
- Le mot de passe est identique sur les deux Pi-hole. Éviter les caractères `|` et `,`.
- Toute modification DNS se fait sur le principal uniquement, le secondaire est écrasé à chaque synchronisation.

## Pironman et Home Assistant

- Service `pironman5` actif sur `pi-edge`. Tableau de bord et API locale sur le port 34001, InfluxDB pour l'historique.
- Intégration Home Assistant via HACS : dépôt personnalisé `https://github.com/ChristophKomenda/pironman5-homeassistant` (catégorie Intégration), hôte `192.168.1.98`, port `34001`.
- Entité RGB : `light.living_room_piedge_rgb`. L'intégration expose aussi les capteurs (température, charge, mémoire, NVMe, réseau, ventilateur) et le mode des ventilateurs.

## Ports et services sur pi-edge

| Port  | Service                                                                                    |
| ----- | ------------------------------------------------------------------------------------------ |
| 22    | SSH (clé)                                                                                  |
| 53    | Pi-hole secondaire (TCP et UDP)                                                            |
| 5403  | corosync-qnetd                                                                             |
| 8080  | Pi-hole secondaire, interface web                                                          |
| 9120  | Komodo                                                                                     |
| 34001 | API Pironman (utilisée par Home Assistant, sans authentification, réseau local uniquement) |
