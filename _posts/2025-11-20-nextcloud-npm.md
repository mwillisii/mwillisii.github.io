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
    My "home lab," complete with dog hair and a rat's nest of cables! I really should tidy this up...
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

There may be better options here, but I'm a sucker for ease-of-use, and NPM fits the bill. So off I go searching for guides on installing Nextcloud behind NPM, and I came across [this amazing guide from the Nextcloud AIO GitHub wiki](https://github.com/nextcloud/all-in-one/discussions/4240) (thanks [4lexRed](https://github.com/4lexRed)!). In this post, 4lexRed includes a Docker compose file, which is used by Docker to provision both Nextcloud and NPM, as well as Portainer (a web-hosted GUI to manage Docker containers, because I loathe managing Docker from the CLI), all from one Docker command. There are a few changes that need to be made to the compose file for my environment, all of which were mentioned in their instructions. [Here's my compose file](https://markw.wtf/assets/misc/2025-11-20-nextcloud-npm(docker-compose).yml) I ended up with after modifying it to suit my environment. Some of the changes were needed because of recent Nextcloud and/or Docker updates.

Now, to create the Docker containers via the Docker compose file. As previously mentioned, I'm running Debian on my home server. I'm the only one in there managing it, so I like to use my home folder for just about everything because I'm lazy. I ran into some issues installing Docker via APT, so I suggest following their official documentation depending on your distro: [Docker install — supported platforms](https://docs.docker.com/engine/install/#supported-platforms)

Next we want to make sure our system is sync'd and up-to-date:

````markdown
sudo apt update && sudo apt upgrade -y
````

Now we can finally get into the meat and potatoes. Create the compose file and copy/paste your modified docker-compose.yml from 4lexRed's guide:

````markdown
nano docker-compose.yml
````

Press ctrl+x, then press "Y" to save the file. Then, we run Docker compose from the same directory as the docker-compose.yml file:

````markdown
docker compose up -d
````

Docker handles all the provisioning of the containers from there! Now we can navigate to our Nextcloud instance: https://your.host.ip.address:8080 (replace the IP with your host's IP, of course)

Make sure to save your AIO interface password, though if you forget to or lose it here is a handy command to retrieve it:

````markdown
sudo docker exec nextcloud-aio-mastercontainer grep password /mnt/docker-aio-config/data/configuration.json
````

After logging in, you'll be prompted to enter the domain name of your Nextcloud instance. Enter your domain and click submit; the domain check will fail, and this is normal. The reason for this is two-fold: our reverse proxy (Nginx Proxy Manager in our case) is not yet configured. Secondly, we are not using host networking in Docker, as is detailed in the compose file. This means our Docker containers do not have access to the host's network stack, which I like to configure this way for security reasons. We want our reverse proxy handling all of that anyway!

So, let's jump into NPM and get that configured. We will also utilize Portainer, which was another container created via the compose file, to get the IP of the new container spun up by Nextcloud for the domain check process. We are going to forward traffic from our Nextcloud domain to the domain check container, which will make our Nextcloud instance happy!

First, we'll need Docker's IP for the domain check container to give to NPM. Open Portainer at https://your.host.ip.address:9443. Set your password and then navigate to the "local" environment, then navigate to Containers - you will see a list of all containers currently running in Docker. Find the container called "nextcloud-aio-domaincheck" and make note of its IP, which should start with 172.18.0.x (I'm going to use 172.18.0.5 for this example). If you don't see this container, then re-run the domain check from your Nextcloud instance, which will fail again.

Open NPM at http://your.host.ip.address:81 and set your admin credentials. From the dashboard, go to Proxy Hosts, then click "Add Proxy Host." Use the following options, replacing the domain name with your own personal Nextcloud domain:

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/2025-11-20-nextcloud-npm(03).png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/2025-11-20-nextcloud-npm(04).png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/2025-11-20-nextcloud-npm(05).png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>

With that done, we'll go back to our Nextcloud instance and run the same domain check again. Success! Go ahead and finish setting up your Nextcloud instance with your preferred settings. Once setup, there will be a new container called "nextcloud-aio-apache." Open Portainer again and you'll see all the containers created by Nextcloud, or use the command "docker ps" to see them in the console.

In NPM, we need to edit the previous Proxy Host entry created for the domain check container:

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/2025-11-20-nextcloud-npm(06).png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/2025-11-20-nextcloud-npm(07).png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>
<div class="caption">
    Edit the previous entry created for nextcloud-aio-domaincheck. You can use either the IP or the hostname for the container, although it's suggested to use the hostname, as the IP can change anytime the containers are rebuilt, which occurs during Nextcloud updates.
</div>

And... we're just about done! The only thing left to do is to install our certificate. Just how to do this will vary depending on your domain registrar. It's also possible to automate this with certbot, although that could be its own separate post. For now, let's install the SSL certificate from our registrar - Porkbun in my case. They provide an SSL bundle with a public key, private key, and intermediate certificate. For our use, we only need the private.key.pem and public.key.pem files.

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/2025-11-20-nextcloud-npm(08).png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/2025-11-20-nextcloud-npm(09).png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>

With the certificate uploaded, let's assign it to our Nextcloud proxy entry:

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/2025-11-20-nextcloud-npm(10).png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/2025-11-20-nextcloud-npm(06).png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/2025-11-20-nextcloud-npm(11).png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>

Finally, we need to add a few lines of custom Nginx configuration that Nextcloud requires when hosted behind NPM:

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/2025-11-20-nextcloud-npm(12).png" class="img-fluid rounded z-depth-1" %}
    </div>
</div>