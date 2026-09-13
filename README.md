# Homelab

A small personal playground.


## Hardware
 
### Compute — Dell OptiPlex 7090 Micro
 
| Component | Spec |
|---|---|
| CPU | Intel i5-10500T (6c/12t, up to 3.8GHz) |
| RAM | 32GB DDR4 |
| Boot drive | 256GB SATA SSD (`local`) |
| NVMe 1 | 500GB — Proxmox pool `local-ssd1` |
| NVMe 2 | 500GB — Proxmox pool `local-ssd2` |
| External HDD | 1TB USB, ext4, mounted at `/mnt/backup` — nightly backup target |
| Hypervisor | Proxmox VE 9.2.2, single node, no clustering |
 
### Secondary router — PC Engines APU
 
| Component | Spec |
|---|---|
| Role | OPNsense, bare metal, secondary router for the LAB segment |
| Image | Serial console image (APU has no VGA output) |
| Filesystem | UFS (ZFS avoided — write amplification on SD card storage) |
 
> [VERIFY] Exact APU model / RAM / storage size not confirmed.
 
### Network hardware
 
| Device | Role | Notes |
|---|---|---|
| Unifi UCG-Ultra | Primary router, DHCP, DNS forwarding, VLAN mgmt | Kept as the stable production router |
| HP ProCurve 24-port 1GbE | Core managed switch | Trunk to UCG; VLAN 40 tagged downstream |
| TP-Link 8-port PoE managed switch | Edge switch, homelab shelf | Currently acting as an unmanaged hub — no per-port VLAN config applied |
| Dell OptiPlex 7090 | Hypervisor host | Behind TP-Link switch |
 
---
 
## Network Architecture
 
### VLANs (UCG-Ultra)
 
| VLAN | Name | Subnet | Internet | Inter-VLAN |
|---|---|---|---|---|
| 10 | DATA | `192.168.10.0/24` | Yes | Yes (to other trusted) |
| 40 | LAB | `192.168.40.0/24` | Yes | Blocked outbound to DATA/MGMT (explicit drop at UCG) |
 
### OPNsense (secondary router, inside VLAN 40)
 
- WAN: static reservation on VLAN 40, `192.168.40.132`
- LAN: serves `10.80.0.0/24` for lab VMs/devices
- Outbound NAT: Hybrid mode (routed mode preserves source IPs end-to-end where used)
- Policy: LAB (`10.80.0.0/24`) cannot reach DATA/MGMT; DATA is allowed inbound to LAB for management; LAB can reach the internet
> [VERIFY] Confirm whether Hybrid NAT is still the active mode or if this has since moved fully to routed mode.
 
### DNS/DHCP
 
- UCG-Ultra is primary DNS resolver for all clients, forwarding unmatched queries to Cloudflare (`1.1.1.1`)
- Every internal service needs a local DNS record in UniFi OS pointing at the Nginx VM IP — adding a new service always requires two steps together:
  1. Nginx `server {}` block in `/etc/nginx/sites-available/homelab`
  2. Matching local DNS record in UniFi OS
---
 
## VM Inventory
 
### Nginx — reverse proxy
- Debian 13 (Trixie), 1 vCPU, 1GB RAM, 16GB on `local-ssd1`
- IP: `10.80.0.20`
- Bare metal Nginx; single unified config at `/etc/nginx/sites-available/homelab`, symlinked into `sites-enabled`
- Sole ingress point for internal web services, terminates TLS (self-signed currently)
### Gitea — source control + database
- Ubuntu Server, IP `10.80.0.21`, named `gitea-2` (rebuilt from original install)
- Gitea running bare metal as `git` systemd service, reverse-proxied through Nginx
- PostgreSQL now runs **local to this VM** (`127.0.0.1:5432`), `pg_hba.conf` scoped to loopback — the standalone SQL VM has been decommissioned
- Nightly backup: `pg_dump` + tar of `/var/lib/gitea` to `/mnt/backup`, 7-day rolling retention, logged to `/var/log/gitea-backup.log`, run from root crontab
> [VERIFY] Backup script needs updating to reflect that Postgres is now local rather than a remote SQL VM target — confirm this has been done.
 
### SQL VM — decommissioned
- Previously a dedicated PostgreSQL VM on `local-ssd2`; superseded by local Postgres on the Gitea VM.
### Proxmox
- Web UI on `10.80.0.10:8006`, DHCP with static reservation on OPNsense, reverse-proxied through Nginx
### Portainer Prod / Test — planned, not deployed
- Prod: Ubuntu Server, 4 vCPU, 8GB RAM, 100GB `local-ssd2`
- Test: Ubuntu Server, 2 vCPU, 4GB RAM, 50GB `local-ssd1`, clone of Prod
- Planned stack: Docker + Portainer CE, InfluxDB, Grafana, Unifi Poller (all via `/opt/stacks/*/compose.yaml`)
### Unifi Poller — ready to deploy
- Debian VM, Docker via compose.yaml
- Needs a source-IP-specific allow rule from the Poller VM to the UCG on TCP 443 in the Gateway Firewall **Local** ruleset (traffic to the UCG itself hits Local, not LAN In)
- Backend (InfluxDB/Grafana) deferred until Portainer Prod exists
---
 
## Backup Strategy
 
| What | Where | How | Schedule |
|---|---|---|---|
| Gitea repos + config + DB | External 1TB HDD (`/mnt/backup`) | `/home/scripts/gitea-backup.sh` | Nightly, root crontab |
 
**Known gaps**
- No Proxmox-level VM snapshots (`vzdump`) scheduled
- Nginx VM has no backup (config treated as minimal/reproducible, not yet version-controlled)
- No backup restore test procedure defined
- Nginx config and the backup script itself are not currently in Gitea
---
 
## Open Items / In Motion
 
- Debugging UCG WireGuard VPN access into the OPNsense LAN (`10.80.0.0/24`) — suspects: VPN zone rules on UCG, static route persistence, OPNsense WAN rules scoped too narrowly
- Domain registration planned via Porkbun (`.com` or `.dev`), Cloudflare DNS, acme.sh wildcard TLS (DNS-01)
- NAS planned behind OPNsense
- Decision pending: separate SSH keys per trust boundary (Gitea, Proxmox, VM SSH) vs. single key reuse
- TP-Link switch VLAN config deferred until multi-VLAN devices are added to the lab shelf
---
 
## Notes on this document
 
This was reconciled against an earlier manifest (dated June 2026) that predates several changes — notably the OPNsense addition, decommissioning of the standalone SQL VM, and the current `10.80.0.0/24` addressing. Items marked **[VERIFY]** are carried over uncertainty; fill in or correct as needed.
