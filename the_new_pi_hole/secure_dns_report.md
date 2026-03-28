Homelab Project Report
# Secure DNS Infrastructure: AdGuard Home, Unbound, Tailscale

## **Executive Summary**

This project delivers a self-hosted, network-level DNS filtering and privacy stack on a Raspberry Pi Zero 2W running DietPi. The system routes all DNS queries on the local network through AdGuard Home for ad and tracker blocking, then through Unbound for recursive resolution that contacts DNS root servers directly. Administrative access to the management interface is restricted exclusively to devices connected via Tailscale VPN, eliminating exposure of the admin panel to the local network.

The project required diagnosis and resolution of multiple DietPi-specific package conflicts, ARM architecture compatibility issues, and systemd service ordering dependencies. All issues were resolved through systematic root cause analysis. The final system survives hard power loss and restores full functionality automatically on reboot.

## **Project Objectives**

- Deploy network-level DNS filtering to block ads and trackers for all devices on the network
- Eliminate reliance on third-party DNS resolvers such as Google or Cloudflare
- Restrict administrative access to authorized devices only via encrypted tunnel
- Ensure automatic service recovery after unplanned power loss
- Document the full build process for reproducibility

## **System Architecture**

The stack uses a layered approach with clear separation of responsibilities across three services.

|**Layer**|**Service**|**Port**|**Responsibility**|
|---|---|---|---|
|DNS filter|AdGuard Home v0.107.73|53 (all interfaces)|Ad/tracker blocking, query logging, DNSSEC|
|Recursive resolver|Unbound 1.22.0|5335 (localhost only)|Root-recursive DNS, no upstream dependency|
|Admin access|Tailscale|100.x.x.x:80|Encrypted tunnel, admin UI restriction|

DNS queries from network devices arrive at AdGuard Home on port 53. Filtered queries forward to Unbound at 127.0.0.1:5335. Unbound resolves recursively from DNS root servers. The AdGuard Home admin web interface binds exclusively to the Tailscale network interface, making it unreachable from the local network without an active Tailscale connection.

## **Hardware and Software**

|**Component**|**Specification**|
|---|---|
|Board|Raspberry Pi Zero 2W (ARM Cortex-A53, 512MB RAM)|
|Operating System|DietPi (Debian Bookworm)|
|DNS Filter|AdGuard Home v0.107.73 (ARMv6 build)|
|Recursive Resolver|Unbound 1.22.0|
|Remote Access|Tailscale (free tier)|
|Network|ISP router + mesh Wi-Fi system|
|Build binary|AdGuardHome_linux_armv6.tar.gz|

## **Technical Design Decisions**

### **AdGuard Home over Pi-hole**

AdGuard Home was selected over Pi-hole because it ships as a single self-contained binary with no external runtime dependencies, provides a built-in web UI without requiring a separate web server, and supports upstream DNS configuration including custom ports natively. Pi-hole requires PHP, lighttpd, and additional package dependencies that increase memory footprint on a 512MB device.

### **Unbound on port 5335, not port 53**

Running Unbound on port 5335 instead of the standard port 53 avoids a port conflict with AdGuard Home. AdGuard Home owns port 53 as the network-facing resolver. Unbound operates as a local-only backend. This separation means each service has a single, well-defined role and neither depends on the other's port behavior.

### **Tailscale for admin access instead of firewall rules**

Firewall rules based on local IP ranges require ongoing maintenance as network topology changes and provide no protection against devices already on the local network. Tailscale provides cryptographic device identity, meaning only explicitly authorized devices with valid keys can reach the admin interface regardless of their network location. This approach scales to multiple sites without reconfiguring firewall rules.

### **Memory tuning for Pi Zero 2W**

The default Unbound configuration targets servers with several gigabytes of RAM. On the Pi Zero 2W's 512MB, default settings cause memory pressure under sustained query load. The deployed configuration disables prefetching, limits the RR set cache to 4MB, limits the message cache to 2MB, and sets a single thread. These values keep Unbound's footprint under 30MB while maintaining adequate resolution performance for a home network.

