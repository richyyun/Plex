# Plex
 Plex Media Server with Dockerized suite

## TODO
- ~~Remote access~~
- ~~Hardware transcoding~~
- ~~Cloudflare tunnel~~
- ~~Double check Radarr / Sonarr functionality~~
- ~~Automatic startup at boot~~
- tdarr (format changer, to h264 or h265?)
- watchtower (automatic container updater)
- start Ubuntu headless (GUI-less at least. `systemctl disable lightdm.service`)
- Usenet
- Git README

## Build
### Hardware
- Beelink S12 Pro Mini PC, Intel 12th Gen Alder Lake- N100, 16GB DDR4 RAM 500GB PCIe SSD
- QNAP TR-004 4 Bay USB Type-C Direct Attached Storage with hardware RAID + 4 4TB WD Red Plus HDD

### OS
- Ubunutu Desktop 22.04.3 LTS
- Linux Kernel 6.4.0-060400-generic
 
### Docker
After much research I decided to implement everything on Docker in a Linux OS. The reasons, from most important to me to the least were:
1. Hardware transcoding with iGPU is supported in Linux but not Windows
2. I was aware of Docker's existence but not familiar with it and I wanted to learn more about Docker
3. Docker makes adding additional programs / services, keeping them updated, and tweaking properties much simpler
4. Docker is more lightweight than individually installed programs

