# Isyrr

Docker compose file to run the stack. You will still need to log in and configure each service once running. \(See below\)

Server Specs

| | Minimum | Recommended |
| --- | --- | --- |
| **CPU**| 2 cores | 4 cores |
| **RAM** | 8gb | 16gb |
| **Configs Disk** | 50gb SSD | 80gb SSD|
| **Data Disk** | depends on size of library, HDD fine | depends on size of library, HDD fine |

## Environment Variables

To run this stack, you will need to add the following environment variables to your .env file

**Base**

`COMMON_PATH=` This is the path to your faster, smaller storage. Ensure this path exists on the server running Docker for this stack. Ex. /home/yourusername/data

`DATA_PATH=` This is the path to your mass storage. Ensure this path exists on the server running Docker for this stack. Ex. /mnt/data1/media

`TZ=` Reference [this](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones)

`SERVER_IP=` The IP of the server running Docker for this stack.

**VPN**

`WIREGUARD_PRIVATE_KEY=` For NordVPN, you will need to follow [this](https://lazyadmin.nl/home-network/nordvpn-wireguard-as-unifi-vpn-client/) guide up to step 6. Just run $privateKey in Powershell after to get the Private Key. If you are using something other than nord... godspeed


Any one of these can be set depending on how specific you want your VPN connection to be. Reference [this](https://raw.githubusercontent.com/qdm12/gluetun/refs/heads/master/internal/storage/servers.json) for list of servers depending on your VPN provider. Can be a comma separated list. I have examples filled in

`SERVER_COUNTRIES="United States"` 

`SERVER_REGIONS=America`

`SERVER_CITIES="New York City, Boston"`

## Compose Configs

Below is each service in the compose file split up with more explanation.

Homepage
```yaml
homepage:
    image: ghcr.io/gethomepage/homepage:latest
    container_name: homepage
    ports:
      - 3000:3000
    volumes:
      - ${COMMON_PATH}/configs/homepage/config:/app/config
      - ${COMMON_PATH}/configs/homepage/images:/app/images
      - ${COMMON_PATH}/configs/homepage/icons:/app/icons
      - /var/run/docker.sock:/var/run/docker.sock:ro
    environment:
      - PUID=0
      - PGID=0
      - TZ=${TZ}
      - HOMEPAGE_VAR_SERVER_IP=${SERVER_IP}
      - HOMEPAGE_ALLOWED_HOSTS=*
    restart: unless-stopped
```

Gluetun
```yaml
nordvpn:
    container_name: gluetun
    image: qmcgaw/gluetun
    cap_add:
      - NET_ADMIN
    ports:
      - 8080:8080
      - 51420:51420
      - 6789:6789
      - 51420:51420/udp
      - 51820:51820
      - 51820:51820/udp
    volumes:
      - ${COMMON_PATH}/configs/gluetun
    environment:
      - VPN_SERVICE_PROVIDER=nordvpn
      - WIREGUARD_PRIVATE_KEY=${WIREGUARD_PRIVATE_KEY}
      - SERVER_COUNTRIES=${SERVER_COUNTRIES}
      - VPN_TYPE=wireguard
    restart: always
```

qBitTorrent
```yaml
qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    network_mode: service:nordvpn
    container_name: qbittorrent
    depends_on:
      - nordvpn
    environment:
      - WEBUI_PORT=8080
      - PUID=0
      - PGID=0
      - TZ=${TZ}
    volumes:
      - ${COMMON_PATH}:${COMMON_PATH}
      - ${COMMON_PATH}/configs/qbittorrent:/config
      - ${DATA_PATH}/qbittorrent/downloads:/downloads
    restart: unless-stopped
```

nzbget
```yaml
nzbget:
    image: lscr.io/linuxserver/nzbget:latest
    container_name: nzbget
    environment:
      - PUID=0
      - PGID=0
      - TZ=${TZ}
    volumes:
      - ${COMMON_PATH}/configs/nzbget:/config
      - ${DATA_PATH}/qbittorrent/downloads:/downloads
    depends_on:
      - nordvpn
    restart: unless-stopped
    network_mode: service:nordvpn
```

Flaresolverr
```yaml
flaresolverr:
    image: ghcr.io/flaresolverr/flaresolverr:latest
    container_name: flaresolverr
    environment:
      - LOG_LEVEL=${LOG_LEVEL:-info}
      - LOG_HTML=${LOG_HTML:-false}
      - CAPTCHA_SOLVER=${CAPTCHA_SOLVER:-none}
      - TZ=${TZ}
    ports:
      - 8191:8191
    restart: unless-stopped
```

Prowlarr
```yaml
prowlarr:
    image: lscr.io/linuxserver/prowlarr:latest
    container_name: prowlarr
    environment:
      - PUID=0
      - PGID=0
      - TZ=${TZ}
    volumes:
      - ${COMMON_PATH}/configs/prowlarr:/config
    ports:
      - 9696:9696
    restart: unless-stopped
```

Jackett
```yaml
jackett:
    image: lscr.io/linuxserver/jackett:latest
    container_name: jackett
    environment:
      - PUID=0
      - PGID=0
      - TZ=${TZ}
    volumes:
      - ${COMMON_PATH}/configs/jackett:/config
    ports:
      - 9117:9117
    restart: unless-sto
```

Sonarr
```yaml
sonarr:
    image: lscr.io/linuxserver/sonarr:latest
    container_name: sonarr
    environment:
      - PUID=0
      - PGID=0
      - TZ=${TZ}
    volumes:
      - ${COMMON_PATH}:${COMMON_PATH}
      - ${COMMON_PATH}/configs/sonarr:/config
      - ${DATA_PATH}/sonarr/tv:/tv
      - ${DATA_PATH}/qbittorrent/downloads:/downloads
    ports:
      - 8989:8989
    restart: unless-stopped
```

Radarr
```yaml
radarr:
    image: lscr.io/linuxserver/radarr:latest
    container_name: radarr
    environment:
      - PUID=0
      - PGID=0
      - TZ=${TZ}
    volumes:
      - ${COMMON_PATH}:${COMMON_PATH}
      - ${COMMON_PATH}/configs/radarr:/config
      - ${DATA_PATH}/radarr/movies:/movies
      - ${DATA_PATH}/qbittorrent/downloads:/downloads
    ports:
      - 7878:7878
    restart: unless-stopped
```

Jellyfin
```yaml
jellyfin:
    image: lscr.io/linuxserver/jellyfin:latest
    container_name: jellyfin
    environment:
      - PUID=0
      - PGID=0
      - TZ=${TZ}
      - NVIDIA_VISIBLE_DEVICES=all
    ports:
      - 8096:8096
      - 8920:8920
      - 7359:7359/udp
      - 1900:1900/udp
    volumes:
      - ${COMMON_PATH}:${COMMON_PATH}
      - ${COMMON_PATH}/configs/jellyfin:/config
      - ${COMMON_PATH}/jellyfin/cache:/cache
      - ${DATA_PATH}/sonarr/tv:/data/tvshows
      - ${DATA_PATH}/radarr/movies:/data/movies
      - ${DATA_PATH}/qbittorrent/downloads:/data/media_downloads
    restart: unless-stopped
```

Jellyseer (will be replaced with Seer soon)
```yaml
jellyseerr:
    image: fallenbagel/jellyseerr:latest
    container_name: jellyseerr
    environment:
      - LOG_LEVEL=debug
      - TZ=${TZ}
    ports:
      - 5055:5055
    volumes:
      - ${COMMON_PATH}/configs/jellyseerr:/app/config
    restart: unless-stopped
```

Bizarr
```yaml
bazarr:
    image: lscr.io/linuxserver/bazarr:latest
    container_name: bazarr
    environment:
      - PUID=0
      - PGID=0
      - TZ=${TZ}
    ports:
      - 6767:6767
    volumes:
      - ${COMMON_PATH}:${COMMON_PATH}
      - ${COMMON_PATH}/configs/bazarr:/config
      - ${DATA_PATH}/sonarr/tv:/tv
      - ${DATA_PATH}/radarr/movies:/movies
    restart: unless-stopped
```

Wizarr
```yaml
wizarr:
    container_name: wizarr
    image: ghcr.io/wizarrrr/wizarr
    ports:
      - 5690:5690
    volumes:
      - ${COMMON_PATH}/wizarr:/data
    environment:
      - APP_URL=http://10.39.199.30:5690
      - DISABLE_BUILTIN_AUTH=false
      - TZ=${TZ}
```

NginxProxyManager
```yaml
npm:
    container_name: npm
    image: jc21/nginx-proxy-manager:latest
    restart: unless-stopped
    ports:
      - 80:80
      - 443:443
      - 81:81
    environment:
      - TZ=${TZ}
    volumes:
      - ${COMMON_PATH}/npm/data:/data
      - ${COMMON_PATH}/npm/letsencrypt:/etc/letsencrypt
```