### **systemd service ordering**

AdGuard Home binds its admin interface to the Tailscale IP address (100.x.x.x). If AdGuard Home starts before Tailscale assigns that IP, the bind fails and AdGuard Home enters a crash loop. The systemd override adds an explicit After=tailscaled.service dependency, ensuring the correct start order on every boot without manual intervention.

## **Issues Encountered and Resolved**

|**Issue**|**Root Cause**|**Resolution**|
|---|---|---|
|Unbound failed to start on DietPi|DietPi's Unbound package ships root-auto-trust-anchor-file.conf, which references /var/lib/unbound/root.key. This file does not exist on DietPi. The pre-start hook exits with status 1.|Removed root-auto-trust-anchor-file.conf and commented out auto-trust-anchor-file in the main Unbound config.|
|AdGuard Home ARMv6 binary returned 404|The generic arm filename was retired in recent releases. The Pi Zero 2W requires the armv6 build.|Downloaded AdGuardHome_linux_armv6.tar.gz directly from the versioned release URL.|
|AdGuard Home crashed on reboot after power loss|Hard power removal during a write operation corrupted the stats.db BoltDB file. Unclean shutdown left an invalid freelist page.|Deleted the corrupted stats.db and allowed AdGuard Home to recreate it on next start.|
|AdGuard Home crash-looped on boot|AdGuard Home started before Tailscale had assigned the 100.x.x.x IP, causing the bind to the Tailscale interface to fail.|Added After=tailscaled.service to the AdGuardHome systemd override.|

## **Verification Results**

All six end-to-end checks passed after a clean reboot with no manual intervention.

|**Check**|**Method**|**Result**|
|---|---|---|
|Unbound resolves on port 5335|dig @127.0.0.1 -p 5335 google.com|NOERROR, answer returned|
|AdGuard Home resolves via Unbound|dig @127.0.0.1 google.com|NOERROR, answer returned|
|Network DNS resolves via Pi|dig @192.168.1.x google.com (from client device)|NOERROR, answer returned|
|Admin UI blocked on local IP|curl -m 5 http://192.168.1.x:80|Connection refused|
|Admin UI accessible via Tailscale|curl http://100.x.x.x:80 from Tailscale device|AdGuard Home UI returned|
|Boot persistence|Hard reboot, systemctl status all three services|All active (running), no manual intervention|

## **Skills Demonstrated**

### **Relevance to Data Analyst Roles**

- Structured troubleshooting: each failure was isolated to a root cause before applying a fix, mirroring data quality debugging workflows
- System logging and log analysis: used journalctl, systemctl status, and grep filters to extract signal from noisy service output
- Scripted configuration: used sed, tee, and cron for repeatable, auditable system changes
- Documentation: produced a fully reproducible written record of the build process

### **Relevance to Robotics and Test Engineering Roles**

- Embedded Linux system configuration on constrained hardware (512MB RAM, ARM Cortex-A53)
- Service dependency management and systemd unit file customization
- End-to-end system verification with defined pass/fail criteria for each check
- Fault isolation and recovery: identified corrupted database state, stopped the failing service, removed the corrupt artifact, and restored operation
- Network stack understanding: DNS resolution chain, port binding, interface-level access control
- Hardware-software integration: ARMv6 binary selection, memory-tuned configuration for specific hardware constraints

## **Ongoing Maintenance**

- Root hints file refreshes automatically on the first of each month via cron
- AdGuard Home and Unbound update via standard apt upgrade
- Tailscale updates automatically
- Admin access requires an active Tailscale connection on the managing device

## **Conclusion**

The project delivers a functional, hardened DNS infrastructure on minimal hardware. All DNS queries from the local network pass through AdGuard Home for filtering and Unbound for private recursive resolution. The admin interface is inaccessible from the local network and reachable only through an authenticated Tailscale tunnel. The system is self-healing after power loss and requires no manual startup procedures.

The build process required working through five distinct failure modes, each resolved through systematic diagnosis. The documented troubleshooting steps make the configuration reproducible on other DietPi installations where the same package conflicts will occur.
