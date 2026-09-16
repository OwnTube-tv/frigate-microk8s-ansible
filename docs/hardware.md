
Hardware Details for the Frigate Server
=======================================

The Frigate server `a264a` is a single fanless mini-PC with a 4-thread Intel CPU, 32 GB memory, and
~2 TB of SSD storage. The storage is configured as two LVM Volume Groups, a 1 TB "`ubuntu-vg`" with
NVMe storage for the OS and container platform, and a 960 GB "`data-vg`" with SATA storage for
Frigate video recordings (mounted at `/srv/frigate`). The Intel HD 620 iGPU handles both video
decoding (VAAPI) and object detection inference (OpenVINO). See additional details below.


Site Overview
-------------

The server is located at site SOC-a264-s5 (Attmarby, Sweden), a rural site where the network rack
sits in the workshop building:

- **Network:** Omada-managed — ER7206 VPN router, TL-SG2210P PoE switch (8× PoE + 2 SFP),
  3× EAP653 Wi-Fi 6 access points spread across three buildings
- **Server LAN:** VPN-LAN 192.168.5.0/24, server at 192.168.5.10
- **WAN:** Starlink Business with fixed public IP `87.251.30.65` via a Starlink Mini dish
  (roof-mounted on the workshop), router in bypass mode. The IP is DHCP-delivered by Starlink —
  expect roughly 50–100 Mbit/s down and 5–20 Mbit/s up through the Mini dish
- **DNS:** `a264a.mabl.online` and `ranchen.owntube.se` are both A records for `87.251.30.65`
  (TTL 300, since the DHCP-delivered IP can change with subscription/service changes)
- **Inter-site connectivity:** IPsec VPN (IKEv2) tunnels to the a12 sites (see the
  [minio-microk8s-ansible](https://github.com/OwnTube-tv/minio-microk8s-ansible) cluster)
- **Camera:** Reolink TrackMix Series P760 — 4K 8MP PTZ PoE, dual-lens (wide + telephoto) with
  built-in auto-tracking, powered by the PoE switch


Power Supply
------------

No UPS protection at the site (yet). Mitigation: the server BIOS is configured with
"Restore on AC Power Loss = Power On" so it recovers unattended from power outages, and all
filesystem mounts use `nofail` so a degraded disk never blocks boot.


SMART Baseline (2026-08-12)
---------------------------

Baseline readings taken right after the recordings disk was provisioned, before any camera
recording started — compare against these to track SSD wear under continuous video writes:

| Attribute         | Kingston A400 (`/dev/sda`)          | WD Black SN7100 (`/dev/nvme0`) |
|-------------------|-------------------------------------|--------------------------------|
| Power-on hours    | 23                                  | 22                             |
| Power cycles      | 21                                  | 19                             |
| Wear indicator    | SSD_Life_Left: 100                  | Percentage Used: 0 %           |
| Reallocated/spare | 0 events                            | Available Spare: 100 %         |
| Data written      | n/a (no LBA attr on this Phison fw) | 40.8 GB                        |
| Temperature       | 33 °C (min/max 18/43)               | 50 °C                          |

Read with `sudo smartctl -a /dev/sda` and `sudo smartctl -a /dev/nvme0` (smartmontools is part
of the common-role baseline).


Server Details for `a264a`
--------------------------

Server setup:

    Site: SOC-a264-s5
    Hostname: a264a.mabl.online
    Model: MSI Cubi 3 Silent
    OS: Ubuntu 24.04
    CPU: Intel i5-7200U, 7th Gen (4 threads)
    GPU: Intel HD Graphics 620 (VAAPI + OpenVINO)
    Memory: 32 GB (DDR4 SDRAM)
    Hard drives:
      - 1 TB SSD (WD Black SN7100 M.2 NVMe, Gen4) — OS, `ubuntu-vg`
      - 960 GB SSD (Kingston A400 SATA) — recordings, `data-vg`
    Network (DHCP reservations in Omada):
      - 1 GbE LAN (enp3s0, MAC 30:9c:23:b0:89:55), address 192.168.5.10/24 (VPN-LAN)
      - 1 GbE LAN (enp0s31f6, MAC 30:9c:23:b0:89:54), spare port,
        reserved 192.168.5.11/24 (VPN-LAN, marked "DO NOT USE")
      - 802.11ac Wi-Fi (wlp2s0, MAC 94:b8:6d:7a:48:06), fallback path,
        reserved 192.168.4.9/24 (default LAN)
