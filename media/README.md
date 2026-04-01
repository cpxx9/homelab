# Server

Docker compose file to run the stack. You will still need to log in and configure each service once running. \(See below\)

Server Specs

| | Minimum | Recommended |
| --- | --- | --- |
| **CPU**| 2 cores | 4 cores |
| **RAM** | 8gb | 16gb |
| **Configs Disk** | 50gb SSD | 80gb SSD|
| **Data Disk** | depends on size of library, HDD fine | depends on size of library, HDD fine |

# Pre-Reqs

Docker

```bash
sudo apt update && sudo apt upgrade
```

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
```

```bash
sh get-docker.sh
```

```bash
usermod -aG docker <user>
```

**Set-up**

Download the compose and .env files from this folder, put them in the *same* directory. I recomend something like ~/compose/isyyr

# Environment Variables

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

# Compose Configs

Below is each service in the compose file split up with more explanation.

## Homepage

Paths needed:

${COMMON_PATH}/configs/homepage/config

${COMMON_PATH}/configs/homepage/images

${COMMON_PATH}/configs/homepage/icons

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

## Gluetun

Paths needed:

${COMMON_PATH}/configs/gluetun

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
      - ${COMMON_PATH}/configs/gluetun:/gluetun
    environment:
      - VPN_SERVICE_PROVIDER=nordvpn
      - WIREGUARD_PRIVATE_KEY=${WIREGUARD_PRIVATE_KEY}
      - SERVER_COUNTRIES=${SERVER_COUNTRIES}
      - VPN_TYPE=wireguard
    restart: always
```

## qBitTorrent

Paths needed:

${COMMON_PATH}/configs/qbittorrent

${DATA_PATH}/qbittorrent/downloads

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

## nzbget

Paths needed:

${COMMON_PATH}/configs/nzbget

${DATA_PATH}/qbittorrent/downloads

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

## Flaresolverr

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

## Prowlarr

Paths needed:

${COMMON_PATH}/configs/prowlarr

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

## Jackett

Paths needed:

${COMMON_PATH}/configs/jackett

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

## Sonarr

Paths needed:

${COMMON_PATH}/configs/sonarr

${DATA_PATH}/sonarr/tv

${DATA_PATH}/qbittorrent/downloads

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

## Radarr

Paths needed:

${COMMON_PATH}/configs/radarr

${DATA_PATH}/radarr/movies

${DATA_PATH}/qbittorrent/downloads

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

## Jellyfin

Paths needed:

${COMMON_PATH}/configs/jellyfin

${COMMON_PATH}/jellyfin/cache

${DATA_PATH}/sonarr/tv

${DATA_PATH}/radarr/movies

${DATA_PATH}/qbittorrent/downloads

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

## Jellyseer (will be replaced with Seer soon)

Paths needed:

${COMMON_PATH}/configs/jellyseerr

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

## Bizarr

Paths needed:

${COMMON_PATH}/configs/bazarr

${DATA_PATH}/sonarr/tv

${DATA_PATH}/radarr/movies

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

## Wizarr

Paths needed:

${COMMON_PATH}/wizarr

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

## NginxProxyManager

Paths needed:

${COMMON_PATH}/npm/data

${COMMON_PATH}/npm/letsencrypt

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

# Starting the stack

Navigate to where the compose file is. If following along, should be ~/compose/isyyr

Run `docker compose up` or `docker compose up -d` (to immediately detach)

If you make changes to the compose file, just run `docker compose up -d` from this directory, docker will rebuilt containers that have changes, will not touch containers with no changes.

# Accessing Applications

Once the applications are deployed, you can access them using the following addresses :

> [!IMPORTANT]  
> Replace `localhost` with the IP address of your machine or remote server if needed.


* Jellyfin : http://localhost:8096
* Jellyseer : http://localhost:5055
* Sonarr : http://localhost:8989
* Radarr : http://localhost:7878
* Jackett : http://localhost:9117
* Prowlarr : http://localhost:9696
* qBittorrent : http://localhost:8080
* nzbget : http://localhost:6789
* Bazarr : http://localhost:6767

Gluetun (Nord VPN) will be automatically configured to be used with the applications.

## **Configuration Guide for Web Interfaces Only**

> [!IMPORTANT]  
> All links containing the container name can be replaced with either the server IP or `localhost`. Also, replace `/COMMON_PATH/` with the path you configured in the `.env` file.


## **qBittorrent**

1. Open the WebUI by clicking on the application icon in the **DOCKER** tab and selecting **WebUI**.
2. Log in with the default credentials:
   - **Username**: `admin`
   - **Password**: `adminadmin`

   *Note: The default credentials may have changed, please check the documentation for updates on this. In most cases, qBittorrent Web UI will generate a temporary password when the container is started. To view this password, check the logs for this container with the command: `docker logs qbittorrent`*

