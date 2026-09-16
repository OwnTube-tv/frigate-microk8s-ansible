
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
- **Cameras:** three Reolink PoE units on the default LAN, all powered by the PoE switch. They are
  numbered in the Ansible variables (`frigate_camera_<n>_host`, with matching credentials in the
  vault) so that re-aiming or replacing one never means renaming a secret:
    1. **Reolink TrackMix P760** at `192.168.4.6` — "Back Yard TrackMix NE". 4K 8MP PTZ, and the
       only one of the three that presents its two lenses as independent channels (wide +
       telephoto), so it is two Frigate cameras; built-in auto-tracking. The wide channel runs
       3840×2160 main, 896×512 substream (16:9), and is switched off: the two Duo panoramas
       overlap and cover that view between them. The telephoto channel is the one recording —
       1920×1080 main, same 896×512 substream. Both lenses sit behind 4K sensors, so the 1080p is
       an encoder setting rather than a limit. Disk cost follows the 4096 kbps bitrate and not the
       resolution, so the telephoto channel produces about the same ~1.8 GB/h the wide one did
    2. **Reolink Duo 2** at `192.168.4.12` — "Driveway Duo 2 W". Fixed dual-lens, stitched to a
       single 180° panoramic stream: 4608×1728 main, 1536×576 substream (2.67:1)
    3. **Reolink Duo 2V** at `192.168.4.13` — "Back Yard Duo 2V E". Fixed dual-lens, stitched to a
       single 180° panoramic stream: 5120×1552 main, 1920×576 substream (3.33:1). Faces the
       TrackMix from the opposite side

  The two Duos are ~7.95 MP on the main stream against the TrackMix's 8.29 MP, so the measured
  2.5 GB/h per continuous 4K recorder carries over to them for retention planning. All three main
  streams are HEVC and all three substreams are H.264, probed over the site VPN on 2026-08-27 —
  so recordings are stream-copied HEVC (Chrome 108+, Edge and Safari play them; other browsers do
  not), while the only decoding the iGPU does is H.264. Every stream carries AAC audio. Note that a
  Reolink substream is not necessarily the same aspect as its main stream — camera 1 is 1.78
  against 1.75 and camera 3 is 3.30 against 3.33 — which puts detection boxes a percent or two
  off when drawn over recordings. Camera 2 is the only exact match of the three.


Camera Encoder Settings
-----------------------

These live on the cameras themselves and Ansible does not manage them — a factory reset would
silently revert them, which is why they are written down here. Verified against each camera's HTTP
API and with `ffprobe` on the streams on 2026-08-27.

| Camera                     | Stream | Resolution | fps | kbps | kbit/frame | kbit/MP | ceiling GB/h |
|----------------------------|--------|------------|-----|------|------------|---------|--------------|
| 1 TrackMix wide (disabled) | main   | 3840×2160  | 10  | 4096 | 410        | 49      | 1.84         |
| 1 TrackMix wide (disabled) | sub    | 896×512    | 20  | 1024 | 51         | 112     | not recorded |
| 1 TrackMix telephoto       | main   | 1920×1080  | 20  | 4096 | 205        | 99      | 1.84         |
| 1 TrackMix telephoto       | sub    | 896×512    | 20  | 1024 | 51         | 112     | not recorded |
| 2 Duo 2 driveway           | main   | 4608×1728  | 8   | 3072 | 384        | 48      | 1.38         |
| 2 Duo 2 driveway           | sub    | 1536×576   | 10  | 1024 | 102        | 116     | not recorded |
| 3 Duo 2V back yard         | main   | 5120×1552  | 8   | 3072 | 384        | 48      | 1.38         |
| 3 Duo 2V back yard         | sub    | 1920×576   | 10  | 1024 | 102        | 93      | not recorded |

The bitrate is a ceiling the encoder sits against, so it rather than the resolution is what
determines disk usage. Frame rate and bitrate therefore have to move together: lowering one alone
only changes how many bits each frame gets. The Duos went from 20 fps at 7168–8192 kbps to 8 fps
at 3072, which took the recording ceiling from 210 GB/day to 111 while still leaving 384 kbit per
frame — more than the back yard camera had before the change.

Measured afterwards the three recording cameras came to 109 GB/day, close to that ceiling but not
distributed the way the per-camera ceilings suggest. The quiet back yard scene sat well under its
cap at 1.22 GB/h against 1.38, which is Variable Bitrate doing its job. The busy driveway sat at
1.46 and the telephoto camera at 1.88, both *above* their nominal caps: a cap counts video only,
and AAC audio plus MP4 container overhead add roughly 150–240 kbps on top of every stream. The
ceiling is a good planning total and a poor prediction for any single camera.

Only the main streams are recorded. A substream's bitrate therefore costs no disk at all and buys
detection quality, which is why the Duos kept 1024 kbps while halving their frame rate: the same
bitrate over half the frames doubled what the detector sees per frame. Their substreams dropped to
10 fps because Frigate asks for 5 and applies `-r 5 -vf fps=5,…` as *output* arguments — every
frame arrives, gets decoded on the iGPU, and the surplus is then discarded by the filter. That
decode shares the iGPU with OpenVINO inference, so the frames not decoded are capacity the
detector gets back.

That decode scales with frames rather than pixels, which the measured per-camera CPU shows
plainly. The back yard substream pushes 11.1 Mpx/s against the telephoto channel's 9.2 and costs
8.9 % of a camera processor against 16.5 % — more pixels, half the frames, roughly half the cost.
The telephoto channel is the expensive one purely because it is the one that cannot be turned
down.

Three constraints found while setting these, all worth knowing before touching them again:

- The TrackMix's floor is 4096 kbps where the Duos reach 3072, and its substream offers 8 fps
  where the Duos offer 7 or 10.
- **Channel 1 of the TrackMix is read-only.** `GetEnc` reports it, but `SetEnc` answers
  `param error` (`rspCode: -4`) for every payload tried, while the identical payload shape against
  channel 0 returns `rspCode: 200`. The Reolink app agrees: it shows a single Clear/Fluent pair,
  which is channel 0, and `GetAbility` returns only one `abilityChn` entry. So the telephoto
  channel's 20 fps at 4096 kbps is fixed, on both its main and its substream.
- Because of that, the app's Fluent entry writes **channel 0's** substream, which feeds the
  disabled wide camera. Changing it there does not affect anything Frigate decodes.

None of that costs image quality. Measured per pixel rather than per frame, the telephoto channel
is the best-fed stream here — about 99 kbit per megapixel against 48–49 on the other main
streams. It looked starved only because kbit per frame was being compared across frames of very
different sizes.

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
