# Cisco Collapsed Core Homelab — Phase 2:SNMP Monitoring (LibreNMS), DNS Filtering (Pi-hole) &amp; Containerized Services (Docker)
# Patrick F — Homelab Build Guide
### Cisco Enterprise Lab · Expanded Build · 2026
**Rack:** R01 · 12U Open-Frame
**Author:** P. Furtado
 
---
 
> ⚠️ **Security Note:** All usernames, passwords, and enable secrets have been removed from this document.
 
---
 
## Overview
 
This homelab is a fully isolated Cisco enterprise-grade network running on physical hardware. It is completely separate from the home network — the ISR4331 acts as the edge device and everything downstream (switches, Pi 5, services) is self-contained. The only external touchpoint is the WAN interface pulling a DHCP address from the ISP.
 
The goal of this lab is to build hands-on experience with Cisco IOS/IOS-XE configuration, inter-VLAN routing, network monitoring, DNS management, and containerized services — skills directly applicable to enterprise networking and infrastructure roles.
 
---
 
## Hardware
 
### Current Equipment
 
| Device | Model | Role |
|---|---|---|
| Router | Cisco ISR4331/K9 | Edge Router / NAT |
| Distribution Switch | Cisco WS-C3560-12PC-S | Distribution / Inter-VLAN Routing |
| Access Switch | Cisco WS-C2960-24PC-L | Access Layer |
| Pi 5 | Raspberry Pi 5 8GB | Services Host |
 
### New Equipment (Incoming — Separate Lab)
 
| Device | Model | Role |
|---|---|---|
| Distribution Switch | Cisco WS-C3560G-24PS-S | Distribution / PoE+ |
| WLC | Cisco AIR-CT5508-K9 | Wireless LAN Controller |
| Access Point | Cisco AIR-CAP2702I-B-K9 | Wireless AP |
| Patch Panel | Jadaol 24-Port Cat6 1U | Cable Termination |
 
---
 
## IP Address & VLAN Plan
 
| VLAN | Subnet | Gateway | Purpose |
|---|---|---|---|
| VLAN 10 | 192.168.10.0/24 | 192.168.10.1 | Management |
| VLAN 20 | 192.168.20.0/24 | 192.168.20.1 | Workstations / Pi 5 |
 
| Key Addresses | IP |
|---|---|
| Pi 5 Static IP | 192.168.20.11 |
| Pi 5 DNS (Pi-hole) | 192.168.20.11 |
| ISR4331 VLAN 10 Subinterface | 192.168.10.254 |
| ISR4331 VLAN 20 Subinterface | 192.168.20.254 |
| DIST-SW VLAN 10 SVI | 192.168.10.1 |
| DIST-SW VLAN 20 SVI | 192.168.20.1 |
| ACCESS-SW VLAN 10 SVI | 192.168.10.2 |
 
---
 
## Physical Cabling
 
```
ISP Modem --> ISR4331 Gi0/0/0 (WAN — DHCP from ISP)
ISR4331 Gi0/0/1 --> DIST-SW Fa0/12 (Router-on-a-Stick trunk — VLANs 10 & 20)
DIST-SW Fa0/1 --> ACCESS-SW Fa0/1 (802.1Q Trunk — VLANs 10 & 20)
ACCESS-SW Fa0/24 --> Raspberry Pi 5 eth0 (VLAN 20 Access Port — 192.168.20.11)
```
 
---
 
## Phase 0 — Pi 5 Initial Setup
 
### Tools Installed
- **Flameshot** — screenshot utility for Ubuntu desktop
- **PuTTY** — terminal emulator for SSH and serial console connections to Cisco devices
 
### Static IP Configuration
The Pi 5 runs Ubuntu with Network Manager. The static IP was set via the Network Manager GUI:
 
- Interface: `eth0`
- IP Address: `192.168.20.11`
- Subnet: `255.255.255.0`
- Gateway: `192.168.20.1`
- DNS: `8.8.8.8` (updated to `192.168.20.11` after Pi-hole deployment)
 
> **Note:** Ubuntu's `systemd-resolved` service occupies port 53 by default and must be disabled before deploying Pi-hole, as Pi-hole takes over DNS on that port.
 
```bash
sudo systemctl stop systemd-resolved
sudo systemctl disable systemd-resolved
```
 
<!-- SCREENSHOT: Static IP — terminal output of `ip addr show eth0` showing 192.168.20.11/24 -->
 <p align="center">
  <img src="https://i.postimg.cc/bwT0jQhX/STATIC-IPI-PI5.png" alt="Raspberry Pi 5 static IP 192.168.20.11 set on eth0" width="650"/>
</p>
 
 
 
---
 
## Phase 1 — Cisco Device Configurations
 
### Console Access
All Cisco devices were configured via console cable using:
```
sudo screen /dev/ttyUSB0 9600
```
PuTTY can also be used with Serial settings: 9600 baud, 8N1, no flow control.
 