There was definitely a steep learning curve. I still don't quite understand all the underlying mechanisms that are operating under the hood, but I'll try to lay out its implementation as simply as possible here. Assuming you know how to navigate Linux, read through the [Docker website](https://docs.docker.com/get-started/overview/) or watch this [youtube video](https://www.youtube.com/watch?v=aLipr7tTuA4) to get a better sense of what Docker is doing. 

To start, we need to install the docker engine, which can be done by following the instructions [here](https://docs.docker.com/engine/install/ubuntu/). I also installed Docker Desktop, and have found it to be really handy for keeping containers organized and inspecting the container instances.

I used docker compose as it makes laying out the services incredibly easy. I also preferred it over CLI as it was cleaner to read. There is a ton of documentation on [general usage](https://docs.docker.com/compose/compose-file/compose-file-v3/) as well as [specific examples](https://docs.linuxserver.io/general/docker-compose/) for setting up services.

In short, all you need to do is create a yaml file in the directory you would like the Docker containers to live in. Defining a service looks like:
```
version: "3.8"                                  # Defines the compose version for the entire yaml file, defined once at the top
services:                                       # Starts the list of services
  heimdall:                                     # Local name of the service
    image: linuxserver/heimdall                 # The docker image to pull - this is what defines what program is actually running
    container_name: heimdall                    # Yet another name, I believe it's not actually used anymore but left in to keep it backwards compatible
    volumes:                                    # Defining volumes to map 
      - /home/user/appdata/heimdall:/config     
    environment:                                # Environment variables
      - PUID=1000
      - PGID=1000
      - TZ=Europe/London
    ports:                                      # Ports to route
      - 80:80
      - 443:443
    restart: unless-stopped                     # Restart option
```
Most of the layout is straightforward except for 3 parts that are common to every service:
1. **volumes** - Docker runs software in an isolated environment with its own file structure. Volumes defines a local directory that is mapped to the container directory (local:container) such that the container can store files in the local machine. Looking up the specific docker image will usually specify what volumes need to be defined and what they mean. I believe every container has the `/config` folder to store settins and logs.
2. **environment** - The most important part to understand here is the PUID and PGID which specify what user the container will be able to act as. It's typically fine to use the user's UID and GUI (found by the `id` command in terminal) but more advanced setups may require different permissions settings.
3. **ports** - services in Docker usually have a webUI that can be accessed via `localhost:port` on a browser. The documentation on the docker image will usually tell you the default ports that need to be mapped, and they're typically defined so that they do not use a common port. Small exception in this example with port 80 (http) but that's just how Heimdall is setup to function. 

All other settings can be found by looking up the image documentation. 

Note: I also use predefined variables, seen as $VAR in my yml file, which can be defined in an .env file as VAR=value.

### Services
1. [gluetun](https://github.com/qdm12/gluetun-wiki) - VPN container
   - Can run various different VPN services (I use ExpressVPN)
   - `/dev/net/tun` device allows for the VPN networking
   - List all ports for services using the VPN network
   - User and passwords are typically grabbed by selecting "manual setup" on your VPN account, NOT your VPN login info.
2. [plex](https://hub.docker.com/r/linuxserver/plex) - Plex media server
   - Use `network_mode:host` if possible. Otherwise, need to map all the ports listed
   - Requires mapping of `/movies` and `/tv`
   - `/dev/dri` device enables hardware transcoding
   - Not using right now. For some reason hardware transcoding with my CPU does not work in Docker but works fine when just installing directly.
3. [radarr](https://hub.docker.com/r/linuxserver/radarr) - movie collection manager
   - Requires mapping of `/downloads` which is where the movies are downloaded to and `/movies` which is where the movies will be stored
4. [sonarr](https://hub.docker.com/r/linuxserver/sonarr) - tv show collection manager
   - Requires mapping of `/downloads` which is where the movies are downloaded to and `/tv` which is where the shows will be stored
5. [prowlarr](https://hub.docker.com/r/linuxserver/prowlarr) - indexer manager. Both radarr and sonarr rely on indexers (ways of tracking down media).
6. [bazarr](https://hub.docker.com/r/linuxserver/bazarr) - subtitle collection manager
7. [qbittorrent](https://hub.docker.com/r/linuxserver/qbittorrent) - torrent client
   - Requires mapping of `/downloads` which is where everything is downloaded to. Must be the same locally as the `/downloads` in radarr and sonarr
8. [overseer](https://hub.docker.com/r/linuxserver/overseerr) - media request management
9. [heimdall](https://hub.docker.com/r/linuxserver/heimdall) - application dashboard
10. [tautulli](https://hub.docker.com/r/linuxserver/tautulli) - Plex server stats monitoring

Note I used to have other services behind the VPN that I removed while I was debugging using an expressvpn image and never got around to using it again. It's likely good practice to put most services behind the VPN as it does not hurt. You do have to keep in mind to use the IP address instead of localhost to point things to that service though.
To put a service on the VPN simply add `network_mode: service:gluetun`. You can also add a `depends_on:` clause so it starts up after the VPN. If not using a VPN (or if the VPN is standalone and not dockerized) you need to setup an internal network such that each service can access one another as they will be isolated otherwise.  

## Setup
### Settings
You can login and/or apply settings to each service by accessing its webUI through localhost:port on a browser. Note that for Plex it's localhost:32400/web/. Besides basic settings, the following connections have to be made between services using authentication tokens:
1. radarr and sonarr need to access qbittorrent
2. bazarr, prowlarr, and overseer need to access radarr and sonarr
3. tautulli needs to access plex
   
### Cloudflare tunnel
Once the above is setup everything should be functioning. However, one major limitation is that you will not be able to access any of the services when outside the local network. One solution is to forward the port, but this can get dangerous especially if you're allowing access to overseer / radarr / sonarr as they have capabilities of downloading files. What I've found to be the best solution both in terms of ease of setup and cheap price is using a [Cloudflare tunnel]([https://www.cloudflare.com/products/tunnel/](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/). This effectively uses Cloudflare as an intermediate network to access the local port.

To set it up, make an account on Cloudflare and purchase a domain name. Afterwards, simply follow [this page](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/get-started/create-remote-tunnel/) to set up a tunnel to one of the services hosted at localhost:port. You can host multiple services using subdomains, e.g. `overseer.richyplex.com` and `tautulli.richyplex.com` to have access to the services you'd like. The tunnel will also give you a single line command to run in Docker for authentification. Note that all the services make you login when first accessing them through a different device. 

I recommend setting it up for at least overseer so you can use its request function while on the go by just opening up a browser. On phones you can also save a website as an app.

Note I could not get it to work on compose so had to do it w/ CLI. I recommend editing the run to give it a `--name` as well as the `-d` flag to make sure it runs detached.

### Webhooks
Another tool I set up is a way for me to be notified when there is an event on one of the services. There are some Docker containers that work to do so, but I've found it to be the easiest to work with webhooks on a Discord server. You simply make a new Discord channel, create a webhook, and copy paste into the services you would like reporting. If you have other home automation setups, you can use webhooks to connect them together so certain functions (like lights dimming) happen when you start a movie, etc. 

### Robusifying
One main issue I ran into after setting it all up was power outages. When the power was restored I had to boot up the machine, start the docker engine, and start the containers. While not a ton of work, I realized this would be an issue if I were to be away and the power go down. To make sure the whole system is able to start itself after the power comes back on you need to:
1. Change your BIOS settings to make the PC boot when power comes on. For the Beelink PC I have it's under Chipset>bottom row>State after G3>set to S0
2. Set restart flags of all containers to be "always". This ensures the container starts up when the docker engine starts. For the cloudflared CLI, you need to add a `--restart always` flag.
3. run `sudo systemctl enable docker` to force the docker service to start. You may need to run `sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin` to ensure that docker works as a service.

One thing I realized is that docker desktop and the docker service engine seem to be operating independently, and for this to work the docker containers must be running on the CLI engine (i.e. the service). To ensure this is happening, close docker desktop and reboot your computer. Then run `docker compose up -d` and the cloudflare docker command. It may pull the images again as well. 

## Troubleshooting
This seems pretty straightforward but I ran into a lot of roadblocks that I had to work around to get everything to actually work. Of those, here are the main difficulties I faced. 

### Permissions
Permissions in Linux based systems is a giant rabbithole. I was not familiar enough with them before starting the setup which definitely made certain components more difficult or confusing. To keep things simple:
- `root` is the "administrator" in Linux systems and has access to everything. It is defined as a different user than the actual user account, but users can access `root` level access by starting commands with `sudo`.
- Groups are lists of users that's often used to define permissions to a number of users without having to list them one by one.
- Docker, as mentioned before, runs software in an isolated environment but has access to volumes that have been mapped to the container. By passing the PUID and GUID of the user (found with command `id`) the container then acts as if it has the same permissions as the user running the docker containers. It is very common to run into permissions issues at first, especially when mounting drives, and keeping track of who owns what is critical to making sure everything works smoothly. 

### Network
Docker defines networks containers that are also isolated from the system (usually IP of 17.x.x.x) which is the default `bridge` mode. To allow different services to communicate with one another you can also [define networks](https://docs.docker.com/network/network-tutorial-standalone/) within the yaml file and loop in each container to that network. Another option is to run the container in `host` mode in which case the container will use the network of the host device without defining its own network.

Not relevant anymore as I'm not running Plex in a container.
For running Plex it's typically recommended (and easier) to run it in `host` mode as the server will automatically know how to pass the proper port. However, for some reason `host` networking was not working for me so I had to go through `bridge` mode.
To forward a port in `bridge` mode you need to define two settings:
1. Environment variable `ADVERTISE_IP` which uses that IP and port to communicate with the router. (NOTE: `ADVERTISE_IP` only functions properly on the official Plex image. However, there is a setting in Plex to set this value in Settings > Network > Custom server access URLs)
2. Port forwarding on the router set to expose port 32400 to the IP and port defined above.  
Finally, it's a good idea to set Settings > Network > LAN Networks to contain all local IPs. As the Plex IP will be defined as the container IP, it often does not see devices on the local network as local leading to direct play not working properly. This setting can take care of that issue (e.g `192.168.0.0/24` to define all IPs 192.168.0.x to be local).

### Hardware transcoding
When using direct play in a local network there isn't a ton of overhead, but accessing Plex remotely or using various devices can lead to the need for transcoding (effectively changing the video format on the fly). Most CPUs can handle a bit using software, but is highly limited. GPUs can use hardware acceleration to transcode streams much more efficiently, so much so that a simple integrated graphics card can handle dozens of streams. 
The first thing to do is to make sure that the iGPU is properly detected in Linux and using the correct drivers (`i915`). There are various ways of checking, such as `hwinfo` or `lspci` as well as making sure there are files within `/dev/dri`. In addition, some iGPUs may need to be set to `ENABLED` within the BIOS and/or set as the main display adapter especially if there is another dedicated GPU. Finally, you may also need to use `modprobe i915` command to actively use the correct driver.

11/18/23 I cannot get hardware transcoding to work properly. I have the iGPU enabled in the BIOS, the drivers working properly as seen through various ways to check, I have a `/dev/dri` folder with files in them in the local machine, I have:
- Edited the permissions to `/dev/dri`
- Made sure to correctly pass the device in the docker container
- Tried passing it in different ways, tried different versions of the official Plex image as well as the Linuxserver image
- Mapping the `/transcode` path which should be deprecated
- etc. 
and have not been able to get the device passed properly to the container (i.e. `/dev/dri' does not exist in the container and the device does not show up in Plex Settings > Transcoding). At this point I'm thinking I may need to wait for better support for Alderlake architecture, but will revisit in the future. One drastic(?) idea I have is to actually install Plex rather than running it in a Docker container, but I would prefer for everything to be in Docker. 

12/6/23 Turns out the Docker image of Plex, both the official image and the linuxserver image, can't seem to get the hardware transcoding to work but installing it directly on the computer works fine. I'm still guessing it's some permissions error, but it could also be because the CPU I'm using is fairly new and the Docker images may not support it. 

