Homelab Project Guide
# Remote Upload Music File Pipeline: Syncthing, Navidrome, Rockbox

---

## Overview

This guide extends the Private Cloud Infrastructure build (Nextcloud, Navidrome, Tailscale). That project deployed Navidrome as a self-hosted music streaming server. This build adds Syncthing as a file sync layer between an Rockbox (connected to a MacBook) and the Navidrome music folder on the Ubuntu server.

The result is a one-way sync pipeline. Music files copied to a folder on the Rockbox sync automatically to the server. Navidrome picks them up on its next scan.

All steps were completed and confirmed working.

---

## Stack Addition

| **Sync Service**   | Syncthing (containerized)         |
| ------------------ | --------------------------------- |
| **Source Device**  | Rockbox (connected via USB to macOS) |
| **Client OS**      | macOS                             |
| **Server OS**      | Ubuntu 24.04 LTS                  |
| **Runtime**        | Docker + Docker Compose           |
| **Music Service**  | Navidrome (existing container)    |

---

## Architecture

```
Rockbox (USB)
  └── conveyor_belt/ folder on Rockbox
        └── Syncthing (macOS native app) watches this folder
              └── syncs one-way to Ubuntu server
                    └── navidrome/music/ folder
                          └── Navidrome scans and streams
```

The folder on the Rockbox acts as a staging area called `conveyor_belt`. Syncthing on macOS sends files from there to the server. The server-side Syncthing container writes directly into the same folder Navidrome reads from. Navidrome picks up new files on its next scheduled scan.

---

## Prerequisites

- The Private Cloud Infrastructure project is already deployed and running
- SSH access from macOS to the Ubuntu server
- Navidrome running in Docker on the Ubuntu server
- Tailscale connected on both macOS and the Ubuntu server
- Rockbox accessible as a mounted volume on macOS (via USB)

---

## Phase 1: Add Syncthing to the Server

### Step 1: SSH into the Ubuntu Server

```bash
ssh your_username@10.0.0.x
```

### Step 2: Find Your UID and GID

```bash
id
```

Both are most likely `1000`. Write them down.

### Step 3: Stop the Running Containers

```bash
cd ~/server && /usr/local/bin/docker-compose down
```

### Step 4: Edit the Docker Compose File

```bash
nano docker-compose.yml
```

### Step 5: Replace the Navidrome Block and Add Syncthing

Find the existing `navidrome:` service block and replace it with the following. Then paste the `syncthing:` block directly after it. Keep the indentation aligned with the other services.

```yaml
  navidrome:
    image: deluan/navidrome:latest
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

  syncthing:
    image: syncthing/syncthing:latest
    container_name: syncthing
    hostname: navidrome-sync
    environment:
      - PUID=1000
      - PGID=1000
      - STGUIADDRESS=0.0.0.0:8384
    volumes:
      - ./syncthing-config:/var/syncthing/config
      - ./navidrome/music:/var/syncthing/music
    ports:
      - 8384:8384
      - 22000:22000/tcp
      - 22000:22000/udp
    restart: unless-stopped
```

The Syncthing volume for music points to the same folder Navidrome reads from: `./navidrome/music`.

Save the file: `Ctrl+X`, then `Y`, then `Enter`.

### Step 6: Start the Containers Back Up

```bash
cd ~/server && /usr/local/bin/docker-compose up -d
```

### Step 7: Fix Permissions

```bash
sudo chown -R 1000:1000 ./syncthing-config
sudo chown -R 1000:1000 ./navidrome/music
docker-compose up -d --force-recreate
```

---

## Phase 2: Configure Syncthing on the Server

### Step 8: Open the Syncthing Web UI (Server Side)

From any device on your Tailscale network, open a browser and go to:

```
http://100.x.x.x:8384
```

Replace `100.x.x.x` with your server's Tailscale IP.

### Step 9: Add Authentication and HTTPS

Go to: `Actions > Settings > GUI`

- Set a GUI Authentication username
- Set a GUI Authentication password
- Check "Use HTTPS for GUI"

---

## Phase 3: Install and Configure Syncthing on macOS

### Step 10: Download and Install Syncthing for macOS

Download the macOS app from the official Syncthing releases page:

```
https://syncthing.net/downloads/
```

Install the `.dmg` and launch the app. It runs in the menu bar and opens a local web UI at:

```
https://127.0.0.1:8384
```

---

## Phase 4: Connect the Two Devices (Handshake)

### Step 11: Get the Server Device ID

On the server Syncthing UI (`http://100.x.x.x:8384`):

```
Actions > Show ID
```

Copy the full device ID string.

### Step 12: Add the Server as a Remote Device on macOS

On the macOS Syncthing UI (`https://127.0.0.1:8384`):

- Click `+ Add Remote Device` (bottom right)
- Paste the server device ID
- Set the device name to `Navidrome-Server`
- Click Save

### Step 13: Accept the Connection on the Server

Wait up to 30 seconds. A green notification bar appears on the server Syncthing UI:

```
"Device [macOS device name] wants to connect."
```

Click `Add Device` to accept.

The two devices are now paired.

---

## Phase 5: Set Up the Rockbox Sync Folder

### Step 14: Prepare the Rockbox

Plug the Rockbox into the MacBook via USB. Open Finder. The Rockbox appears as a mounted volume.

Create a folder called `conveyor_belt` in the same directory as the existing `Music` folder on the Rockbox.

### Step 15: Add the Folder in macOS Syncthing

On the macOS Syncthing UI (`https://127.0.0.1:8384`), click `+ Add Folder`.

Fill in the tabs as follows:

**General Tab**
- Folder Label: your choice (e.g., `Rockbox Conveyor Belt`)
- Folder Path: `/Volumes/[Rockbox Name]/conveyor_belt`

**Sharing Tab**
- Check `Navidrome-Server`

**Ignore Patterns Tab**
- Check `Add Ignore Patterns`
- Enter the following pattern:
```
!/conveyor_belt/**
```

**Advanced Tab**
- Check `Watch for Changes`
- Full Rescan Interval: `60` (change back to `3600` after testing)
- Folder Type: `Send Only`
- Check `Ignore Permissions`

Click Save.

---

## Phase 6: Accept the Folder on the Server

### Step 16: Accept the Shared Folder

On the server Syncthing UI (`http://100.x.x.x:8384`), a notification appears to accept the new folder.

Click `Add` and configure the tabs:

**General Tab**
- Folder Path: `/var/syncthing/music`

**Advanced Tab**
- Check `Watch for Changes`
- Full Rescan Interval: `60` (change back to `3600` after testing)
- Folder Type: `Receive Only`
- Check `Ignore Permissions`

Click Save.

### Step 17: Enable Ignore Delete on the Server

This setting prevents files from being removed on the server when they are deleted from the Rockbox.

```
Actions > Advanced > Folders > [your folder name] > Check "Ignore Delete"
```

Click Save.

---

## Testing the Pipeline

1. Copy a music file into the `conveyor_belt` folder on the Rockbox
2. Watch the Syncthing UI on macOS: the file should appear in the sync queue
3. Watch the Syncthing UI on the server: the file should arrive in `/var/syncthing/music`
4. Trigger a manual scan in Navidrome or wait for the scheduled scan
5. The track appears in the Navidrome library

---

## Useful Commands

Check Syncthing container logs:
```bash
docker logs syncthing
```

Restart Syncthing only:
```bash
docker-compose restart syncthing
```

Confirm the music folder contents on the server:
```bash
ls ~/server/navidrome/music
```

Stop all containers:
```bash
cd ~/server && /usr/local/bin/docker-compose down
```

Start all containers:
```bash
cd ~/server && /usr/local/bin/docker-compose up -d
```

---

## Notes

- The `conveyor_belt` folder is a staging area only. Files sync from it to the server. The original `Music` folder on the Rockbox is untouched.
- Folder type `Send Only` on macOS and `Receive Only` on the server enforces one-way flow. The server never pushes changes back to the Rockbox.
- `Ignore Delete` on the server means removing a file from the Rockbox does not remove it from Navidrome.
- HTTPS is not configured on Navidrome or Nextcloud in this setup. Access is limited to the Tailscale network. A reverse proxy with SSL is a future improvement.
- The server goes offline when the Ubuntu laptop is shut down or suspended. This is expected behavior for this use case.