### Credentials
> Usernames, passwords, and enable secrets have been removed. Configure your own before deploying.
 
All devices use:
- Local admin account with privilege 15
- SSH v2 enabled (RSA 1024-bit)
- Console and VTY secured with local login
 
---
 
### ISR4331 — Router Configuration
 
#### What Was Unchanged
- `Gi0/0/0` — WAN interface with DHCP from ISP, NAT outside
- `Gi0/0/1.10` and `Gi0/0/1.20` — Router-on-a-Stick subinterfaces for VLAN 10 and VLAN 20
- NAT overload (PAT) on `Gi0/0/0`
 
#### What Was Removed / Fixed
- **NAT access-list** — Previously only permitted VLAN 20. Corrected to include both VLANs:
 
```
! Before (incorrect — VLAN 20 only)
access-list 10 permit 192.168.20.0 0.0.0.255
 
! After (corrected — both VLANs)
access-list 10 permit 192.168.10.0 0.0.0.255
access-list 10 permit 192.168.20.0 0.0.0.255
```
 
- **`service config`** — Removed. Was causing repeated TFTP config pull errors on boot:
 
```
no service config
```
 
#### What Was Added
- **SSH** — Enabled SSH v2 with local authentication:
 
```
IP domain-name homelab.local
username [removed] privilege 15 secret [removed]
crypto key generate rsa modulus 1024
ip ssh version 2
line vty 0 4
login local
transport input ssh
```
 
- **DHCP pool** — Added for VLAN 20 with Pi-hole as DNS:
 
```
ip dhcp excluded-address 192.168.20.1 192.168.20.10
ip dhcp pool VLAN20-WORKSTATIONS
network 192.168.20.0 255.255.255.0
default-router 192.168.20.254
dns-server 192.168.20.11
domain-name homelab.local
lease 2
```
 
- **SNMP** — Enabled for LibreNMS monitoring:
 
```
snmp-server community public RO
snmp-server community private RW
snmp-server location Rack-R01
snmp-server contact admin@homelab.local
snmp-server host 192.168.20.11 version 2c public
```
 
---
 
### DIST-SW (Catalyst 3560) — Distribution Switch
 
The 3560 handles inter-VLAN routing via Layer 3 SVIs with `ip routing` enabled. Uplinks to ISR4331 on Fa0/12 and trunks to ACCESS-SW on Fa0/1.
 
```
ip routing
 
interface Vlan10
ip address 192.168.10.1 255.255.255.0
 
interface Vlan20
ip address 192.168.20.1 255.255.255.0
 
interface FastEthernet0/12
switchport trunk encapsulation dot1q
switchport mode trunk
switchport trunk allowed vlan 10,20
description Trunk to ISR4331 Gi0/0/1
 
interface FastEthernet0/1
switchport trunk encapsulation dot1q
switchport mode trunk
switchport trunk allowed vlan 10,20
description Trunk to ACCESS-SW Fa0/1
 
ip route 0.0.0.0 0.0.0.0 192.168.20.254
 
snmp-server community public RO
snmp-server community private RW
snmp-server location Rack-R01
snmp-server contact admin@homelab.local
snmp-server host 192.168.20.11 version 2c public
```
 
---
 
### ACCESS-SW (Catalyst 2960) — Access Switch
 
The 2960 provides access ports for end devices. Fa0/24 is the Pi 5 port on VLAN 20.
 
```
interface FastEthernet0/1
switchport mode trunk
switchport trunk allowed vlan 10,20
description Trunk to DIST-SW Fa0/1
 
interface FastEthernet0/24
switchport mode access
switchport access vlan 20
spanning-tree portfast
description Pi5 eth0 — VLAN 20 — 192.168.20.11
 
snmp-server community public RO
snmp-server community private RW
snmp-server location Rack-R01
snmp-server contact admin@homelab.local
snmp-server host 192.168.20.11 version 2c public
```
 
---
 
## Phase 2 — Pi 5 Services
 
All services run as Docker containers on the Pi 5 (`192.168.20.11`) managed via Portainer.
 
### Why Docker?
Docker allows each service to run in its own isolated container without conflicting dependencies. It makes services easy to update, restart, and manage — especially useful on a Pi where resources are limited.
 
### Docker Engine
 
```bash
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
```
 
---
 
### Portainer
**Why Portainer?** Provides a web-based GUI to manage all Docker containers without the command line. Start, stop, view logs, and monitor containers from any browser on the lab network.
 
- Access: `http://192.168.20.11:9000`
 
```bash
docker volume create portainer_data
docker run -d -p 8000:8000 -p 9000:9000 --name portainer --restart=always \
-v /var/run/docker.sock:/var/run/docker.sock \
-v portainer_data:/data \
portainer/portainer-ce:latest
```
 
