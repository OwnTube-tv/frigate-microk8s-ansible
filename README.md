
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

_Further playbooks to follow: the Frigate Kubernetes deployment with Let's Encrypt TLS, deferred
until the server is on-site at a264 (ACME HTTP-01 needs the site's inbound 80/443 path)._
