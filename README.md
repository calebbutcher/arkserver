![GitHub Workflow Status](https://img.shields.io/github/workflow/status/calebbutcher/arkserver/dockerhubci)![Docker Pulls](https://img.shields.io/docker/pulls/impunitus/arkserver)![GitHub](https://img.shields.io/github/license/calebbutcher/arkserver)

# arkserver
Container to run an Ark Server, built off steamcmd base image with arkmanager

## Overview

This is an image for running an ARK: Survival Evolved server in a Docker container. It is heavily influenced by [thmhoag](https://github.com/thmhoag)'s [thmhoag/arkserverr](https://github.com/thmhoag/arkserver). We are also using [FezVrasta](https://github.com/FezVrasta)'s [arkmanager](https://github.com/FezVrasta/ark-server-tools) (ark-server-tools) to manage a single-instance ARK: Survival Evolved Server inside a docker container.

We are inheriting from [calebbutcher/steamcmd](https://github.com/calebbutcher/steamcmd) to ensure we have the latest version of `steamcmd`

For additional information on `arkmanager`, see: https://github.com/FezVrasta/ark-server-tools

### Features
* Auto install
* Configuration via Environment Variables
* Easy crontab manipulation for automated backups, updates, daily restarts, weekly dino wipes, etc
* Simple volume structure for server files, config, logs, backups, etc
* All features present in `arkmanager` 

### Tags
* latest

## Usage

### Pulling container
```bash
docker pull calebbutcher/arkserver:latest
```

### Running the ARK Server
To run a basic server with no configuration changes:
```bash
$ docker run -d \
    -v steam:/home/steam/Steam \  # mounted so that workshop downloads are persisted
    -v ark:/ark \  # mounted as the directory to contain the server/backup/log/config files
    -p 27015:27015 -p 27015:27015/udp \  # steam query port
    -p 7778:7778 -p 7778:7778/udp \  # gameserver port
    -p 7777:7777 -p 7777:7777/udp \ # gameserver port
    impunitus/arkserver
```

### docker-compose (recommended, [see linuxserver's docs for more info](https://docs.linuxserver.io/general/docker-compose))
```yaml
version: "3.4"
services:
  ark-server:
    container_name: arkserver
    image: calebbutcher/arkserver:latest
    restart: unless-stopped
    environment:
        - PUID=${PUID} # default user id, defined in .env
        - PGID=${PGID} # default group id, defined in .env
        - TZ=${TZ} # timezone, defined in .env
    volumes:
        - /dockerMounts/steam:/home/steam/Steam # Mounted for workshop and mod persistance
        - /dockerMounts/ark:/ark
    ports:
        - 7777:7777/tcp
        - 7777:7777/udp
        - 7778:7778/tcp
        - 7778:7778/udp
        - 27015:27015/tcp
        - 27015:27015/udp
```

If the exposed ports are modified (in the case of multiple containers/servers on the same host) the `arkmanager` config will need to be modified to reflect the change as well. This is required so that `arkmanager` can properly check the server status and so that the ARK server itself can properly publish its IP address and query port to steam.

## Environment Variables

A set of required environment variables have default values provided as part of the image. You can edit these after first run of the container assuming you have persistant storage. More on that below:

| Variable | Value | Description |
| - | - | - |
| am_ark_SessionName | `Ark Server` | Server name as it will show on the steam server list |
| am_serverMap | `TheIsland` | Game map to load |
| am_ark_ServerAdminPassword | `k3yb04rdc4t` | Admin password to be used via ingame console or RCON |
| am_ark_MaxPlayers | `70` | Max concurrent players in the game |
| am_ark_QueryPort | `27015` | Steam query port (allows the server to show up on the steam list) |
| am_ark_Port | `7778` | Game server port (allows clients to connect to the server) |
| am_ark_RCONPort | `32330` | RCON port |
| am_arkwarnminutes | `15` | Number of minutes to wait/warn players before updating/restarting |
| am_arkflag_crossplay | `false` | Allow crossyplay with Players on Epic |

### Adding Additional Variables

Any configuration value that is available via `arkmanager` can be set using an environment variable. This works by taking any environment variable on the container that is prefixed with `am_` and mapping it to the corresponding environment variable in the `arkmanager.cfg` file. 

For a complete list of configuration values available, please see [FezVrasta](https://github.com/FezVrasta)'s great documentation here: [arkmanager Configuration Files](https://github.com/FezVrasta/ark-server-tools#configuration-files)

## Volumes

This image has two main volumes that should be mounted as named volumes or host directories for the persistence of the server install and all related configuration files. More information on Docker volumes here: [Docker: Use Volumes](https://docs.docker.com/storage/volumes/)

| Path | Description |
| - | - |
| /home/steam/Steam | Directory of steam cache and other steamcmd-related files. Should be mounted so that mod installs are persisted between container runs/restarts |
| /ark | Directory that will contain the server files, config files, logs and backups. More information below |

### Subdirectories of /ark

Inside the `/ark` volume there are several directories containing server related files:

| Path | Description |
| - | - |
| /ark/backup | Location of the zipped backups genereated from the `arkmanager backup` command. Compressed using bz2. |
| /ark/config | Location of server config files. More information: |
| /ark/log | Location of the arkmanager and arkserver log files |
| /ark/server | Location of the server installation performed by `steamcmd`. This will contain the ShooterGame directory and the actual server binaries. |
| /ark/staging | Default directory for staging game and mod updates. Can be changed using in `arkmanager.cfg` |

### Editing arkmanager.cfg 
I had two primary reasons for creating my own image for this was due to how the information in arkmanager.cfg was being overwriten on container restarts. The `run.sh` was taking values from the initial config and ignoring if the file it was creating already existed or not. I wanted to allow that data to persist so modified the script slightly to do so. The other issue was the base image of steamcmd was built on an out of date build of Ubuntu, and I wished to standardize this container on the same build of Ubuntu I run in my labs currently. 
