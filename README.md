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

There was definitely a steep learning curve, but I'll try to lay it out as simply as possible here.
To start, we need to install the docker engine, which can be done by following the instructions [here](https://docs.docker.com/engine/install/ubuntu/). I also installed Docker Desktop, mostly on accident, but have found it to be really handy for keeping containers organized and inspecting the container instances.

I used docker compose as it makes laying out the services incredibly easy. I also preferred it over CLI as it was cleaner / easier to read. There is a ton of documentation on [general usage](https://docs.docker.com/compose/compose-file/compose-file-v3/) as well as [specific examples](https://docs.linuxserver.io/general/docker-compose/) for setting up services.
