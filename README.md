
# `frigate-microk8s-ansible` – Frigate NVR on Single-Node MicroK8s

Ansible playbook to configure our Ubuntu 24.04 LTS server to run the [Frigate](https://frigate.video)
NVR with GPU-accelerated object detection. MicroK8s provides the container platform, and
cert-manager + ingress publish the web UI at https://ranchen.owntube.se/ — everything above the
OS baseline is deployed as Kubernetes manifests, so the setup doubles as a reference for running
Frigate in any Kubernetes environment.

## Getting Started

Clone the repo:

    git clone git@github.com:OwnTube-tv/frigate-microk8s-ansible.git
    cd frigate-microk8s-ansible/

Create a virtual environment (Python 3.12 or newer) and install the dependencies:

    python3 -m venv venv
    source venv/bin/activate
    pip install -r requirements.txt

Add the Ansible Vault password to a file named `.ansible_vault_password` and restrict readability:

    echo theSecretAnsibleVaultPassword > .ansible_vault_password
    chmod og-r .ansible_vault_password

Verify that the hosts are reachable:

    ansible frigate_microk8s_servers -m ping

Run through the bootstrap playbook in `--check` mode to verify that provisioning can execute:

    ansible-playbook 0-bootstrap.yml --check


## Live Deployment

### Initial Setup

The initial setup steps for a live deployment are as follows:

1. Run the `0-bootstrap.yml` playbook to prepare the server baseline and install MicroK8s:

    ```shell
    ansible-playbook 0-bootstrap.yml
    ```

2. Run the `1-microk8s-cluster.yml` playbook to enable the MicroK8s add-ons and configure
   host-local kubectl access for the admin users:

    ```shell
    ansible-playbook 1-microk8s-cluster.yml
    ```

    After successful completion, verify the single-node cluster from the server:

    ```shell
    kubectl get nodes -o wide
    ```

3. Run the `2-frigate-server.yml` playbook to deploy Frigate itself — the namespace, the rendered
   ConfigMap, the Deployment with `/dev/dri` access and memory-backed `/dev/shm`, the Service and
   the TLS ingress. Camera credentials live in the Ansible Vault and are not referenced by the
   playbook, so they have to be passed in:

    ```shell
    ansible-playbook 2-frigate-server.yml -e @secrets.yaml
    ```

    The web UI is then served at https://ranchen.owntube.se/ behind Frigate's own authentication.
    Five Reolink PoE cameras feed it; their encoder settings live on the cameras themselves rather
    than in Ansible, and are written down in [docs/hardware.md](docs/hardware.md).