<!-- SCREENSHOT: Portainer container list showing all running containers -->
 <p align="center">
  <img src="https://i.postimg.cc/VkGSR4wn/CONTAINER-LIST.png" alt="Portainer container list showing all running Docker containers including LibreNMS stack and Pi-hole" width="650"/>
</p>
 
 
 
---
 
### Pi-hole
**Why Pi-hole?** Acts as a DNS sinkhole for the entire lab network. All DNS queries from lab devices route through Pi-hole, blocking ads and trackers at the network level. Also provides a full DNS query log.
 
- Access: `http://192.168.20.11/admin`
- Runs on port 53 (DNS) and port 80 (web admin)
- ISR4331 DHCP updated to hand out `192.168.20.11` as DNS for all VLAN 20 clients
 
> **Important:** Ubuntu's `systemd-resolved` occupies port 53 by default. Must be disabled before deploying Pi-hole or the container will fail with a `port already allocated` error.
 
```bash
sudo systemctl stop systemd-resolved
sudo systemctl disable systemd-resolved
```
 
```bash
docker run -d \
--name pihole \
--restart=always \
-p 53:53/tcp -p 53:53/udp \
-p 80:80 \
-e TZ="America/New_York" \
-e WEBPASSWORD="[removed]" \
-e PIHOLE_DNS_1=8.8.8.8 \
-e PIHOLE_DNS_2=8.8.4.4 \
--dns=8.8.8.8 \
-v pihole_data:/etc/pihole \
-v dnsmasq_data:/etc/dnsmasq.d \
pihole/pihole: latest
```
 
<!-- SCREENSHOT: Pi-hole dashboard showing active status, query count, and blocklist -->
 <p align="center">
  <img src="https://i.postimg.cc/bwBJbJ0T/PIHOLE-DASH.png" alt="Pi-hole dashboard showing active status, DNS query count, and blocklist" width="650"/>
</p>
 
 
 
---
 
### LibreNMS
**Why LibreNMS?** Full network monitoring platform. Connects to all three Cisco devices via SNMP and auto-discovers device details, generating real-time graphs of interface traffic, CPU, memory, and link status.
 
- Access: `http://192.168.20.11:8080`
- Deployed as a Docker Compose stack
- Port changed from default `8000` to `8080` to avoid conflict with Portainer
- All three Cisco devices added via Devices → Add Device (SNMP v2c, community: public)
 
```bash
cd ~/librenms
docker compose up -d
```
 
<!-- SCREENSHOT: LibreNMS devices list showing all 3 Cisco devices discovered with correct platform and OS -->
 <p align="center">
  <img src="https://i.postimg.cc/mZjTCkB4/LIBRENMS-DEVICE-CONFIG.png" alt="LibreNMS devices list showing all 3 Cisco devices discovered via SNMP with correct platform and OS" width="650"/>
</p>

 <p align="center">
  <img src="https://i.postimg.cc/Cxgy2VhS/LIBRENMS-DEVICES-2.png" alt="LibreNMS devices list showing all 3 Cisco devices discovered via SNMP with correct platform and OS" width="650"/>
</p>
 
---
 
## Service Access Summary
 
| Service | URL | Port |
|---|---|---|
| Portainer | http://192.168.20.11:9000 | 9000 |
| Pi-hole Admin | http://192.168.20.11/admin | 80 |
| LibreNMS | http://192.168.20.11:8080 | 8080 |
 
---
 
## Phase 3 — Planned (Later)
 
### Ansible
Automation tool for pushing configs to Cisco devices via SSH. Deferred until manual configuration experience is built up. First playbook target: SNMP config across all devices.
 
### New Equipment Lab
The incoming 3560G, CT5508 WLC, and AIR-CAP2702I AP will be documented in a separate README once hardware arrives.
 
---
 
## Checklist
 
### Phase 0 — Pi 5 Initial Setup
- [x] Install Flameshot
- [x] Install PuTTY
- [x] Set static IP `192.168.20.11` on eth0
- [x] Disable systemd-resolved for Pi-hole port 53
- [x] Verify Pi reachable from all VLANs
 
### Phase 1 — Cisco Device Configs
- [x] Configure ISR4331
- [x] Configure DIST-SW (3560)
- [x] Configure ACCESS-SW (2960)
- [ ] Configure CT5508 WLC — pending hardware
- [x] Enable SNMP on all devices
- [x] Verify VLANs, routing, and internet connectivity
 
### Phase 2 — Pi 5 Software Setup
- [x] Install Docker Engine
- [x] Install Portainer
- [x] Deploy Pi-hole
- [x] Update ISR4331 DHCP DNS to Pi-hole
- [x] Deploy LibreNMS
- [x] Add Cisco devices to LibreNMS via SNMP
 
### Phase 3 — Later
- [ ] Ansible automation
- [ ] New equipment lab (3560G, WLC, AP)
 
---
 
*Patrick F · Homelab R01 · Cisco Enterprise · 2026*
