### Homelab Project Report
# Private Cloud Infrastructure

| | |
|---|---|
| **Type** | Personal Homelab Project |
| **Status** | Complete |
| **Platform** | Ubuntu, Docker, Tailscale |
| **Stack** | Nextcloud, Navidrome, PostgreSQL |

---

## Problem Statement

Commercial cloud storage and streaming platforms collect and monetize user data. Free tiers often involve data sharing with third parties. Paid subscriptions add recurring costs for services that offer no data ownership or transparency.

The goal was to eliminate dependence on third-party cloud services for personal file storage and music streaming, with full ownership of data and infrastructure, at no recurring cost.

---

## Objectives

- Store and access personal files of all types from any device
- Stream a personal music library remotely without a subscription service
- Keep all data on local hardware with no third-party involvement
- Access services securely from outside the local network

---

## Solution

A self-hosted server was built on a personal laptop running Ubuntu. All services run in Docker containers orchestrated with Docker Compose, keeping the host system clean and each service isolated.

### File Storage: Nextcloud

Nextcloud provides file storage, sync, and browser-based access across all devices. PostgreSQL was selected as the database backend over the default SQLite for better long-term performance as the file library grows. Files sync to connected devices using the Nextcloud desktop and mobile clients.

### Music Streaming: Navidrome

Navidrome handles music streaming via the Subsonic API. It was selected for its low resource footprint and dedicated audio focus. The service streams FLAC and MP3 files directly from local storage. Mobile playback uses the Substreamer app on iOS connected over Tailscale.

### Remote Access: Tailscale

Tailscale creates a private WireGuard-based mesh network between the server, phone, and Mac. All remote access to Nextcloud and Navidrome routes through this encrypted tunnel. No ports are exposed to the public internet.

---

## Results

- Full file library accessible from Mac, phone, and browser with no third-party cloud involvement
- Music library of FLAC and MP3 files streaming reliably to mobile over Tailscale
- Zero recurring subscription costs for storage or streaming
- All data stored on local hardware under direct ownership and control
- Services start automatically on boot via Docker restart policies

---

## Skills Demonstrated

- Linux system administration (Ubuntu, SSH, package management)
- Docker and Docker Compose: multi-container orchestration
- PostgreSQL: database configuration and integration
- Networking: LAN configuration, Tailscale VPN, trusted domain management
- Self-hosted infrastructure: service deployment, container management, troubleshooting
