Homelab Project Guide
# Secure DNS Infrastructure: AdGuard Home, Unbound, Tailscale

**Platform:** Raspberry Pi Zero 2W | **OS:** DietPi | **Status:** Complete

A self-hosted, network-level DNS filtering stack running on a Raspberry Pi Zero 2W. AdGuard Home handles ad and tracker blocking on port 53. Unbound performs recursive DNS resolution on port 5335, querying root servers directly with no third-party upstream. Tailscale restricts admin access to authorized devices only. The entire stack survives hard reboots and comes up in the correct service order automatically.

---

## Architecture

```
Devices on network
       |
       v
AdGuard Home (port 53)     <-- blocks ads, trackers
       |
       v
Unbound (port 5335)        <-- recursive resolver, no upstream dependency
       |
       v
DNS Root Servers

Admin UI: accessible only via Tailscale (100.x.x.x:80)
```

---

## Hardware and Software

| Component | Details |
|---|---|
| Board | Raspberry Pi Zero 2W |
| OS | DietPi (Debian Bookworm) |
| DNS Filter | AdGuard Home v0.107.73 |
| Recursive Resolver | Unbound 1.22.0 |
| Remote Access | Tailscale |
| Network | Mesh Wi-Fi system |

---

## Prerequisites

Before starting, complete these steps:

- Assign a static local IP to the Pi via your router's DHCP reservation. This guide uses `192.168.1.x` as the placeholder. Replace it with your actual IP throughout.
- Have SSH access to the Pi over your local network.
- Have a Tailscale account at tailscale.com. The free tier is sufficient.

---

## Phase 1: Update DietPi

SSH into the Pi and update the system.

```bash
sudo dietpi-update
sudo apt update && sudo apt upgrade -y
sudo reboot
```

Wait 60 seconds after reboot, then SSH back in.

---

## Phase 2: Install and Configure Unbound

Unbound acts as a recursive DNS resolver. It queries root DNS servers directly, eliminating reliance on third-party resolvers like Google or Cloudflare.

### Install

```bash
sudo apt install unbound -y
```

### Download root hints

```bash
sudo curl -o /var/lib/unbound/root.hints https://www.internic.net/domain/named.cache
```

### Fix DietPi-specific trust anchor issue

DietPi's Unbound package includes a startup config that references `/var/lib/unbound/root.key`. This file does not exist on DietPi and causes Unbound to fail on start. Remove it.

```bash
sudo rm /etc/unbound/unbound.conf.d/root-auto-trust-anchor-file.conf
```

Also comment out the `auto-trust-anchor-file` line in the main config if present:

```bash
sudo sed -i 's/^\s*auto-trust-anchor-file:/#auto-trust-anchor-file:/' /etc/unbound/unbound.conf
```

### Create Unbound config

This config is tuned for the Pi Zero 2W's 512MB RAM. It disables prefetching and limits cache size to avoid memory pressure.

```bash
sudo nano /etc/unbound/unbound.conf.d/pi-hole.conf
```

Paste the following:

```
server:
    verbosity: 0
    interface: 127.0.0.1
    port: 5335
    do-ip4: yes
    do-udp: yes
    do-tcp: yes
    do-ip6: no
    prefer-ip6: no
    root-hints: "/var/lib/unbound/root.hints"
    harden-glue: yes
    harden-dnssec-stripped: yes
    use-caps-for-id: no
    edns-buffer-size: 1232
    prefetch: no
    num-threads: 1
    so-rcvbuf: 1m
    rrset-cache-size: 4m
    msg-cache-size: 2m
    private-address: 192.168.0.0/16
    private-address: 10.0.0.0/8
    private-address: 172.16.0.0/12
    private-address: 100.64.0.0/10
```

### Enable and start

```bash
sudo systemctl enable unbound
sudo systemctl start unbound
```

### Verify

Install DNS utilities if not present:

```bash
sudo apt install bind9-dnsutils -y
```

Test Unbound on port 5335:

```bash
dig @127.0.0.1 -p 5335 google.com | grep -E "status:|ANSWER"
```

Expected: `status: NOERROR` with an answer section containing an IP address.

---

## Phase 3: Install AdGuard Home

AdGuard Home listens on port 53 and forwards filtered queries to Unbound at `127.0.0.1:5335`.

### Clear port 53

DietPi does not run systemd-resolved by default. Confirm port 53 is free:

```bash
sudo ss -lnp | grep ':53'
```

If nothing appears, proceed. If something is listed, stop and disable it before continuing.

### Set a static resolv.conf

```bash
sudo rm -f /etc/resolv.conf
echo "nameserver 1.1.1.1" | sudo tee /etc/resolv.conf
```

### Download and install

The Pi Zero 2W requires the ARMv6 build.

