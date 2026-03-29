Homelab Project Report
# Remote Upload Music File Pipeline: Syncthing, Navidrome, Rockbox

---

## Project Specifications

| **Project Name**    | Self-Hosted Music Sync: Syncthing + Navidrome + Rockbox |
| ------------------- | ---------------------------------------------------- |
| **Host OS**         | Ubuntu 24.04 LTS (Noble Numbat)                      |
| **CPU**             | Intel Core i9-13900H                                 |
| **GPU**             | NVIDIA RTX 4070 Laptop                               |
| **RAM**             | 32GB DDR5                                            |
| **Runtime**         | Docker + Docker Compose v5.1.1                       |
| **Sync Service**    | Syncthing (containerized, latest)                    |
| **Music Service**   | Navidrome (existing, latest)                         |
| **Source Device**   | Rockbox (USB-connected to macOS)                        |
| **Client OS**       | macOS                                                |
| **Server OS**       | Ubuntu 24.04 LTS                                     |
| **Remote Access**   | Tailscale (existing)                                 |
| **Access Method**   | Tailscale 100.x.x.x IP addresses                    |

---

## Problem Statement

This report documents the design, deployment, and outcome of a sync pipeline added to an existing self-hosted music streaming server. The prior project deployed Navidrome on an Ubuntu laptop running Docker, with Tailscale for remote access. That setup required manually copying music files to the server to add new tracks to the library.

The objective of this build was to automate that transfer step. The target workflow: add files to the Rockbox, and have them appear in Navidrome without manually moving files to the server.

The solution uses Syncthing as a containerized sync service that shares a folder with Navidrome directly on the server filesystem. On the macOS side, Syncthing runs as a native app and watches a staging folder on the Rockbox. Files added to that folder sync one-way to the server. Navidrome picks them up on its next scheduled scan.

---

## Objectives

- Eliminate the manual file transfer step for adding music to Navidrome
- Keep all data on local hardware with no third-party sync service
- Enforce one-way sync: source device sends, server receives only
- Preserve existing files on the server if source files are removed from the Rockbox

---

## Solution

### Syncthing as the Sync Layer

Syncthing was selected over alternatives for several reasons. It runs as an open-source, peer-to-peer sync daemon with no relay server required when both devices are on the same Tailscale network. It supports folder-level sync direction controls (Send Only, Receive Only) and an Ignore Delete setting that prevents deletions on the source from propagating to the destination. Both of those controls are necessary for this use case.

Other options considered: rsync over SSH works for manual or cron-scheduled transfers but requires scripting and does not watch for changes in real time. A shared network drive adds complexity and breaks when the Rockbox is not mounted. Syncthing handles all of this at the application layer with a web UI for monitoring and configuration.

Syncthing was added as a new container in the existing Docker Compose file rather than installed on the host. This keeps the host system clean and the service lifecycle tied to the same Compose stack as Navidrome and Nextcloud.

### Folder Architecture

The Syncthing container is given two volume mounts: one for its configuration directory and one for the music folder. The music volume mount points to the same host directory that Navidrome reads from: `./navidrome/music`. This means Syncthing writes files directly into the folder Navidrome scans. No additional copy or move step is needed after sync completes.

On the macOS side, the watched folder is a directory called `conveyor_belt` created on the Rockbox volume. This keeps sync activity isolated from the Rockbox's existing Music folder. Files are copied into `conveyor_belt` and synced to the server. The original Music folder is not touched.

### Permissions

The Syncthing container runs with `PUID=1000` and `PGID=1000` to match the default Ubuntu user. Ownership of both the config directory and the music folder was set explicitly with `chown` before forcing a container recreate. Without this step, Syncthing fails to write to the mounted volume due to permission errors.

### Sync Direction and Delete Behavior

The macOS Syncthing folder is configured as Send Only. The server folder is configured as Receive Only. This enforces a strict one-way flow. Ignore Delete is enabled on the server folder so that removing a file from the Rockbox staging area does not remove it from the Navidrome library.

---

## Deployment Log

| **Task**                        | **Outcome**                                                                 |
| ------------------------------- | --------------------------------------------------------------------------- |
| Docker Compose edited           | Navidrome block updated, Syncthing block added with shared music volume     |
| Permissions corrected           | chown applied to syncthing-config and navidrome/music directories           |
| Containers recreated            | docker-compose up -d --force-recreate completed without errors              |
| Server Syncthing UI accessible  | Confirmed at http://100.x.x.x:8384 over Tailscale                          |
| Server auth configured          | GUI username, password, and HTTPS enabled                                   |
| macOS Syncthing installed       | App installed via .dmg, local UI confirmed at https://127.0.0.1:8384        |
| Device pairing completed        | Server device ID added on macOS, connection accepted on server              |
| Rockbox folder created             | conveyor_belt directory created on Rockbox volume in Finder                    |
| macOS folder configured         | Send Only, Watch for Changes, Ignore Permissions, Ignore Patterns set       |
| Server folder accepted          | Receive Only, Watch for Changes, Ignore Permissions, Ignore Delete set      |
| End-to-end sync confirmed       | Test file copied to Rockbox, synced to server, appeared in Navidrome library   |

---

## Service Status

| **Component**        | **Status**   | **Notes**                                           |
| -------------------- | ------------ | --------------------------------------------------- |
| Syncthing (server)   | Operational  | Container running, UI accessible over Tailscale     |
| Syncthing (macOS)    | Operational  | Native app running, paired with server              |
| Navidrome            | Operational  | Picks up synced files on scheduled scan             |
| Tailscale            | Operational  | Existing remote access, used for Syncthing UI       |

---

## Skills Demonstrated

- Docker Compose service extension: adding a new container to an existing multi-service stack
- Volume architecture design: shared filesystem between two containers (Syncthing and Navidrome)
- Linux file permissions: UID/GID alignment between container user and host filesystem
- Syncthing configuration: device pairing, folder sync direction, delete propagation controls
- Cross-platform setup: coordinating server-side (Ubuntu/Docker) and client-side (macOS/native app) configuration
- Network access: Tailscale used for remote Syncthing UI access without additional port exposure
- End-to-end pipeline validation: file ingested at source, transferred, confirmed in destination service

---

## Conclusion

The sync pipeline was deployed successfully. Files added to the `conveyor_belt` folder on the Rockbox sync automatically to the Navidrome music directory on the Ubuntu server. Navidrome picks them up on its next scan without any manual intervention. The one-way sync configuration and Ignore Delete setting keep the server library stable regardless of changes on the source device.

This project extends the prior Private Cloud Infrastructure build with a working file ingestion pipeline, replacing a manual transfer step with an automated, monitored sync process.
