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
        {% include figure.liquid loading="eager" path="assets/img/2025-11-20-nextcloud-npm(02).jpg" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    My Nginx Proxy Manager instance
</div>

There may be better options here, but I'm a sucker for ease-of-use, and NPM fits the bill.