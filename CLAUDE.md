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
  Europe/Stockholm timezone + en_US.UTF-8 locale, base packages (`common` role), plus MicroK8s
  snap installation, kubectl alias and group membership (`microk8s-node` role)
- **`1-microk8s-cluster.yml`** — enables MicroK8s add-ons (dns, hostpath-storage, ingress,
  cert-manager, metrics-server, host-access, helm/helm3 — deliberately **no dashboard**;
  helm comes pre-enabled in the MicroK8s snap either way) and configures host-local
  kubectl/k9s access for admin users (`microk8s-cluster` role)
- _Planned:_ the Frigate Kubernetes deployment (namespace, ConfigMap, Deployment with `/dev/dri`
  access and memory-backed `/dev/shm`, Service, Ingress) together with the Let's Encrypt
  ClusterIssuer — **deferred until the server is physically at a264**, since ACME HTTP-01
  requires the site's inbound 80/443 path. Do not create issuers or TLS ingresses before then.

### Node Identity and the WiFi Fallback

The server keeps its WiFi interface as a fallback path, because it sits unattended at a remote
site and losing the cable must not mean losing access. That makes it a machine with two
addresses, and **Kubernetes does not consult the routing table when deciding which one identifies
the node** — `kubelet` and Calico each run their own autodetection and can pick the fallback even
though all ordinary traffic takes the cable. This happened here: before the cluster was rebuilt on
2026-08-12 the node was registered with its WiFi address as `InternalIP`.

An address that Kubernetes advertises but that can disappear eventually breaks the CNI's path to
the API server, and without the CNI no pod can start. All three autodetections are therefore
pinned to the wired address rather than left to guess:

- `--advertise-address` and `kubelet --node-ip`, set by the `microk8s-node` role from
  `microk8s_node_ip` (a host variable in `hosts.yaml`; the role defaults to the node's
  default-route address, which is correct for single-homed machines)
- Calico's `IP_AUTODETECTION_METHOD`, set by the `microk8s-cluster` role from
  `calico_ip_autodetection_method`

The sibling repo solved the same problem by removing WiFi from its nodes
(OwnTube-tv/minio-microk8s-ansible#23). That is not an option here — the fallback path is the
point — so this deployment pins instead.

### Role Structure

**`microk8s-node`** and **`microk8s-cluster`** are deliberately kept file-diffable against their
minio-microk8s-ansible counterparts — same task logic, with single-node behavior falling out of
the inventory (a one-host group makes the designated-master expression pick the only node and
empties every "other nodes" loop). The meaningful deltas are:
- `microk8s-node`: the clustering-instructions task is gated on the inventory group holding
  more than one host
- `microk8s-cluster`: role defaults stay minio-identical — the site's choices live in
  `1-microk8s-cluster.yml`, which passes a `microk8s_plugins` override (dashboard off) and
  the task toggles `create_letsencrypt_issuer: no` / `create_dashboard_ingress: no`
  taking precedence over role defaults. Flip `create_letsencrypt_issuer` to `yes` in the
  playbook once the server is on-site at a264 — never before.

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
- **Admin access:** SSH is reachable via the public IP (`87.251.30.65`) on port 622 (router
  port-forward 622→22), or via the site-to-site VPN at `192.168.5.10:22`. Ports 443/80 are
  forwarded to the MicroK8s ingress.
- **Web service posture:** the Frigate UI is deliberately availability-first (PoC/demo purpose)
  with Frigate's built-in authentication — a shared viewer account for demos plus personal
  admin accounts.

## Additional Documentation

- **`docs/hardware.md`:** Hardware specifications for the `a264a` server and the a264 site

## Related Repositories

- **[OwnTube-tv/minio-microk8s-ansible](https://github.com/OwnTube-tv/minio-microk8s-ansible):**
  the 4-node production MinIO S3 + MicroK8s cluster whose patterns this repo follows