```bash
cd /tmp
wget https://github.com/AdguardTeam/AdGuardHome/releases/download/v0.107.73/AdGuardHome_linux_armv6.tar.gz
tar -xzf AdGuardHome_linux_armv6.tar.gz
sudo mv AdGuardHome /opt/AdGuardHome
sudo /opt/AdGuardHome/AdGuardHome -s install
```

### First-time setup wizard

Open a browser and go to `http://192.168.1.x:3000`. Complete the wizard:

- Admin web interface: all interfaces, port 80
- DNS server: all interfaces, port 53
- Set a strong username and password

### Configure upstream DNS

After the wizard, go to Settings > DNS Settings.

Under Upstream DNS servers, delete all existing entries and add:

```
127.0.0.1:5335
```

Enable DNSSEC under DNS server configuration. Save.

### Verify

```bash
dig @127.0.0.1 google.com | grep -E "status:|ANSWER"
```

Check the AdGuard Home query log to confirm the query appears.

---

## Phase 4: Install Tailscale

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

A URL will appear. Open it in a browser and authenticate. The Pi will join your tailnet.

Get the Pi's Tailscale IP:

```bash
tailscale ip -4
```

This returns a `100.x.x.x` address. Save it.

Enable Tailscale on boot:

```bash
sudo systemctl enable tailscaled
```

---

## Phase 5: Restrict Admin UI to Tailscale Only

Stop AdGuard Home:

```bash
sudo systemctl stop AdGuardHome
```

Edit the config:

```bash
sudo nano /opt/AdGuardHome/AdGuardHome.yaml
```

Find the `http:` section. Change the address from `0.0.0.0:80` to your Tailscale IP:

```yaml
http:
  address: 100.x.x.x:80
```

The `dns:` section with `bind_hosts: - 0.0.0.0` stays unchanged.

Restart:

```bash
sudo systemctl start AdGuardHome
```

Verify the local IP no longer serves the admin UI:

```bash
curl -m 5 http://192.168.1.x:80 || echo "GOOD: local UI blocked"
```

The admin UI is now only reachable via Tailscale at `http://100.x.x.x:80`.

---

## Phase 6: Set Your Network DNS to the Pi

In your router or mesh system admin interface, set the LAN DNS to the Pi's static local IP:

```
Primary DNS:   192.168.1.x
Secondary DNS: 9.9.9.9  (optional fallback)
```

To set DNS manually per device:

| Device | Path |
|---|---|
| iPhone | Settings > Wi-Fi > network > Configure DNS > Manual |
| macOS | System Settings > Network > Wi-Fi > Details > DNS |
| Windows 11 | Settings > Network & Internet > Ethernet > Edit DNS > Manual |
| Ubuntu (ethernet) | Edit `/etc/systemd/resolved.conf`, set `DNS=192.168.1.x`, restart systemd-resolved |
| Android TV (ethernet) | Settings > Network > Expert Settings > set static IP, DNS field |

---

## Phase 7: Fix Service Boot Order

AdGuard Home must start after Tailscale since it binds to the Tailscale IP. Add a systemd override:

```bash
sudo systemctl edit AdGuardHome
```

Add:

```ini
[Unit]
After=network-online.target tailscaled.service
Wants=network-online.target
```

Reload:

```bash
sudo systemctl daemon-reload
```

---

## Phase 8: Monthly Root Hints Refresh

```bash
sudo crontab -e
```

Add:

```
0 3 1 * * curl -s -o /var/lib/unbound/root.hints https://www.internic.net/domain/named.cache && systemctl restart unbound
```

---

## End-to-End Verification

Run all six checks after a clean reboot:

```bash
# 1. Unbound resolves directly
dig @127.0.0.1 -p 5335 google.com | grep -E "status:|ANSWER"

# 2. AdGuard Home resolves via Unbound
dig @127.0.0.1 google.com | grep -E "status:|ANSWER"

# 3. DNS from the network (run from another device)
dig @192.168.1.x google.com | grep -E "status:|ANSWER"

# 4. Admin UI blocked on local IP
curl -m 5 http://192.168.1.x:80 || echo "GOOD: local UI blocked"

# 5. Admin UI accessible via Tailscale (from a Tailscale-connected device)
curl -m 5 http://100.x.x.x:80 | grep -i "adguard"

# 6. Tailscale connectivity
tailscale status
```

All six should return clean results.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Unbound fails to start | Trust anchor file missing | Remove `root-auto-trust-anchor-file.conf`, comment out `auto-trust-anchor-file` in main config |
| AdGuard Home crashes on boot | Corrupted stats.db after power loss | `sudo rm /opt/AdGuardHome/data/stats.db` |
| Admin UI unreachable via Tailscale | Wrong IP in `AdGuardHome.yaml` | Run `tailscale ip -4` and update `http.address` |
| Port 53 in use | Another process | `sudo ss -lnp \| grep ':53'` to identify and stop it |
| AdGuard Home starts before Tailscale | Missing systemd override | Complete Phase 7 |

---
