### Homelab Project Guide
# Private Cloud Infrastructure: Nextcloud, Navidrome, and Tailscale

---

## Overview

This project documents the setup of a personal homelab server running on a laptop with Ubuntu. The stack replaces commercial cloud storage and music streaming with self-hosted alternatives, accessible from any device over a private Tailscale network.

All services run in Docker containers managed with Docker Compose. No cloud subscriptions. No data on third-party servers.

---

## Stack

- Ubuntu (Intel CPU, Nvidia GPU)
- Docker and Docker Compose
- Nextcloud: file storage and sync
- Navidrome: music streaming
- PostgreSQL: Nextcloud database
- Tailscale: private remote access

---

## Use Case

The server stores and serves the following file types:

- Documents: txt, pdf, docx, xls, sql
- Code: py, cpp
- Media: flac, png, img, mp4

Music files are streamed via Navidrome using the Subsonic API. The Substreamer app on iOS connects to Navidrome over Tailscale for on-the-go playback.

---

## Prerequisites

- A machine running Ubuntu (this guide uses a laptop with an ethernet connection)
- SSH access from another machine on the same network
- A Tailscale account at tailscale.com

---

## Step 1: Connect via SSH

Find the machine's local IP address. Run this on the Ubuntu machine:

```bash
ip a
```

Look for the `inet` address under your active network interface (eth0, enx..., or wlp...). It will look like `192.168.x.x` or `10.0.0.x`.

Then from another machine on the same network:

```bash
ssh your_username@192.168.x.x
```

If SSH is not installed on the Ubuntu machine, install it first:

```bash
sudo apt install openssh-server -y
sudo systemctl enable ssh
sudo systemctl start ssh
```

---

## Step 2: Update the System

```bash
sudo apt update && sudo apt upgrade -y
```

---

## Step 3: Install Docker

```bash
sudo apt install docker.io -y
sudo usermod -aG docker $USER
newgrp docker
```

The version of Docker Compose in apt is outdated on Ubuntu with Python 3.12. Install the latest binary directly:

```bash
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-linux-x86_64" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

Verify both are working:

```bash
docker run hello-world
/usr/local/bin/docker-compose --version
```

---

## Step 4: Create the Folder Structure

```bash
mkdir ~/server && cd ~/server
mkdir -p nextcloud/data navidrome/data navidrome/music postgres/data
```

> Place your music files in `~/server/navidrome/music`. Navidrome scans this folder automatically.

---

## Step 5: Write the Docker Compose File

```bash
cd ~/server && nano docker-compose.yml
```

Paste the following:

```yaml
services:
  db:
    image: postgres:15
    container_name: postgres
    restart: unless-stopped
    environment:
      POSTGRES_DB: nextcloud
      POSTGRES_USER: nextcloud
      POSTGRES_PASSWORD: nextcloud123
    volumes:
      - ./postgres/data:/var/lib/postgresql/data

  nextcloud:
    image: nextcloud
    container_name: nextcloud
    restart: unless-stopped
    ports:
      - "8080:80"
    depends_on:
      - db
    environment:
      POSTGRES_HOST: db
      POSTGRES_DB: nextcloud
      POSTGRES_USER: nextcloud
      POSTGRES_PASSWORD: nextcloud123
    volumes:
      - ./nextcloud/data:/var/www/html

  navidrome:
    image: deluan/navidrome
    container_name: navidrome
    restart: unless-stopped
    ports:
      - "4533:4533"
    volumes:
      - ./navidrome/data:/data
      - ./navidrome/music:/music
    environment:
      ND_SCANSCHEDULE: 1h
      ND_LOGLEVEL: info
      ND_MUSICFOLDER: /music
```

Save the file with `Ctrl+X`, then `Y`, then `Enter`.

---

## Step 6: Start the Containers

```bash
cd ~/server && /usr/local/bin/docker-compose up -d
```

Docker pulls the images and starts all three containers. Verify they are running:

```bash
docker ps
```

You should see `postgres`, `nextcloud`, and `navidrome` all listed with status `Up`.

---

## Step 7: Configure Nextcloud

Get the machine's local IP:

```bash
ip a
```

Open a browser and go to `http://192.168.x.x:8080`. The Nextcloud setup page loads.

Fill in the fields:

- Admin username: your choice
- Admin password: your choice
- Database: select PostgreSQL
- Database user: `nextcloud`
- Database password: `nextcloud123`
- Database name: `nextcloud`
- Database host: `db`

Click Install. Setup takes one to two minutes.

---

## Step 8: Configure Navidrome

Open a browser and go to `http://192.168.x.x:4533`. Create an admin account on first launch.

Navidrome scans the music folder automatically based on the `ND_SCANSCHEDULE` set in the compose file. To trigger a manual scan, go to Settings in the Navidrome UI.

---

## Step 9: Set Up Tailscale

Install Tailscale on the Ubuntu machine:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

Start Tailscale and authenticate:

```bash
sudo tailscale up
```

A URL prints to the terminal. Open it in a browser and log in to your Tailscale account. The machine joins your Tailscale network.

Get the Tailscale IP:

```bash
tailscale ip -4
```

This returns a `100.x.x.x` address. Install Tailscale on your other devices (phone, Mac, etc.) and connect them to the same account.

---

## Step 10: Add Tailscale IP as Trusted Domain in Nextcloud

Nextcloud blocks access from unrecognized addresses. Add the Tailscale IP as a trusted domain:

```bash
docker exec -it nextcloud php occ config:system:set trusted_domains 1 --value=100.x.x.x
```

Replace `100.x.x.x` with your actual Tailscale IP.

---

## Accessing Your Services

From any device connected to your Tailscale network, open a browser and go to:

- Nextcloud: `http://100.x.x.x:8080`
- Navidrome: `http://100.x.x.x:4533`

For music on mobile, use a Subsonic-compatible app. This setup was tested with Substreamer on iOS. Point it to `http://100.x.x.x:4533` with your Navidrome credentials.

For file sync on desktop, install the Nextcloud desktop client from nextcloud.com/install and point it to `http://100.x.x.x:8080`.

---

## Useful Docker Commands

Stop all containers:

```bash
cd ~/server && /usr/local/bin/docker-compose down
```

Start all containers:

```bash
cd ~/server && /usr/local/bin/docker-compose up -d
```

View running containers:

```bash
docker ps
```

View logs for a specific container:

```bash
docker logs nextcloud
docker logs navidrome
docker logs postgres
```

Update all images:

```bash
/usr/local/bin/docker-compose pull
/usr/local/bin/docker-compose up -d
```

---

## Notes

- This setup does not include HTTPS. For local and Tailscale access only, HTTP is acceptable. Adding a reverse proxy like Nginx Proxy Manager with SSL is a future improvement.
- The server goes offline when the laptop is shut down or suspended. This is intentional for this use case.
- PostgreSQL is used instead of SQLite for better long-term performance as the file count grows.
- Navidrome is used instead of Jellyfin for its low resource footprint and purpose-built audio streaming.
