---
layout: post
title: setting up nextcloud aio with nginx proxy manager and portainer
date: 2025-11-20 12:00:00
description: a guide for configuring nextcloud aio to function behind nginx proxy manager, including a docker compose file for setting up nextcloud, npm, and portainer
tags: docker nextcloud nginx proxy npm portainer
categories: 
featured: true
---

It seems we are in a dark age of the Internet where enshitification is taking over everything. I've used Google for just about everything over the past decade or so - email, calendar, contacts, photos, etc. These services used to be solid, albeit not exactly privacy-focused. Not so much these days... AI is invading every service known to man, Google is snooping on everything you do to train their models, and who knows what else. So I'm finally taking the overdue plunge to self-host as much as possible from my own home.

Both my wife and I were heavy Google Photos users - we have years and years of photos backed up. My first goal in this self-hosting adventure is to simply de-Google. Email is fairly easy - I've switched to mailbox.org, which allows using your own domain name for just a few bucks a month. Drive and Photos, on the other hand... not so simple. I decided on Nextcloud since it includes a replacement for all the Google services I used - photos, calendars, contacts, and notes. They also have both Android and iOS apps for auto-upload, which is important because my wife is not quite as tech-savvy, so I want the switch to be seamless for her.

First things first, I needed hardware. I used to run a Raspberry Pi for Pi Hole and other small projects, but I knew I needed to upgrade. After a little research, I ended up going with [this GMKtec mini-PC](https://www.gmktec.com/products/nucbox-g3-plus-enhanced-performance-mini-pc-with-intel-n150-processor?variant=22e83d5e-d20e-41a7-994f-e2aa05e5b6c8) with 16 GB of RAM and 1 TB NVME SSD. I figure 1 TB will be enough storage to get me started (even with all our photos, we only utilized ~200 GB of Photos/Drive storage). If I end up needing more storage in the future... well I don't know, that's a problem for future me. But this will hold us over for a while.

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/2025-11-20-nextcloud-npm(01).jpg" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    My "home lab," complete with dog hair and a rat nest of cables! I really should tidy this up...
</div>

This mini PC came with Windows, but I'm not a Microsoft fan, so I wiped it and installed Debian. Now that I've got all that out of the way, it's time to install Nextcloud. I'm not reinventing the wheel here - there are lots of guides out there on how to install Nextcloud. The official method from Nextcloud themselves is to [install Nextcloud AIO via Docker](https://github.com/nextcloud/all-in-one). This is all well and good, and I even utilized this method for a few months while I got my feet wet. However, the problem I ran into is in regards to the rest of my hosted services on the same server: any service that uses the same port as Nextcloud is black-holed, since Nextcloud "hijacks" the port on the host to redirect it to the Docker container. And since Nextcloud uses both port 80 (http) and 443 (https), this essentially broke any other service using those ports. Considering over the past few months I've setup several self-hosted services on this box (Pi Hole, Plex, Sonarr, Radarr, Miniflux, etc.), this was problematic.

This is where a reverse proxy comes in. Essentially, it takes traffic coming into the server and redirects it to the appropriate service. I went with Nginx Proxy Manager, since it has a nice little webpage GUI that makes configuration super easy.

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/2025-11-20-nextcloud-npm(02).png" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    My Nginx Proxy Manager instance
</div>

There may be better options here, but I'm a sucker for ease-of-use, and NPM fits the bill. So off I go searching for guides on installing Nextcloud behind NPM, and I came across [this amazing guide from the Nextcloud AIO GitHub wiki](https://github.com/nextcloud/all-in-one/discussions/4240) (thanks [4lexRed](https://github.com/4lexRed)!) In this post, 4lexRed includes a Docker compose file, which is used by Docker to provision both Nextcloud and NPM, as well as Portainer (a web-hosted GUI to manage Docker containers, because I loathe managing Docker from the CLI), all from one Docker command. There are a few changes that need to be made to the compose file for my environment, all of which were mentioned in their instructions.


```yaml
# sample compose for a nextcloud all in one container, running behind a nginx proxy manager
# (optionally) uses portainer to give a better overview of the docker environment and the running containers
#
# the network here is using fixed ip addesses for some hosts, as the nextcloud setting
# NC_TRUSTED_PROXIES needs an ip address to relate to.
#
# Note: the data of npm and portainer container are stored in local folder instead of volumes.
#       you need to create these folders or comment these lines
#
# questions? post to https://github.com/nextcloud/all-in-one/discussions/4240
#
# 4lexRed

services:
  npm:
    container_name: npm
    image: 'jc21/nginx-proxy-manager:latest'
    restart: 'always'
    ports:
      # These ports are in format <host-port>:<container-port> - both MUST be open
      - '80:80' # Public HTTP Port  - [I myself disable port 80 after the setting up the ssl certificates - as most traffic is https nowadays]
      - '443:443' # Public HTTPS Port

      # Admin Web Port - disable as soon as the NPM is running and you set up a route on localhost and port 81
      # see here https://github.com/NginxProxyManager/nginx-proxy-manager/issues/1658
      #
      - '81:81'   # enable if you want to access the npm service via [your.server.ip.address:81]

      # Add any other Stream port you want to expose
      # - '21:21' # FTP

    volumes:
      # These volumes are in format <local-dir>:<container-dir>
      - '/home/mark/nginx-proxy-manager/data:/data'
      - '/home/mark/nginx-proxy-manager/letsencrypt:/etc/letsencrypt'
      # - '/opt/npm_logs/etc:/etc/nginx/conf.d:ro'  # for ip ban
      # - '/opt/npm_logs/logs:/var/log/nginx:rw'  # for ip-ban

    environment:
      # Uncomment this if you want to change the location of the SQLite DB file within the container
      # DB_SQLITE_FILE: "/data/database.sqlite

      # changing the user id may lead to PORT 80 not being reachable
      # PUID: 1000
      # PGID: 1000

      # DISABLE_IPV6: 'true'  # Uncomment this if IPv6 is not enabled on your host (WAN interface)
      TZ: "America/New_York"   # the timezone of the server

    healthcheck:
      test: ["CMD", "/usr/bin/check-health"]
      interval: 10s
      timeout: 3s

    # Either use netmorke mode or networks.
    # Both at the same time is not possible.
    # Nextcloud AIO recommends in their guide to set it to network_mode: host.
    # https://github.com/nextcloud/all-in-one/blob/main/reverse-proxy.md#adapting-the-sample-web-server-configurations-below

    # But I dislike to do this and instead have all traffic going through the npm using the networks approach.

    # network_mode: host  # for details in the docs see https://docs.docker.com/compose/compose-file/05-services/#network_mode

    networks: # Is needed when you want to create the nextcloud-aio network with ipv6-support using this file, see the network config at the bottom of the file
      nextcloud-aio:
        aliases:
         - npm
        ipv4_address: '172.18.0.2'

# # # # # # # # # # # #
# portainer usage is optional. you can delete these lines if you are fine with monitoring the state via the cli.

  portainer:
    container_name: portainer
    image: portainer/portainer-ce:2.20.2
    restart: always
    security_opt:
      - no-new-privileges:true

    # ports:
    # when uncommented, these ports are publicly accessable [bypassing the npm] by calling [your.server.ip.address:9000]
    # so disable them when the npm is set up to forward the port 9443 when calling docker.myhostname.com (or whichever your subdomain for the portainer is)
      # - 9000:9000
      # - 9443:9443

    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - '/docker_data/portainer_data:/data'  # the data folder of the portainer

    networks:
      nextcloud-aio:
        aliases:
         - portainer
        ipv4_address: '172.18.0.3'


# # # # # # # # # # # #
# The nextcloud All in one container
# see https://github.com/nextcloud/all-in-one#faq

  nextcloud:
    image: nextcloud/all-in-one:latest
    init: true
    restart: always
    container_name: nextcloud-aio-mastercontainer  # This line is not allowed to be changed as otherwise AIO will not work correctly

    volumes:
      - nextcloud_aio_mastercontainer:/mnt/docker-aio-config # This line is not allowed to be changed as otherwise the built-in backup solution will not work
      - /var/run/docker.sock:/var/run/docker.sock:ro # May be changed on macOS, Windows or docker rootless. See the applicable documentation. If adjusting, don>

    ports:
      - 8080:8080  # this is the port of the admin interface used for the setup of NC-AIO
      # - 5080:80 # Can be removed when running behind a web server or reverse proxy (like Apache, Nginx, Cloudflare Tunnel and else). See https://github.com/n>
      # - 8443:8443 # Can be removed when running behind a web server or reverse proxy (like Apache, Nginx, Cloudflare Tunnel and else). See https://github.com>

    environment:
      # - NEXTCLOUD_TRUSTED_DOMAINS = nextcloud.myhostname.com   # the dns name under which your nextcloud should be accessable

      - APACHE_PORT=11000 # Is needed when running behind a web server or reverse proxy (like Apache, Nginx, Cloudflare Tunnel and else).
                          # See https://github.com/nextcloud/all-in-one/blob/main/reverse-proxy.md

      - APACHE_IP_BINDING=0.0.0.0 # Should be set to '0.0.0.0' when running behind a web server or reverse proxy (like Apache, Nginx, Cloudflare Tunnel and els>
                                  # See https://github.com/nextcloud/all-in-one/blob/main/reverse-proxy.md
                                  # If set to 0.0.0.0 the apache will listen to all http requests, no matter what their target ip address is

      - NEXTCLOUD_UPLOAD_LIMIT=10G # Can be adjusted if you need more. See https://github.com/nextcloud/all-in-one#how-to-adjust-the-upload-limit-for-nextcloud
      # - NEXTCLOUD_DATADIR=/mnt/nfs-storage/my_folder/nextcloud_data   # Allows to set the host directory for Nextcloud's datadir. ⚠⚠⚠ Warning: do not set or >   # - NEXTCLOUD_MOUNT=/mnt/nfs-storage/my_folder/nextcloud_data2  # Allows the Nextcloud container to access the chosen directory on the host. See https://>

      # settings overwriting the config.php parameters
      - NC_TRUSTED_PROXIES="172.18.0.2"  # this is the NPM proxy ip address in the docker network !

      # - AIO_DISABLE_BACKUP_SECTION=false # Setting this to true allows to hide the backup section in the AIO interface. See https://github.com/nextcloud/all->
      # - BORG_RETENTION_POLICY=--keep-within=7d --keep-weekly=4 --keep-monthly=6 # Allows to adjust borgs retention policy. See https://github.com/nextcloud/a>
      # - COLLABORA_SECCOMP_DISABLED=false # Setting this to true allows to disable Collabora's Seccomp feature. See https://github.com/nextcloud/all-in-one#ho>
      # - NEXTCLOUD_MAX_TIME=3600 # Can be adjusted if you need more. See https://github.com/nextcloud/all-in-one#how-to-adjust-the-max-execution-time-for-next>
      # - NEXTCLOUD_MEMORY_LIMIT=512M # Can be adjusted if you need more. See https://github.com/nextcloud/all-in-one#how-to-adjust-the-php-memory-limit-for-ne>
      # - NEXTCLOUD_TRUSTED_CACERTS_DIR=/path/to/my/cacerts # CA certificates in this directory will be trusted by the OS of the nexcloud container (Useful e.g>
      # - NEXTCLOUD_STARTUP_APPS=deck twofactor_totp tasks calendar contacts notes # Allows to modify the Nextcloud apps that are installed on starting AIO the>
      # - NEXTCLOUD_ADDITIONAL_APKS=imagemagick # This allows to add additional packages to the Nextcloud container permanently. Default is imagemagick but can>
      # - NEXTCLOUD_ADDITIONAL_PHP_EXTENSIONS=imagick # This allows to add additional php extensions to the Nextcloud container permanently. Default is imagick>
      # - NEXTCLOUD_ENABLE_DRI_DEVICE=true # This allows to enable the /dev/dri device in the Nextcloud container. ⚠⚠⚠ Warning: this only works if the '/dev/dr>   # - NEXTCLOUD_KEEP_DISABLED_APPS=false # Setting this to true will keep Nextcloud apps that are disabled in the AIO interface and not uninstall them if t>
      # - TALK_PORT=3478 # This allows to adjust the port that the talk container is using. See https://github.com/nextcloud/all-in-one#how-to-adjust-the-talk->
      # - WATCHTOWER_DOCKER_SOCKET_PATH=/var/run/docker.sock # Needs to be specified if the docker socket on the host is not located in the default '/var/run/d>

    networks: # Is needed when you want to create the nextcloud-aio network with ipv6-support using this file, see the network config at the bottom of the file
      nextcloud-aio:
        aliases:
         - nextcloud
        ipv4_address: '172.18.0.4'

volumes: # If you want to store the data on a different drive, see https://github.com/nextcloud/all-in-one#how-to-store-the-filesinstallation-on-a-separate-dri>
  nextcloud_aio_mastercontainer:
    name: nextcloud_aio_mastercontainer # This line is not allowed to be changed as otherwise the built-in backup solution will not work

networks:
  nextcloud-aio:
    name: nextcloud-aio
    ipam:
      driver: default
      config:
        - subnet: 172.18.0.0/24
```