1. Once logged in, click the gear icon to go to **Options**.
2. Under the **Downloads** tab, configure the backup settings as follows:
   - **Default Torrent Management Mode**: `Automatic` (required for category-based save paths to work)
   - **When Torrent Category changed**: `Relocate torrent`
   - **When Default Save Path changed**: `Relocate affected torrents`
   - **When Category Save Path changed**: `Relocate affected torrents`
   - **Default Save Path**: `/downloads` 
3. Click **SAVE**.

### **Category Configuration**

1. In the WebUI, expand **CATEGORIES** in the left menu. Right-click on **All** and select **Add category...**.
2. In the **New Category** window, configure as follows:
   - **Category**: `radarr` (this corresponds to the category you will later configure in Radarr)
   - **Save path**: `/downloads/radarr`
3. Click **Add**.
4. Right-click on **All** again, select **Add category...**.
5. Configure as follows:
   - **Category**: `sonarr` (this should match the category configured later in Sonarr, by default `sonarr-tv`, but this guide uses `sonarr`)
   - **Save path**: `/downloads/sonarr`
6. Click **Add**.

---

## **Radarr**

### **Media Management**

1. Open the WebUI and go to **Settings** > **Media Management**.
2. Click **Add Root Folder**, add the path `/COMMON_PATH/radarr/movies`, and click **OK**.
3. Click **Show Advanced** at the top, scroll down to **Importing**, and make sure **Use Hardlinks instead of Copy** is enabled.

### **Download Clients**

