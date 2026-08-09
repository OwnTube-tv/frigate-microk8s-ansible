# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This Ansible project configures a single-node MicroK8s server running the
[Frigate](https://frigate.video) NVR for a horse-monitoring PoC at site `SOC-a264-s5` (Attmarby,
Sweden). The server, `a264a.mabl.online`, ingests video from a Reolink PoE camera, performs object
detection (horse/person/car) on the Intel iGPU via OpenVINO, and records to local storage. The web
UI is published at https://ranchen.owntube.se/ through the MicroK8s ingress with cert-manager
(Let's Encrypt) TLS, over the site's Starlink uplink.

**Design philosophy:** everything above the OS baseline is deployed as Kubernetes manifests
(cert-manager, ingress, Frigate) rather than docker-compose or bare systemd services. This mirrors
the deployment patterns of the sibling repo
[minio-microk8s-ansible](https://github.com/OwnTube-tv/minio-microk8s-ansible), serves as
onboarding material for Kubernetes-based deployment thinking, and produces demo manifests that
customers and open source users can evaluate for deployment in their own Kubernetes environments.

## Development Environment Setup

1. Create Python virtual environment (Python 3.12 or newer) and install dependencies:
   ```bash
   python3 -m venv venv
   source venv/bin/activate
   pip install -r requirements.txt
   ```

2. Create Ansible Vault password file (never commit this file):
   ```bash
   echo theSecretAnsibleVaultPassword > .ansible_vault_password
   chmod og-r .ansible_vault_password
   ```

3. Verify host connectivity:
   ```bash
   ansible frigate_microk8s_servers -m ping
   ```

## Common Commands

Test a playbook in check mode (dry-run):
```bash
ansible-playbook <playbook>.yml --check
```

Run the bootstrap playbook (initial server setup):
```bash
ansible-playbook 0-bootstrap.yml
```

Check syntax:
```bash
ansible-playbook <playbook>.yml --syntax-check
```

Run with specific tags / on specific hosts:
```bash
ansible-playbook <playbook>.yml --tags <tag_name> --limit node-1
```

Edit the encrypted secrets:
```bash
ansible-vault edit secrets.yaml
```

## Architecture

### Server

|                 |                                                                                     |
|-----------------|-------------------------------------------------------------------------------------|
| Host            | `a264a.mabl.online` (inventory name `node-1`)                                       |
| Hardware        | Fanless Intel i5-7200U, 32 GB DDR4, Intel HD 620 iGPU                               |
| OS disk         | 1 TB NVMe — Ubuntu 24.04 LTS, LVM volume group `ubuntu-vg`                          |
| Recordings disk | 960 GB SATA SSD — volume group `data-vg`, mounted at `/srv/frigate`                 |
| Site network    | 192.168.5.10 on the a264 VPN-LAN (192.168.5.0/24, VLAN 5)                           |
| WAN             | Starlink Business with fixed public IP `87.251.30.65` (bypass mode, DHCP-delivered) |
| DNS             | `a264a.mabl.online` and `ranchen.owntube.se` → A `87.251.30.65` (TTL 300)           |

### Ingress Path

```
internet → ranchen.owntube.se (87.251.30.65) → router port-forward 443/80
         → MicroK8s nginx ingress (TLS via cert-manager/Let's Encrypt)
         → Frigate Service (port 8971, Frigate built-in authentication)
```

### Deployment Phases

- **`0-bootstrap.yml`** — server baseline: sysadmin/service accounts, SSH hardening,
  Europe/Stockholm timezone, base packages (via the `common` role)
- _Planned:_ MicroK8s installation and add-on configuration (`dns`, `hostpath-storage`,
  `ingress`, `cert-manager`), then the Frigate Kubernetes deployment (namespace, ConfigMap,
  Deployment with `/dev/dri` access and memory-backed `/dev/shm`, Service, Ingress)

### Role Structure

**`common`** — same baseline as in minio-microk8s-ansible:
- Configures sysadmin accounts and Ansible service account with SSH key authentication
- Hardens SSH (no root login, MaxAuthTries 2, AllowUsers whitelist, no password auth for sudoers)
- Sets timezone to Europe/Stockholm
- Installs essential packages (incl. smartmontools for SSD health monitoring)
- Removes firewalld (conflicts with Calico networking)

### Configuration Files

- **`ansible.cfg`**: roles path, inventory file (`hosts.yaml`), vault password file
- **`hosts.yaml`**: the `frigate_microk8s_servers` inventory group (single node)
- **`secrets.yaml`**: Ansible Vault encrypted — camera RTSP/ONVIF credentials and Frigate
  web UI account passwords

## Important Implementation Notes

### Ansible Best Practices

- Always use FQCN (Fully Qualified Collection Name) for modules: `ansible.builtin.user`,
  `community.general.timezone`, `ansible.posix.authorized_key`
- Tag all role tasks for selective execution (e.g. `common`)
- Kubernetes resources are applied via `kubernetes.core` modules with manifest templates —
  keep manifests readable and self-contained so they double as customer-facing demos

### Security Considerations

- **Secrets management:** all sensitive data in Ansible Vault (`secrets.yaml`). Never commit
  unencrypted secrets or the `.ansible_vault_password` file.
- **Admin access:** SSH via Tailscale or the site-to-site VPN — no SSH port forwarding on the WAN.
  Only 443/80 are forwarded (to the ingress).
- **Web service posture:** the Frigate UI is deliberately availability-first (PoC/demo purpose)
  with Frigate's built-in authentication — a shared viewer account for demos plus personal
  admin accounts.

## Additional Documentation

- **`docs/hardware.md`:** Hardware specifications for the `a264a` server and the a264 site

## Related Repositories

- **[OwnTube-tv/minio-microk8s-ansible](https://github.com/OwnTube-tv/minio-microk8s-ansible):**
  the 4-node production MinIO S3 + MicroK8s cluster whose patterns this repo follows
