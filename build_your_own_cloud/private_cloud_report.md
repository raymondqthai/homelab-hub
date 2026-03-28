Homelab Project Report
# Private Cloud Infrastructure: Nextcloud, Navidrome, Tailscale

## **Project Specifications**

| **Project Name**   | Homelab Private Cloud Infrastructure |
| ------------------ | ------------------------------------ |
| **Host OS**        | Ubuntu 24.04 LTS (Noble Numbat)      |
| **CPU**            | Intel Core i9-13900H                 |
| **GPU**            | NVIDIA RTX 4070 Laptop               |
| **RAM**            | 32GB DDR5                            |
| **Runtime**        | Docker + Docker Compose v5.1.1       |
| **Database**       | PostgreSQL 15                        |
| **File Service**   | Nextcloud (latest)                   |
| **Music Service**  | Navidrome (latest)                   |
| **Remote Access**  | Tailscale (WireGuard-based VPN mesh) |
| **Client Devices** | macOS, iOS                           |
| **Access Method**  | Tailscale 100.x.x.x IP addresses     |

---

## Problem Statement

This report documents the design, deployment, and outcome of a self-hosted private cloud infrastructure built on a personal laptop running Ubuntu 24.04 LTS. The project objective was to replace commercial cloud storage and music streaming services with open-source alternatives under full personal control, accessible remotely from any device without relying on third-party platforms.

The deployed stack consists of Nextcloud for file storage and sync, Navidrome for music streaming, PostgreSQL as the database backend, and Tailscale for secure remote access. All services run in Docker containers managed by Docker Compose. The project was completed successfully with all target services operational and accessible from mobile and desktop clients over the Tailscale network.

---

## Objectives

- Store and access personal files of all types from any device
- Stream a personal music library remotely without a subscription service
- Keep all data on local hardware with no third-party involvement
- Access services securely from outside the local network

---

## Solution

A self-hosted server was built on a personal laptop running Ubuntu. All services run in Docker containers orchestrated with Docker Compose, keeping the host system clean and each service isolated.

### Service selection

Nextcloud was selected as the file storage layer because it combines file sync, a web interface, and mobile and desktop clients in a single package. Competing options like Syncthing lack user account management and a web UI. Seafile is comparable but has a less active open-source community. Nextcloud is the most complete personal cloud solution without requiring multiple tools.

Navidrome was selected for music streaming over Jellyfin because it is purpose-built for audio. Jellyfin covers video, books, and music together, which carries unnecessary overhead for a music-only use case. Navidrome exposes a Subsonic-compatible API that works natively with established mobile clients on both iOS and Android.

PostgreSQL was selected as the database over SQLite and MariaDB. SQLite, Nextcloud's default, degrades under concurrent access as file counts grow. MariaDB is Nextcloud's officially documented recommendation. PostgreSQL is what Nextcloud uses in its own official All-in-One Docker image. For a long-term personal server, PostgreSQL is the more durable choice.

### Containerization

Docker with Docker Compose was chosen over a manual installation for three reasons. First, Ubuntu 24.04 ships with Python 3.12, which breaks the apt version of docker-compose (1.29.2) with a ModuleNotFoundError on the distutils module. A direct install from GitHub resolved this. Second, containers isolate each service with its own dependencies, preventing version conflicts on the host. Third, updates and resets are single-command operations that do not require modifying the host system.

### Remote access

Tailscale was chosen over a traditional VPN because it requires no port forwarding, no static IP, and no dynamic DNS configuration. It uses WireGuard under the hood and assigns each device a stable 100.x.x.x address that works consistently across different networks. Setup time was under 15 minutes across three devices.

---

## Results

### Deployment Log

| **System update**       | sudo apt update && sudo apt upgrade completed without errors                                   |
| ----------------------- | ---------------------------------------------------------------------------------------------- |
| **Docker install**      | docker.io installed via apt. User added to docker group. hello-world test passed               |
| **Docker Compose**      | apt version 1.29.2 failed due to Python 3.12 incompatibility. Replaced with v5.1.1 from GitHub |
| **Directory structure** | ~/server with subdirectories for nextcloud, navidrome, postgres created                        |
| **Compose file**        | Three-service docker-compose.yml written for postgres, nextcloud, and navidrome                |
| **Stack launch**        | docker-compose up -d: all three containers started successfully on first run                   |
| **Nextcloud setup**     | Admin account created, PostgreSQL selected as database, initial setup completed                |
| **Navidrome setup**     | Admin account created, music folder configured, 1-hour scan interval set                       |
| **Tailscale**           | Installed via install.sh, authenticated, Tailscale IP assigned and confirmed                   |
| **Trusted domain**      | Tailscale IP added to Nextcloud trusted_domains via occ command                                |
| **Client access**       | Nextcloud and Navidrome confirmed accessible from macOS and iOS over Tailscale                 |

### Service Status

| **Component**     | **Status**  | **Notes**                               |
| ----------------- | ----------- | --------------------------------------- |
| **PostgreSQL 15** | Operational | Database backend for Nextcloud          |
| **Nextcloud**     | Operational | File storage and sync active            |
| **Navidrome**     | Operational | Music streaming active                  |
| **Tailscale**     | Operational | Remote access confirmed on all devices  |


---

## Skills Demonstrated

- Linux system administration on Ubuntu 24.04 LTS
- Docker and Docker Compose for multi-service container orchestration
- Service configuration and inter-container networking (Nextcloud to PostgreSQL)
- Network troubleshooting: interface identification, local IP discovery, Ethernet vs WiFi
- Dependency conflict resolution: Python 3.12 and docker-compose incompatibility
- Remote access configuration with Tailscale and WireGuard
- Nextcloud administration via the occ command-line tool
- SSH-based remote server administration from macOS
- Infrastructure documentation for technical and professional audiences

---

## Conclusion

The deployment was completed successfully. All target services are operational and accessible remotely from macOS and iOS over Tailscale. File storage, music streaming, and remote access work fully using only open-source software on personal hardware with no recurring subscription costs.