1. In the WebUI, go to **Settings** > **Download Clients**.
2. Click **+** under **Download Clients**, then select **qBittorrent** from the **Add Download Client** window.
3. Fill in the fields as follows:
   - **Name**: `qBittorrent` (or another name of your choice)
   - **Host**: `qbittorrent`
   - **Username**: `admin`
   - **Password**: `adminadmin` (change it if you've modified it in qBittorrent)
   - **Category**: `radarr` (this should match the category set in qBittorrent)
4. Click **Test**. If you see a checkmark, it means the connection is working; if not, there is an error.
5. Click **Save**.

_Note: if entering `qbittorrent` as the Host does not work, try entering the IP address instead (ex: `192.168.x.x`)_

> [!WARNING]
> On new installations, Radarr may complain that the `/downloads/radarr` directory does not exist inside the container (this is generally flagged as an error by Radarr in  **System** > **Status**). To fix this, simply move into the directory `/COMMON_PATH/qbittorrent/downloads` and manually create the `radarr` directory. Then, simply delete qBittorrent from Radarr and re-add it -  you should see the error disappear.

### **Indexer Jackett (Optional)**

1. In the WebUI, go to **Settings** > **Indexers**.
2. Click **+** under **Add Indexer**, then select **Torznab**.
3. Fill in the fields as follows:
   - **Name**: `Torznab` (or another name of your choice)
   - **URL**: `http://Jackett:9117/api/v2.0/indexers/YOUR_INDEXERS/results/torznab/`
   - **ApiKey**: Find the API key in the home menu at the top right.
4. Click **Test**. If you see a checkmark, it means the connection is working; if not, there is an error.
5. Click **Save**.

---

## **Sonarr**

### **Media Management**

1. Open the WebUI and go to **Settings** > **Media Management**.
2. Click **Add Root Folder**, add the path `/COMMON_PATH/sonarr/tv`, and click **OK**.
3. Click **Show Advanced**, scroll down to **Importing**, and enable **Use Hardlinks instead of Copy**.

_Note: if entering `qbittorrent` as the Host does not work, try entering the IP address instead (ex: `192.168.x.x`)_

### **Download Clients**

1. In the WebUI, go to **Settings** > **Download Clients**.
2. Click **+** under **Download Clients**, then select **qBittorrent**.
3. Fill in the fields as follows:
   - **Name**: `qBittorrent` (or another name of your choice)
   - **Host**: `qbittorrent`
   - **Username**: `admin`
   - **Password**: `adminadmin` (change it if you've modified it in qBittorrent)
   - **Category**: `sonarr` (this should match the category set in qBittorrent)
4. Click **Test**. If you see a checkmark, it means the connection is working.
5. Click **Save**.

### **Indexer Jackett (Optional)**

1. In the WebUI, go to **Settings** > **Indexers**.
2. Click **+** under **Add Indexer**, then select **Torznab**.
3. Fill in the fields as follows:
   - **Name**: `Torznab` (or another name of your choice)
   - **URL**: `http://Jackett:9117/api/v2.0/indexers/YOUR_INDEXERS/results/torznab/`
   - **ApiKey**: Find the API key in the home menu at the top right.
4. Click **Test**. If you see a checkmark, it means the connection is working; if not, there is an error.
5. Click **Save**.

---

## **Prowlarr**

### **Configure Torrent Indexers**

1. Open the WebUI and go to **Indexers** > **Add New Indexer**.
2. Select **1337x** (or another tracker of your choice).
   - You can modify the settings as per your preference, but the default values generally work well.
   - Sorting by **Seeders** can be useful for faster downloads.
3. Click **Test**. If you see a checkmark, the connection is functional; otherwise, there's an error.
4. Click **Save**.

### **Configure FlareSolverr**

1. Go to **Settings** and click **+** under **Indexer**.
2. Select **FlareSolverr** and fill in the information as follows:
   - **Name**: `FlareSolverr`
   - **Tags**: `flaresolverr`
   - **Host**: `http://flaresolverr:8191/`
3. Click **Test** to check the connection.
4. Click **Save**.

### **Configure Radarr**

1. Go to **Settings** and click **Apps**.
2. Select **Radarr** and fill in the information as follows:
   - **Sync Level**: `Full Sync`
   - **Prowlarr Server**: `http://prowlarr:9696`
   - **Radarr Server**: `http://radarr:7878`
   - **ApiKey**: Find the API key in the Radarr interface under **Settings** > **General** > **API Key**.
3. Click **Test** to check the connection.
4. Click **Save**.

### **Configure Sonarr**

1. Go to **Settings** and click **Apps**.
2. Select **Sonarr** and fill in the information as follows:
   - **Sync Level**: `Full Sync`
   - **Prowlarr Server**: `http://prowlarr:9696`
   - **Sonarr Server**: `http://sonarr:8989`
   - **ApiKey**: Find the API key in the Sonarr interface under **Settings** > **General** > **API Key**.
3. Click **Test** to check the connection.
4. Click **Save**.

---

## **Jellyfin**

### **Initial Setup**

1. Open the Web UI by going to the **DOCKER** tab, click the app logo for Jellyfin, and select **WebUI**.
2. Select a preferred display language (or use the default English). Click **Next** ➝.
3. Create an administrator account, fill out the credentials as desired, and click **Next** ➝.
4. Click **Add Media Library** and fill in the following:
   - **Content type**: Movies
   - **Folders**: `/COMMON_PATH/radarr/movies`
   - Configure the rest as you see fit; the default settings are typically fine.
5. Click **OK**.
6. Click **Add Media Library** again and fill in the following:
   - **Content type**: Shows
   - **Folders**: `/COMMON_PATH/sonarr/tv`
   - Configure the rest as you see fit; the default settings are typically fine.
7. Click **OK**.
8. Click **Next** ➝.
9. Configure the **Preferred Metadata Language** (or use the default), and click **Next** ➝.
10. In **Configure Remote Access**, leave **Allow Remote Connections to this Server** checked and **Enable Automatic Port Mapping** unchecked.
11. Click **Next** ➝, then click **Finish**.
12. Sign in with your administrator account.

Once you sign in, if you already have media in your `/COMMON_PATH/*` folders, it should start appearing in Jellyfin. If not, the content will populate as the folders are filled.

### **Adding Users to Jellyfin**

If you want other users to access your Jellyfin server, you can create additional user accounts. This step is optional if you're the only user.

1. Open the left menu by clicking on the three horizontal lines (hamburger menu) in the upper left corner.
2. Select **Users** and click the **+** button on the left to add a new user.
3. Fill in the following details for the new user:
   - **Name**: `<username>`
   - **Password**: `<password>`
   - Under **Library Access**, check the boxes for the libraries (Movies, TV shows, etc.) that you want the user to have access to.
4. Click **Save** to create the user.
5. Repeat this process for all users you wish to add to the server.

---

## **Jellyseerr**

### **Sign In / Configuration**

1. Open the WebUI and in the **Welcome to Jellyseerr** screen, select **Use your Jellyfin account**.
2. Fill in the information as follows:
   - **Jellyfin URL**: `http://jellyfin:8096/`
   - **Email Address**: `<your email address>`
   - **Username**: `<your Jellyfin username>`
   - **Password**: `<your Jellyfin password>`
3. Select **Sign In**.
4. Go to **Sync Libraries** under **Jellyfin Libraries**, select your Jellyfin libraries, then click **Continue**.

### **Integrating with Radarr**

1. Go to **Radarr Settings**, then click **Add Radarr Server**.
2. Fill in the information as follows:
   - **Default Server**: Check this box
   - **Server Name**: `Radarr`
   - **Name or IP Address**: `http://radarr`
   - **Port**: `7878`
   - **API Key**: Find the API key in the Radarr interface under **Settings** > **General** > **API Key**.
3. Click **Test** to check the connection.
4. Click **Save Changes**.

### **Integrating with Sonarr**

1. Go to **Sonarr Settings**, then click **Add Sonarr Server**.
2. Fill in the information as follows:
   - **Default Server**: Check this box
   - **Server Name**: `Sonarr`
   - **Name or IP Address**: `http://sonarr`
   - **Port**: `8989`
   - **API Key**: Find the API key in the Sonarr interface under **Settings** > **General** > **API Key**.
3. Click **Test** to check the connection.
4. Click **Save Changes**.

---

## **Bazarr**

### **Initial Setup**

1. Open the WebUI by navigating to `http://localhost:6767` (or replace `localhost` with your server's IP address).
2. The setup wizard will guide you through the initial configuration:
   - **Language**: Select your preferred language and click **Next**.
   - **Authentication**: Configure authentication if desired (optional for local access).
   - **General Settings**: Configure your general preferences.
3. Click **Next** and then **Save**.

> [!NOTE]  
> **Path Configuration**: Since you're using the provided Docker Compose files, all directory paths and volume mappings are already configured correctly. Bazarr will automatically detect your Sonarr and Radarr libraries without needing manual path configuration.

### **Configure Sonarr Integration**

1. Go to **Settings** > **Sonarr**.
2. Click **Add** and fill in the following information:
   - **Name**: `Sonarr`
   - **Enabled**: Check this box
   - **Address**: `http://sonarr`
   - **Port**: `8989`
   - **Base URL**: Leave empty
   - **API Key**: Find the API key in the Sonarr interface under **Settings** > **General** > **API Key**.
   - **Minimum Score**: Set according to your preference (recommended: 70-80)
3. Click **Test** to verify the connection.
4. Click **OK** to save.

### **Configure Radarr Integration**

1. Go to **Settings** > **Radarr**.
2. Click **Add** and fill in the following information:
   - **Name**: `Radarr`
   - **Enabled**: Check this box
   - **Address**: `http://radarr`
   - **Port**: `7878`
   - **Base URL**: Leave empty
   - **API Key**: Find the API key in the Radarr interface under **Settings** > **General** > **API Key**.
   - **Minimum Score**: Set according to your preference (recommended: 70-80)
3. Click **Test** to verify the connection.
4. Click **OK** to save.

### **Configure Subtitle Providers**

1. Go to **Settings** > **Providers**.
2. Add your preferred subtitle providers by clicking **Add** and selecting from available providers. Based on community recommendations, here are the most effective providers:

**Recommended Free Providers (No Account Required):**
   - **TVSubtitles**: Excellent for TV shows, no registration needed
   - **YIFYSubtitles**: Great for movies, no registration needed
   - **SuperSubtitles**: Good general provider, no registration needed
   - **EmbeddedSubtitles**: Extracts subtitles from video files
   - **AnimeTosho**: Specialized for anime content

**Recommended Providers (Free Account Required):**
   - **OpenSubtitles.com**: Free account required, much better than the old .org version
   - **Addic7ed**: Free account required, excellent for TV shows

> [!IMPORTANT]  
> **Note about OpenSubtitles**: The old opensubtitles.org now requires a VIP subscription and is no longer recommended for free users. Use **opensubtitles.com** instead, which offers free accounts with good download limits.

3. For providers requiring authentication:
   - **OpenSubtitles.com**: Register at opensubtitles.com and use your username/password
   - **Addic7ed**: Register at addic7ed.com and use your username/password
4. Configure each provider according to your preferences and authentication requirements.
5. Click **Save**.

> [!TIP]  
> Many users report achieving 99% subtitle coverage for movies and 90% for TV episodes using this combination of providers. 
> *Provider recommendations based on community feedback from [r/bazarr](https://www.reddit.com/r/bazarr/comments/1fevojd/the_best_unlimited_provider/)*

### **Configure Languages**

1. Go to **Settings** > **Languages**.
2. Select your preferred languages for subtitles:
   - **Languages Filter**: Add the languages you want subtitles for
   - **Default Enabled**: Check the box for languages you want enabled by default
   - **Series**: Configure language preferences for TV series
   - **Movies**: Configure language preferences for movies
3. Click **Save**.

### **Configure Subtitles**

1. Go to **Settings** > **Subtitles**.
2. Configure your subtitle preferences:
   - **Download**: Set when to search for subtitles (recommended: Manually and when subtitles are wanted)
   - **Subtitle Folder**: Configure how subtitles should be stored (recommended: Alongside media file)
   - **Upgrade Subtitles**: Enable if you want Bazarr to replace existing subtitles with better ones
3. Click **Save**.

Once configured, Bazarr will automatically monitor your Sonarr and Radarr libraries and download subtitles based on your configured preferences.


# **Updating Applications**

To update the applications, you need to stop the running containers and remove the existing Docker images. You can use the following commands to perform these operations:

```bash
docker-compose down
docker image prune -a
```

Then, you can run `docker-compose up -d` to restart the containers with the latest versions of the applications.
