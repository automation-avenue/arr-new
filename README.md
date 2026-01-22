# ARR stack NEW VERSION <br />
Below are instructions for Debian / Ubuntu operating system, but docker can be natively run on any linux distro <br />
and if you have Windows or Mac - you can use for tools like Docker Desktop to run docker containers. <br />

### Install docker compose and prepare environment <br />

Install docker compose as per instructions here: [LINK](https://docs.docker.com/compose/) <br />

So - go to `Install` (on the left) to Plugin - scroll down to `Install using the Repository` – Ubuntu (unless you install on other OS): <br />
You should be in [THIS LOCATION](https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository) <br /> <br />

Copy ALL those commands that are listed there, sth like: <br />
```
# Add Docker's official GPG key:
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
```
<br />

Then run:
```
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl status docker
sudo docker run hello-world
```

To test if docker compose has been installed, run : <br />
`docker compose` <br />

You should get a lot of command arguments including 'version' one, so run again: <br />
`docker compose version` <br />

That will show all works as expected. <br />
The stack was created by following this [TRASH GUIDE](https://trash-guides.info/File-and-Folder-Structure/How-to-set-up/Docker/) now: <br />
Create folder structure (this is for torrents only, for torrents + usenet client like NZBGet or SABnzbd - see the other example on trash guide):  <br />

```
sudo mkdir -p /data/{torrents/{tv,movies,music},media/{tv,movies,music}}
sudo apt install tree
tree /data
sudo chown -R 1000:1000 /data
sudo chmod -R a=,a+rX,u+w,g+w /data
ls -ln /data
```
 <br />

Trash guide docker-compose configuration can be found [HERE](https://trash-guides.info/File-and-Folder-Structure/How-to-set-up/Docker/) (scroll down a bit) <br />
You can find more information on [SERVARR](https://wiki.servarr.com/radarr/installation/docker) <br />
My docker-compose.yml file can be found [HERE](https://github.com/automation-avenue/arr-new/blob/main/docker-compose.yml) <br />
You can use command like `git clone https://github.com/automation-avenue/arr-new.git` or simply copy that docker-compose file from that repo: <br />
`sudo nano docker-compose.yml` - and paste it <br /> <br />

Note that hostnames are not needed here as we have dedicated network for our containers <br />

You should be able to run all services now with simple `sudo docker compose up -d` :) <br />

***************************

# Configure services: <br />

Now you need to ensure your internal application settings match. <br />
 - Radarr: Inside the web UI, your "Root Folder" for your library should be `/data/media/movies` (`/data/media/tv` for Sonarr and `/data/media/music` for Lidarr). <br />
 - qBittorrent: Your download path should be set to `/data/torrents` <br />
 - because both paths are on the same mount (`/data`), the OS treats them as the same file system, enabling instant hard links, see below: <br />

   
## qBittorrent: <br />
Check logs for qbittorrent container: <br />
`sudo docker logs qbittorrent` <br />
You will see in the logs something like: <br />
*The WebUI administrator username is: admin <br />
The WebUI administrator password was not set. A temporary password is provided for this session: <your-password-will-be-here>*  <br /><br />
Now you can go to URL: <br />
If you are on the host: `http://localhost:8080` <br />
From other device on your network: `http://<host ip address>:8080` <br />
and log on using details provided in container logs. <br />
Go to `Tools - Options - WebUI` - you can change the user and password here but remember to scroll down and save it. <br /><br />

In left panel go to Categories - All - right click and 'add category': <br />

For Radarr: `Category: movies` <br />
`Save Path: movies` (this will be appended to '/data/torrents/ Default Save Path you set above) <br /> 
For Sonarr: `Category: tv` <br />
`Save Path: tv` <br />
For Lidarr: `Category: music` <br />
`Save Path: music` <br />

Create categories first and only then configure the steps below, as doing it opposite way round caused the Categories to disappear :) <br />

With categories created - go to -  `Tools - Options - Downloads` and in `Saving Management` make sure your settings match [THIS](https://trash-guides.info/Downloaders/qBittorrent/How-to-add-categories/) <br />
So `Default Torrent Management Mode - Automatic`<br />
`When Torrent Category changed - Relocate torrent`  <br />
`When Default Save Path Changed - Switch affected torrents to Manual Mode`  <br />
`When Category Save Path Changed - Switch affected torrents to Manual Mode`  <br />
Tick BOTH BOXES for `Use Subcategories` and `Use Category paths in Manual Mode` (NOT shown on Trash Guides) <br />
Default Save Path: - set to `/data/torrents` (so it matches your folder structure) - then scroll down and `Save`. <br />
On Trash Guides it shows `Copy .torrent files to` but its optional, you can leave it blank <br /> <br />

If you still have problems with adding categories, you can use different image like the one below:
```
  qbittorrent:
    <<: *common-keys
    container_name: qbittorrent
    image: ghcr.io/qbittorrent/docker-qbittorrent-nox:latest
    ports:
      - 8080:8080
      - 6881:6881
      - 6881:6881/udp
    environment:
      - QBT_LEGAL_NOTICE=confirm
      - WEBUI_PORT=8080
      - TORRENTING_PORT=6881
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /docker/appdata/qbittorrent:/config
      - /data:/data
```

Thats it for qBittorrent.<br /><br />

Now configure Prowlarr service (each of these services will require to set up user/pass): <br />
Use 'Form (login page) authentication and set your user and pass for all. <br />

## Prowlarr: <br />
`http://<host ip>:9696` <br />
Go to `Settings - Download Clients` - `+` symbol - Add download client - choose `qBittorrent` (unless you decided touse different download client) <br />
UNTICK the `Use SSL` (unless you have SSL configured in qBittorrent - Tools - Options -WebUI but by default it is not used) <br />
Host - use `qbittorrent` and port - put the port id matching the WebUI in docker-compose for qBittorrent (default is `8080`) <br />
username and password - use the one that you configured for qBittorrent in previous step <br />
Click little `Test` button at the bottom, make sure you get a green `tick` then `Save`.<br />



## Radarr: <br />
`http://<host ip address>:7878` <br />
Go to `Settings - Media Management - Add Root Folder` (scroll down to the bottom) - set  `/data/media/movies` as your root folder <br />
Still in `Settings - Media Management - click Show Advanced - Importing - Use Hardlinks instead of Copy` - make sure its 'ticked' <br />
Optional - you can also tick `Rename Movies` and `Delete empty movie folders during disk scan` , and in `Import Extra Files` - make sure that box is ticked <br />
and in `Import Extra files` field type `srt,sub,nfo` (those 3 changes are all optional) <br /><br />

Then `Settings- Download clients` - click `plus` symbol, choose `qBittorrent` etc - basically same steps as for Prowlarr <br />
so Host `qbittorrent`, port `8080`, ,make sure SSL is unticked, username admin and password - one you configured for qBittorrent <br /> 
and change the Category to `movies` (needs to match qbittorrent Category) <br /> <br />
Now click the `Test` and if you have green 'tick' - `Save`.<br />
Now go to `Settings - General` - scroll down to API key - Copy API key - go back to `Prowlarr - Settings - Apps` -click `+` - Radarr - paste  API key. <br />
Then change `Prowlarr Server` to `http://prowlarr:9696` and `Radarr Server` to `http://radarr:7878` <br />
Click `Test` and if Green - Save <br /><br />

BTW - you can see how to configure each service for  hardlinks [HERE](https://trash-guides.info/File-and-Folder-Structure/Examples/) <br />
You need to configure SABnzbd / qbittorrent and any other services you run too, not only Radarr or Sonarr <br />



## Sonarr: <br />
`http://<host ip>:8989` br />
Go to `Settings - Media Management - Add Root Folder` - set  `/data/media/tv` as your root folder <br />
Still in `Settings - Media Management - Show Advanced - Importing - Use Hardlinks instead of Copy` - make sure its 'ticked' <br />
Optional - you can also tick `Rename Episodes` and `Delete empty Folders - delete empty series and season folders during disk scan` <br />
Then in `Import Extra Files` - make sure that box is ticked and in `Import Extra files` field type `srt,sub,nfo` (those 3 changes are all optional) <br /><br />

Then `Settings- Download clients` - click `plus` symbol, choose `qBittorrent` etc - basically same steps as for previous services<br />
Host `qbittorrent`, port `8080`, ,make sure SSL is unticked, username admin and password - one you configured for qBittorrent <br /> 
and change the Category to 'tv' (by default its 'tv-sonarr', but you need to match qbittorrent Category) <br /><br />
Now click the 'Test' and if you have green 'tick' - Save.<br />
Now go to `Settings - General` - scroll down to API key - Copy API key - go back to Prowlarr - Settings - Apps -click '+' - Sonarr - paste  API key. <br />
Then change `Prowlarr Server` to `http://prowlarr:9696` and `Sonarr Server` to `http://sonarr:8989`<br />
Click `Test` and if Green - `Save`<br />



## Lidarr: <br />
`http://<host ip address>:8686` <br />
Go to Settings - Media Management - Add Root Folder - set path to /data/media/music as your root folder, set name to Root or whatever and save <br />
Then Settings- Download clients - click 'plus' symbol, choose qBittorrent etc - basically same steps as for previous services<br />
Host 'qbittorrent', port 8080, ,make sure SSL is unticked, username admin and password - one you configured for qBittorrent <br /> 
and change the Category to 'music' (by default its 'lidarr', but you need to match qbittorrent Category) <br />
Now click the 'Test' and if you have green 'tick' - Save.
Now go to Settings - General - scroll down to API key - Copy API key - go back to Prowlarr - Settings - Apps -click '+' - Sonarr - paste  API key. <br />
Then change `Prowlarr Server` to `http://prowlarr:9696` and `Lidarr Server` to `http://lidarr:8686` <br />
Click `Test` and if Green - `Save` <br />


## Bazarr: <br />
`http://host ip address>:6767` <br />
Languages: Go to Settings > Languages and create a "Language Profile" (e.g., "English" or "Any"). <br />
Providers: Go to Settings > Providers and add your subtitle sources (OpenSubtitles.org, Subscene, etc.). Most require a free account. <br />
Sync: After connecting Radarr/Sonarr, go to the Series or Movies tab and click "Update" to pull in your existing library. <br />

****************************

## Restart services: <br />
It might be a good idea to restart all services and see if they come up as expected: <br /> 

```
sudo docker compose down
sudo docker compose up -d
```
 <br />
If the first line that says : <br />
`WARN[0000] No services to build`  - this message is actually expected here.  <br />

**************************

## Remaining config: <br />
That should be it, you just need to add some indexers to Prowlarr. <br />
You can add more indexers - just google for something like 'what are the best legal indexers for Prowlarr' or something similar. <br />

It is a common misconception that the "Arr" stack is only for pirated content.  <br />
In reality, these are powerful automation tools for managing media, and there is a wealth of legal, copyright-free, and open-source content you can use them for. <br />
In Radarr, you can download movies that have entered the Public Domain or are released under Creative Commons licenses. <br />
Public Domain Classics: These are "Golden Age" movies where the copyright was not renewed like: <br />
Night of the Living Dead (1968), His Girl Friday (1940), Charade (1963), and The General (1926). <br />
Configure Prowlarr with The "Gold Standard" Indexer for legal media like The Internet Archive (Archive.org). <br />
They host thousands of public domain movies. <br />

## Jellyfin <br />
`http://<host ip address>:8096` <br />
To watch your movies, just log on to Jellyfin, create user and password and you can `Add Media Library`. <br />
For Content Type - choose `Movies` and find folder `/data/media/movies`. <br />
Add more content types like TV or Music accordingly, binding them to correct media folder. <br />

**************************

# Troubleshooting: <br />
### DNS check:
Test if your containers use CloudFlare DNS (configured in docker-compose.yml file): <br />
`sudo docker exec -it radarr cat /etc/resolv.conf` <br />

### Hardlinks check:<br />
Check if the hardlinks work as expected: <br />
Go to `/data` folder on your host and run `tree` and `du -sch *` commands to see the folder structure. <br />
Find the same file in torrents and media that you have just downloaded and run commands: <br />
`ls -i /data/media/movies/<your video>` and check its inode id (in first column, like 3881112) <br />
Then run again the same command but for the torrent folder: <br />
`ls -i /data/torrents/movies/<your video>` and see if the inode id is the same as above. <br />
If they are - your hardlinks work as expected. <br />
If they don't - first go to logs to see what is the problem (for Radarr/Sonarr go to System - Log Files) <br />
If you have issue where the file is copied rather than hardlinked, then the most probable cause <br />
is the read/write permission on either source or destination, but that can all be found in those logs so start there. <br />


### Files do not move from torrents to media folder: <br />
If the video does not move automatically from torrents to media, then check the Activity - Queue. <br />
You might have a flag saying: 'Downloaded - Unable to Import Automatically' <br />
Click the Manual Import (icon that looks like human head on the far right of the item row) <br />
Confirm the Movie: In the popup, ensure the correct movie is selected in the dropdown. If it is correct, click 'Import' <br />


### FlareSolverr: <br />
You might want to add FlareSolverr if you find Prowlarr is failing to index some sites due to "Cloudflare" blocks: <br />
```
###################################
# FLARESOLVERR - Cloudflare Bypass
###################################

  flaresolverr:
    <<: *common-keys
    container_name: flaresolverr
    image: ghcr.io/flaresolverr/flaresolverr:latest
    ports:
      - 8191:8191
    environment:
      - LOG_LEVEL=info
```

 Once the container is running, you need to tell Prowlarr to use it: <br />
 - Open your Prowlarr Web UI (http://localhost:9696) <br />
 - Go to Settings > Indexers. <br />
 - Click the + (Add) button under Indexer Proxies and select FlareSolverr. <br />
 - Fill in the details: <br />
 - Name: FlareSolverr <br />
 - Host: http://flaresolverr:8191 (Note: Using the service name flaresolverr works because they are on the same Docker network). <br />
 - Tags: Give it a tag like cloudflare (this is important). <br />
 - Save the proxy <br /> <br />


### Jellyfin hardware acceleration: <br />
For Jellyfin hardware acceleration you might want to add bottom 2 lines:  <br />

```
jellyfin:
    <<: *common-keys
    <...snip...>
    devices:
      - /dev/dri:/dev/dri # << container setting to pass through GPU (this requires more steps outside of docker compose though)
```
