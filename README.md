# Plex
 Plex Media Server w/ Docker Compose

## TODO
- Remote access
- Hardware transcoding
- Cloudflare tunnel
- Double check Radarr / Sonarr functionality
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

There was definitely a steep learning curve. I still don't quite understand all the underlying mechanisms that are operating under the hood, but I'll try to lay out its implementation as simply as possible here. Read through the [Docker website](https://docs.docker.com/get-started/overview/) or watch this [youtube video](https://www.youtube.com/watch?v=aLipr7tTuA4) to get a better sense of what Docker is doing. 

To start, we need to install the docker engine, which can be done by following the instructions [here](https://docs.docker.com/engine/install/ubuntu/). I also installed Docker Desktop, mostly on accident, but have found it to be really handy for keeping containers organized and inspecting the container instances.

I used docker compose as it makes laying out the services incredibly easy. I also preferred it over CLI as it was cleaner / easier to read. There is a ton of documentation on [general usage](https://docs.docker.com/compose/compose-file/compose-file-v3/) as well as [specific examples](https://docs.linuxserver.io/general/docker-compose/) for setting up services.

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

### Services
In order of the services that are on my Docker compose yaml file and specific settings:
1. [expressvpn](https://hub.docker.com/r/polkaned/expressvpn) - VPN
   - Requires an activation code
   - Needs to be a `NET_ADMIN`
   - `/dev/net/tun` device allows for the VPN networking
   - List all ports for services using the VPN network  
2. [plex](https://hub.docker.com/r/linuxserver/plex) - Plex media server
   - Use `network_mode:host` if possible. Otherwise, need to map all the ports listed
   - Requires mapping of `/movies` and `/tv`
   - `/dev/dri` device enables hardware transcoding
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

Note I have every service other than Plex itself on the VPN as I figured it wouldn't hurt. You can also put Plex on the VPN but accessing outside your local network might get more complicated. 
To put a service on the VPN simply add `network_mode: service:expressvpn`. You should also add a `depends_on:` clause so it starts up after the VPN. If not using a VPN (or if the VPN is standalone and not dockerized) you need to setup an internal network such that each service can access one another as they will be isolated otherwise.  

### Setup
#### Settings
#### Cloudflare tunneling
#### Webhooks

## Troubleshooting
### Permissions
### Network
### Hardware transcoding



