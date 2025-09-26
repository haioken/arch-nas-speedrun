# Speedrun NAS setup

This is an ArchLinux-centric guide, you'll need to change some things to your package manager if you use an inferior distribution ;o)
This guide will assist with getting a base ArchLinux installation running with docker containers for:
- Sonarr (For automation of TV show monitoring and queueing of NZBs in NZBGET)
- Radarr (For automation of Movie monitoring and queueing of NZBs in NZBGET)
- NZBGet (This receives NZB files, which tell it where in a newsgroup server to download media from. It then downloads them, allowing Sonarr/Radarr to rename and arrange them for JellyFin)
- JellyFin (Media Server)
- JellySeerr (Media request framework. This provides a curated way for people to "request" new TV shows, movies etc to be downloaded, while allowing you some control)

Please note that this installation configures the system with NO GUI. maintenance is done via SSH on a terminal.
There will be web UIs for the various apps, but no system GUI, it is designed to run headless (no monitor, keyboard, mouse)
This assumes a level of linux competency, and a clean ArchLinux install, if you are installing to an existing system you may need to tweak some things (like UID and GID if they're not already appropriately set)
Don't stress if you're not a Linux native, I'm happy to help with any questions, and if stuck, given availability, I'll even pop around and assist.

Additional assumptions are:

- You will have or obtain a domain name that you can control the DNS for
- You will configure your domain to use CloudFlare for your DNS provider
- You will add the either a wildcard subdomain or the following subdomains A record(s) pointing to your IP:
  - sonarr
  - radarr
  - dl
  - tv
  - jellyseerr
- You will add port forwards on your router to forward 443 (HTTPS) to your NAS
- Your primary storage for your media is in `/mnt/raidarray`, Adjust the following and the docker-compose.yaml file accordingly.
- You're planning to pirate primarily from NZBs and newsgroups, which requires:
  - A newsgroup provider subscription (Recommend frugalnews, ~$6 AUD/mo) - This is where you download from
  - A newsgroup indexer subscription  (Recommend nzbfinder.ws, $15 USD/yr) - This contains nzb files, the newsgroup equivalent of a `.torrent` file or magnet link

I *STRONGLY* recommend having a separate "root" device to your bulk storage, and have written this to reflect that configuration.
This means that your OS is effectively on a separate drive.

For disk configuration, I recommend:
- 1x nvme SSD for root, 256Gb OK, recommend 512Gb. Our docker containers will reside here, so needs some space.
- Any number of mechanical disks for bulk storage. These should ideally be configured with software RAID (Redundant array of Independent disks) for a level of disaster mitigation.

If your bulk storage disks will ALL be the same size (IE: All 4tb, or all 6tb, or all 8tb), use a standard "mdraid" type, easier to maintain
If however your bulk storage disks will differ in size, (IE: 2x6tb + 1x2Tb + 1x4Tb + 1x12Tb) I recommend BTRFS which allows for RAID with different disk sizes.
bcachefs (which I use) also does this, but it's HIGHLY experimental, and not recommended without being willing to undertake in-depth debugging.
BTRFS and bcachefs also have the added advantage of being able to add or remove disks at a whim and resize the partition to match.
I recommend a btrfs RAID1 for simplicity and ease of use. This means you should be able to lose 1 disk without losing any data.
You can then replace that disk, rebalance and be secure in that your data is still present.
So you can (for example) have your existing array, add a new 12Tb disk, and then do something like:
```
btrfs device add /dev/sdd /mnt/raidarray
```
Basic guide to BTRFS: https://wiki.tnonline.net/w/Btrfs/Getting_Started

Installation of Arch can be very difficult or quite simple depending on the method you use.

For ease of use, I recommend grabbing the latest Arch installation ISO - https://mirror.aarnet.edu.au/pub/archlinux/iso/2025.09.01/archlinux-x86_64.iso

Download Rufus, and create a bootable USB using the above ISO - https://rufus.ie/en/

Attach the USB to your NAS, select the USB as the boot device, and start up. You'll end up at a terminal with instructions to login.

Login, and install the `archinstall` package, which will allow you to run the modern Arch installer and install to your NAS.

If you want to download primarily with torrent, it's more complicated.
Torrents don't use a standard indexing format like newznab that NZBs use, so you need something that pretends to be an indexer for Sonarr and Radarr, and talks to torrent providers.
Typically this is another app called `Jackett`, which can also be added to your docker-compose.yaml file.
I have added a qBittorrent container to the docker-compose.yaml, but replicating my config will not automate torrent downloads
I have it there for one offs, for example an Anime episode that only have english dubs on NZB - it's rare, but it happens.

Create user and group called "media", add the "media" user to the media group:
    sudo groupadd --gid 1001 -f media && sudo useradd --uid 1001 -m -g media media

Install prerequisites
```
sudo pacman -S \
    openssh \
    docker \
    docker-compose \
    nginx \
    certbot \
    python-certbot-dns-cloudflare \
    curl \
    fail2ban \
    jq \
    net-tools \
    samba
```

Create some necessary folders and files with appropriate permissions
```
sudo mkdir -p \
    /etc/letsencrypt
    /etc/nginx/sites{available,enabled} \
    /mnt/raidarray/media \
    /mnt/raidarray/container-data \
    /mnt/raidarray/container-data/docker-compose
sudo chmod 600 /etc/letsencrypt/cloudflare.ini
sudo touch \
    /etc/letsencrypt/cloudflare.ini \
    /etc/nginx/nginx.com
sudo chown media:media \
    /mnt/raidarray/media \
    /mnt/raidarray/container-data \
    /mnt/raidarray/container-data/docker-compose
```
Configure a cloudflare API token in your cloudflare control panel
Add the following to /etc/letsencrypt/cloudflare.ini:
    dns_cloudflare_api_token = YOUR_CLOUDFLARE_API_TOKEN

Request the (free) SSL from letsencrypt, using cloudflare's API to create the DNS verification records as required:
```
sudo certbot certonly \
    --dns-cloudflare \
    --dns-cloudflare-credentials /etc/letsencrypt/cloudflare.ini \
    -d "*.myactual.domain" -d myactual.domain \
    --agree-tos --no-eff-email --email your@email.address
```

Add a basic nginx starting config (See attached `nginx.conf`)
Create configs for nginx in `/etc/nginx/sites-available` (See attached `siteconfig-*.conf`)
These configs will allow traffic for all sites from your local net, but only external access to Jellyfin and Jellyseerr


Enable docker
```
systemctl enable --now docker nginx
```

Create a `/mnt/raidarray/container-data/docker-compose/docker-compose.yaml` file with a list of containers to run. (See attached `docker-compose.yaml`)
Bring up the containers up (This is an all-in-one command, downloads, builds, runs)
```
docker compose -f /mnt/raidarray/container-data/docker-compose/docker-compose.yaml up -d
```

When you get that far, I can help with configuring nzbget, sonarr, radarr and Jellyfin.
If you go the `Jackett` route, you're shit out of luck and doing it yourself :o)
